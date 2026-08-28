# Архитектура шлюза

## Путь запроса

```
браузер
  │ https://analytics.gh.uz/referal/deals
  ▼
nginx  (public_network — единственный контейнер, торчащий наружу)
  │
  ├─ 1. auth_request → auth-service GET /verify   (по cookie token)
  │       401 → редирект на /login
  │       403 → редирект на /access-denied
  │       200 → отдаёт заголовки X-User-*
  │
  └─ 2. proxy_pass http://referal:80/deals  (service_network)
          + X-User-Name, X-User-ID, X-User-Service-Roles,
            X-User-Service-Permissions, X-Service-Key, …
```

Бизнес-сервис не проверяет пароли и не видит JWT. Он получает уже проверенную
личность в заголовках и решает только «что этому набору прав можно».

## Сети

| Сеть | Тип | Кто в ней | Зачем |
|---|---|---|---|
| `public_network` | bridge | **только** nginx | приём внешнего трафика |
| `service_network` | bridge, `internal: true` | nginx, auth-service, notification-bot, notification-service, monitoring-service, docker-socket-proxy, dozzle + все бизнес-сервисы | межсервисное общение, наружу хода нет |
| `data_network` | bridge, `internal: true` | auth-service, mongo | доступ к MongoDB — больше ни у кого |
| `egress_network` | bridge, **external**, `enable_icc=false` | notification-bot, сервисы, которым нужен интернет | выход к Telegram API, внешним MySQL и т.п.; контейнеры внутри сети друг друга **не видят** |
| `quiz_network` | bridge, `internal: true` | nginx, quiz-* | quiz изолирован от auth-service и остальных |
| `notification_network` | bridge, `internal: true` | notification-service, notification-postgres | БД уведомлений полностью закрыта |
| `smtp_network` | bridge, `internal: false` | notification-service | исходящий SMTP |

`egress_network` создаётся не compose'ом, а скриптом `start_all.sh` — ему нужна
опция `com.docker.network.bridge.enable_icc=false`, которой в compose нет.
Поэтому в `docker-compose.yaml` она объявлена `external: true` и **compose
упадёт, если сеть не создана заранее**.

## Контракт `auth_request`

`GET /verify` вызывается nginx'ом изнутри (`location = /verify` помечен
`internal`). Сервис определяется по первому непустому сегменту
`X-Original-URI`: `/referal/deals` → сервис `referal`.

Ответ:

| Код | Когда | Что делает nginx |
|---|---|---|
| `401` | нет cookie `token`, токен невалиден или в блэклисте | `302 /login?redirect=…` |
| `403` | у пользователя нет **ни одной** роли и **ни одного** права в этом сервисе | `302 /access-denied?service=…` |
| `200` | доступ есть | проксирует запрос дальше с заголовками ниже |

### Заголовки, которые шлюз кладёт в запрос к сервису

| Заголовок | Содержимое |
|---|---|
| `X-User-Name` | логин |
| `X-User-ID` | ObjectID пользователя |
| `X-User-Full-Name` | ФИО, **base64** |
| `X-User-Full-Name-Encoding` | всегда `base64` — признак кодировки предыдущего |
| `X-User-Email`, `X-User-Phone` | контакты (если заполнены) |
| `X-User-Avatar` | путь вида `/avatar/<userID>` |
| `X-User-Service-Roles` | роли пользователя **в этом сервисе** |
| `X-User-Service-Permissions` | права пользователя **в этом сервисе** |
| `X-User-Roles`, `X-User-Permissions` | легаси-поля |
| `X-User-Passport-Number`, `-Giver`, `-Date`, `-Address`, `X-User-PINFL`, банковские | данные из документов профиля |
| `X-Service-Key`, `X-Service-Prefix` | какой сервис и под каким префиксом отвечает |
| `X-Forwarded-Prefix` | `/referal` — сервису нужен для `url_for()` |

`X-User-Admin` **отменён**: шаблон конфига явно затирает его
(`proxy_set_header X-User-Admin "";`), иначе клиент мог бы прислать его сам.
Администратор получает доступ обычным путём — ролью `admin` с правом
`<service>.*`, которую заводит `EnsureServiceAdminRoles()` при старте
auth-service.

## Service discovery

Маршруты бизнес-сервисов **не пишутся руками**. Схема:

