# dozzle — просмотр логов

Веб-просмотр логов docker-контейнеров прямо в портале, с разграничением
доступа по сервисам.

- **Образ:** `amir20/dozzle:latest`
- **Контейнер:** `gateway-dozzle`, слушает `:8080`
- **Сеть:** только `service_network` — наружу выходит через nginx
- **Docker API:** через [docker-socket-proxy](docker-socket-proxy.md),
  сам сокет не смонтирован
- **Healthcheck:** `/dozzle healthcheck`
- **База пути:** `DOZZLE_BASE=/logs`

## Как устроен доступ

Dozzle запущен в режиме `DOZZLE_AUTH_PROVIDER=forward-proxy` — собственной
аутентификации у него нет, он доверяет заголовкам, которые ставит nginx:

```
браузер → nginx /logs
             │
             ├─ auth_request → auth-service /verify-logs-access?service=<key>
             │       200 + Remote-User / Remote-Email / Remote-Name / Remote-Filter
             │       401 → /login,  403 → /menu (или на страницу сервиса)
             │
             └─ proxy_pass → gateway-dozzle:8080  (+ Remote-* заголовками)
```

`Remote-Filter` — фильтр контейнеров, который Dozzle применяет к сессии.
Именно он превращает общий просмотрщик в «логи только моего сервиса».

## Два входа

| Путь | Кто видит | Фильтр |
|---|---|---|
| `/logs` | админ — все контейнеры; остальные — объединение разрешённых сервисов | из прав пользователя |
| `/services/<key>/logs` | владелец сервиса | `getContainerFilterForService(<key>)` — **применяется и к админу** |

Второй вариант встраивается в карточку сервиса iframe'ом. Чтобы iframe
работал, блоки Dozzle переопределяют `X-Frame-Options` на `SAMEORIGIN`
(общий заголовок server-блока — `DENY`).

Когда iframe делает свои API-запросы к `/logs/api/…`, путь сервиса в URL
уже теряется. Его восстанавливает `map $http_referer $logs_referer_service`
в `nginx.conf` — из Referer вытаскивается `/services/([^/]+)/logs`, и фильтр
остаётся на месте.

## Права на логи

Проверяются в `verifyLogsAccessHandler` (auth-service):

| Право / роль | Что даёт |
|---|---|
| системный админ (`auth.*`, `auth/GOD`, `auth/admin`) | все логи без фильтра на `/logs` |
| роль `service-manager` в сервисе | логи своего сервиса |
| `auth.logs.view` | логи **всех внешних** сервисов (не auth) |
| `auth.<serviceKey>.logs.view` | логи одного сервиса |
| `auth.logs.system.view` | логи самого auth-service и mongo |

Пользователь без единого из этих прав получает `403` → редирект на `/menu`.

Список «внешних» сервисов для сборного фильтра захардкожен в
`getAllowedLogsServices`: `referal`, `client-service`, `notification`,
`monitoring`. Новый сервис туда надо добавлять руками.

## Соответствие «сервис → контейнеры»

`getContainerFilterForService` (auth-service):

| serviceKey | Фильтр Dozzle |
|---|---|
| `referal` | `name=referal*` |
| `client-service` | `name=client*` |
| `notification` | `name=notification*` |
| `monitoring` | `name=monitoring*` |
| `auth` | `name=gateway-auth*,name=gateway-mongo*` |
| `hr_bot` | `name=hr-bot*` (ключ с подчёркиванием, контейнеры с дефисом) |
| прочее | `name=<serviceKey>*` |

Если контейнеры сервиса названы не по ключу, фильтр по умолчанию не совпадёт
и пользователь увидит пустой список — нужна явная запись в этой карте.

## Настройки контейнера

| Переменная | Значение | Зачем |
|---|---|---|
| `DOCKER_HOST` | `tcp://docker-socket-proxy:2375` | доступ к API без сокета |
| `DOZZLE_AUTH_PROVIDER` | `forward-proxy` | доверяет `Remote-*` от nginx |
| `DOZZLE_BASE` | `/logs` | префикс путей |
| `DOZZLE_LEVEL` | `info` | уровень собственных логов |
| `DOZZLE_NO_ANALYTICS` | `true` | не ходить наружу |

WebSocket-проксирование настроено в nginx: `proxy_http_version 1.1`,
`Upgrade`/`Connection`, таймауты 300 с, `proxy_buffering off` — иначе логи
не идут в реальном времени.

## Грабли

- **Dozzle доверяет заголовкам.** Если запрос дойдёт до `:8080` минуя nginx,
  аутентификации не будет вовсе. Поэтому контейнер живёт только в
  `service_network` и портов не публикует.
- **`INFO=1`, `LOG=1`, `EVENTS=1` у прокси обязательны** — Dozzle v10+
  дёргает `GET /info` на старте, стримит логи и слушает события.
- **Фильтр не защищает от подбора имён контейнеров** внутри разрешённого
  паттерна — это разграничение по сервисам, а не по контейнерам.