```
сервис стартует
   │ POST /api/registry/register  (X-API-Key: INTERNAL_API_KEY)
   │   {service_key, container_name, internal_url, health_check_path}
   ▼
auth-service
   ├─ пишет service_instances в MongoDB
   ├─ генерирует /etc/nginx/conf.d/dynamic/service-<key>.conf
   ├─ перегенерирует dynamic/services.conf (master include)
   └─ POST docker-socket-proxy /containers/gateway-nginx-1/kill?signal=HUP
          └─ nginx перечитывает конфиг

далее каждые N секунд: POST /api/registry/heartbeat
```

Каталог `dynamic/` — общий docker-том `nginx_dynamic_config`, смонтированный
и в auth-service (на запись), и в nginx (на чтение).

Особенности, которые ломают ожидания:

- **Дерегистрация не удаляет маршрут.** `DELETE /api/registry/unregister/:key`
  снимает инстанс с учёта, но конфиг nginx не трогает — иначе рестарт
  контейнера превращал бы 502 в 404 для всех пользователей.
- **Маршрут строится по ВСЕМ незаделённым сервисам**, а не только по живым:
  `GetActiveServicesForNginx()` берёт `services` со статусом ≠ `deleted` и
  подставляет параметры любого известного инстанса.
- **`unmanaged_routing: true`** у сервиса отключает автогенерацию — так живёт
  quiz, чей маршрут описан вручную в `nginx/conf/quiz.inc`.
- **Health-монитор** в auth-service раз в 30 с помечает инстансы без heartbeat
  2 минуты как `unhealthy`, а через 10 минут удаляет их из `service_instances`.

## Хранилища

| Что | Где | Кто пишет |
|---|---|---|
| пользователи, роли, права, сервисы, инстансы, токены Telegram, сессии, логи импорта | MongoDB `authdb` | только auth-service |
| очередь и история уведомлений, настройки SMTP и каналов | PostgreSQL `notifications` | только notification-service |
| аватарки, вложения документов | bind-mount `auth-service/data` | auth-service |
| динамические конфиги nginx | том `nginx_dynamic_config` | auth-service (пишет), nginx (читает) |

Общей базы у сервисов нет: всё межсервисное — через HTTP с `X-API-Key`.

## Аутентификация между сервисами

| Направление | Механизм |
|---|---|
| бизнес-сервис → auth-service `/api/*` | `X-API-Key: INTERNAL_API_KEY` (общий) |
| сервис → notification-service | `X-API-Key` = свой `NOTIFICATION_API_KEY` из `SERVICE_API_KEYS` (**персональный**); фоллбек — легаси `INTERNAL_API_KEY` |
| auth-service → notification-bot | `X-API-Key: INTERNAL_API_KEY` |
| notification-bot → auth-service | `X-API-Key: INTERNAL_API_KEY` |
| notification-service → auth-service (резолв получателей) | `X-API-Key: INTERNAL_API_KEY` |
| auth-service → docker-socket-proxy | без ключа, ограничение сетью и набором разрешённых операций |

`INTERNAL_API_KEY` обязателен: `internalAPIKeyRequired()` в auth-service делает
`log.Fatal`, если переменная пуста, — чтобы `/api/*` не остался открытым.

## Уведомления: цепочка

```
сервис ──POST /api/v1/notifications──► notification-service
                                          │
                         login? ──────────┤ POST /api/recipients/resolve ──► auth-service
                                          │        (логин портала → email / chat_id)
                                          │
                                   очередь канала (PostgreSQL)
                                          │
                              ┌───────────┴───────────┐
                         email│                       │telegram
                              ▼                       ▼
                            SMTP                notification-bot ──► Telegram API
```

Ключевое: **вызывающий сервис не хранит контакты**. Он указывает `login`
пользователя портала, а адрес доставки для нужного канала подставляет
auth-service. Получателя вне портала задаёт отдельное поле
`external_recipient`. Подробности — в
[notification-service.md](notification-service.md).

## Безопасность контейнеров

Всем сервисам шлюза выставлено `security_opt: no-new-privileges:true`,
Go-сервисам дополнительно `cap_drop: ALL` и точечный `cap_add`
(`NET_BIND_SERVICE` для `:80`, `CHOWN`/`SETUID`/`SETGID`/`DAC_OVERRIDE` там,
где нужен bind-mount).

Доступ к докеру не прямой: `/var/run/docker.sock` смонтирован только в
`docker-socket-proxy`, который пропускает узкий набор операций
(см. [docker-socket-proxy.md](docker-socket-proxy.md)).
