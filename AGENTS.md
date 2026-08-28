# AGENTS.md — брифинг для ИИ-агента

Читай этот файл первым, если работаешь с платформой Golden House: со шлюзом
(`!gateway`) или с сервисом, который к нему подключается. Здесь нет
бизнес-логики конкретных сервисов — только устройство платформы, инварианты,
которые нельзя нарушать, и маршрут «задача → какой документ открыть».

Всё сверено с кодом. Если код и документ расходятся — прав код, а документ
надо поправить.

---

## 1. Что это за система за 10 строк

- Внешний трафик принимает **один** контейнер — `nginx` шлюза. Всё остальное
  живёт во внутренних docker-сетях и наружу не смотрит.
- Личность проверяет **auth-service**: nginx на каждый запрос делает
  `auth_request` в `GET /verify`, получает `200/401/403` и набор заголовков
  `X-User-*`, которые кладёт в запрос к сервису.
- **Конечный сервис не аутентифицирует пользователя.** Ни паролей, ни JWT, ни
  своей формы логина. Он читает заголовки и решает только «что этому набору
  прав можно».
- Права принадлежат сервисам (ADR-001): сервис объявляет каталог прав,
  auth-service его забирает, админ собирает из прав роли, роли выдаются людям
  внутри конкретного сервиса.
- Маршруты сервисов **генерируются автоматически** из реестра: сервис
  регистрируется по HTTP, auth-service пишет конфиг nginx и перезагружает его
  сигналом.
- Уведомления — только через `notification-service`; Telegram — только через
  `notification-bot`. Сервис не хранит ни SMTP-паролей, ни контактов людей.

Полная картина: [gateway/architecture.md](gateway/architecture.md).

---

## 2. Куда смотреть — маршрут по задачам

| Задача | Документ |
|---|---|
| подключить новый сервис к шлюзу | [integration/GATEWAY_SERVICE_INTEGRATION_API.md](integration/GATEWAY_SERVICE_INTEGRATION_API.md) |
| подключить `auth-connector` к Flask, поднять сервис локально | [integration/AUTH_CONNECTOR_SETUP_AND_LOCAL_DEV.md](integration/AUTH_CONNECTOR_SETUP_AND_LOCAL_DEV.md) |
| слать уведомления, в том числе в Telegram | [integration/TELEGRAM_NOTIFICATIONS_INTEGRATION_GUIDE.md](integration/TELEGRAM_NOTIFICATIONS_INTEGRATION_GUIDE.md) |
| понять сети, путь запроса, service discovery | [gateway/architecture.md](gateway/architecture.md) |
| роуты, права, миграции, MongoDB auth-service | [gateway/auth-service.md](gateway/auth-service.md) |
| конфиги nginx, rate-limit, заголовки | [gateway/nginx.md](gateway/nginx.md) |
| очереди доставки, приоритеты, лимиты каналов | [gateway/notification-service.md](gateway/notification-service.md) |
| Telegram: привязка, вход, лимиты Bot API | [gateway/notification-bot.md](gateway/notification-bot.md) |
| health-опрос и алерты | [gateway/monitoring-service.md](gateway/monitoring-service.md) |
| коллекции Mongo, бэкап, миграция на `--auth` | [gateway/mongo.md](gateway/mongo.md) |
| доступ к логам контейнеров | [gateway/dozzle.md](gateway/dozzle.md) |
| что разрешено делать с Docker API | [gateway/docker-socket-proxy.md](gateway/docker-socket-proxy.md) |
| авто-карантин по индикаторам компрометации | [gateway/guard-watchdog.md](gateway/guard-watchdog.md) |
| сервис с ручной маршрутизацией (пример) | [gateway/quiz.md](gateway/quiz.md) |

Карта всех сервисов шлюза с контейнерами и портами — [gateway/README.md](gateway/README.md).

---

## 3. Инварианты — нарушать нельзя

1. **Порт 80 внутри контейнера.** Реестр хранит `internal_url`, шаблон nginx
   подставляет его как есть.
2. **`service_key` == префикс URL.** `/verify` определяет сервис по первому
   сегменту пути. Сервис с ключом `finder` живёт на `/finder/...` и нигде
   больше. Ключ после заведения не меняют — это путь.
3. **`GET /health` → 200** без чувствительных данных. Его дёргают nginx,
   docker healthcheck и monitoring-service.
4. **`GET /api/sync/permissions`** отдаёт `{"success": true, "permissions":[…]}`
   по пути от корня приложения (без префикса сервиса), **без** `X-API-Key`.
5. **Проверка прав — только `permission_granted()`** из `auth_connector`.
   Прямое `if perm in permissions` ломается на wildcard `finder.*`.
6. **Никакой своей аутентификации.** Страницы логина/логаута заменяются
   редиректами на `/login` и `/logout`.
7. **Токена Telegram-бота в сервисе быть не должно.** Второй потребитель
   `getUpdates` ломает long polling бота портала.
8. **Пользовательские загрузки не в `static/`.** Шаблон nginx отдаёт
   `<prefix>/static/` **без** `auth_request`, а `<prefix>/static/uploads/`
   возвращает 404. Файлы — своим роутом с проверкой прав.
9. **CDN недоступен.** Прод в изолированной сети: ассеты вендорить в
   `static/vendor/` или брать из общего `/_shared/` шлюза. Разрешения CDN
   в CSP — наследие, а не разрешение ими пользоваться.
10. **`X-User-Admin` не существует.** Шаблон nginx явно затирает заголовок.
    Админ приходит обычной ролью `admin` с правом `<service>.*`.
11. **`X-User-Full-Name` — base64.** Признак кодировки в
    `X-User-Full-Name-Encoding`.
12. **Получателя уведомления задаёт `login` портала**, а не email и не
    telegram-ник. Внешний адресат — отдельным полем `external_recipient`.
    Заполнено должно быть ровно одно поле.

---

## 4. Контракт в двух экранах

### Заголовки, приходящие в сервис

| Заголовок | Что |
|---|---|
| `X-User-Name` | логин |
| `X-User-ID` | ObjectID |
| `X-User-Full-Name` + `-Encoding` | ФИО в base64 |
| `X-User-Email`, `X-User-Phone` | контакты |
| `X-User-Avatar` | путь `/avatar/<id>` — **абсолютный от корня домена** |
| `X-User-Service-Roles` | роли в этом сервисе |
| `X-User-Service-Permissions` | права в этом сервисе |
| `X-Service-Key`, `X-Service-Prefix`, `X-Forwarded-Prefix` | кто отвечает и под каким префиксом |
| `X-User-Passport-*`, `X-User-PINFL`, банковские | данные документов профиля |

Коды `auth_request`: `401` → редирект на `/login`; `403` (ни одной роли и ни
одного права в сервисе) → `/access-denied`; `200` → проксирование.

### Порядок подключения

```
1. Завести сервис в админке: https://analytics.gh.uz/services/new
   key == префикс URL. Без этой записи POST /api/registry/register вернёт 500.
2. Получить INTERNAL_API_KEY (тот же, что в !gateway/auth-service/.env).
3. Объявить права в permissions_setup.py, включая wildcard <key>.*
4. Поднять контейнер в service_network — auth-connector сам зарегистрируется,
   пойдёт heartbeat раз в 30 с и синхронизация прав.
5. Проверить: curl -H "X-API-Key: $INTERNAL_API_KEY" \
        http://auth-service:80/api/registry/services
6. Создать роли и раздать людям: /services/<key>
```

Развёрнуто, с формами и примерами — раздел 2
[integration/GATEWAY_SERVICE_INTEGRATION_API.md](integration/GATEWAY_SERVICE_INTEGRATION_API.md).

### Отправка уведомления

```json
POST http://notification-service:80/api/v1/notifications
X-API-Key: <персональный ключ сервиса>

{ "type": "email", "login": "ivanov", "subject": "Тема", "content": "Текст" }
```

`type`: `email` | `telegram` | `telegram_system` | `sms` | `push`.
Пачка — `POST /api/v1/notifications/batch`.

---

## 5. Эталоны для копирования

Когда нужно «как у всех», смотри на **`referal/`** и **`apartment_finder/`** —
они подключены к шлюзу по текущему контракту.

**Не бери за образец `quiz/`**: он намеренно живёт в изолированной сети
`quiz_network`, с ручной маршрутизацией (`unmanaged_routing: true`) и без
`auth_request` на публичных путях. Это исключение, а не шаблон.

---

## 6. Правила работы в этом репозитории

- **Git — только на чтение.** Коммиты, ветки, push и `git submodule add`
  делает человек. Агент готовит файлы и выдаёт команды, но не выполняет их.
- **Сабмодули тянуть по веткам.** `git submodule update` откатывает сабмодуль
  на записанный в родителе коммит и роняет прод. В сабмодуле — `git pull`
  в его ветке.
- **`env_file` не перечитывается при `docker compose restart`.** После правки
  `.env` нужен `docker compose up -d --force-recreate <service>`.
- **Ключ `X-API-Key` регистрировать до выдачи.** Сначала запись в
  `SERVICE_API_KEYS` у notification-service, потом `NOTIFICATION_API_KEY`
  у сервиса. Иначе первые запросы получат `401` и попадут в счётчик
  security guard'а.
- **В bash-командах путь `!gateway` требует `set +H`** — иначе история
  подстановок съест восклицательный знак. В PowerShell проблемы нет.
- **Мобильная вёрстка шлюза**: `static/css/mobile.css` грузится последним и
  одноэкранная раскладка включена только при `min-width: 769px`. Остальные
  сервисы этого слоя пока не имеют.
- **Библиотеку предпочитать самописному**: для сложного UI и алгоритмов брать
  готовое решение, а не писать с нуля. С поправкой на инвариант №9 — вендорить
  локально, не с CDN.

---

## 7. Диагностика: симптом → причина

| Симптом | Причина |
|---|---|
| `502` при живом контейнере | `internal_url` не резолвится: контейнер не в `service_network` или имя другое |
| `404` на всём сервисе | экземпляр дерегистрирован и конфиг перегенерирован; штатный рестарт даёт `502`, а не `404` |
| все страницы `401` | нет `AuthMiddleware` либо запрос идёт мимо nginx |
| `403` при выданном `<key>.*` | проверка через `in`, а не `permission_granted()` |
| карточки нет в `/menu` | у пользователя нет ни одной активной роли в сервисе |
| права не появились в админке | auth-service не достучался до `/api/sync/permissions` |
| ссылки без префикса | нет `ProxyFix(x_prefix=1)` / `PrefixMiddleware` |
| ФИО в кракозябрах | не декодирован base64 |
| `504` на тяжёлом отчёте | лимит nginx 60 с — нужна асинхронная выгрузка |
| `400 user_not_found` на живом сотруднике | в `login` ушёл email или telegram-ник вместо логина портала |
| разом `no_service_access` | `SERVICE_AUTH_KEY_MAP` не сопоставил имя ключа с `service_key` |
| `503 auth_unavailable` | auth-service недоступен, уведомление **не создано** |
| «Email sent», но письма нет | включён `debug_mode` — письма уходят на debug-адрес |
| адреса в логах как `d.***@gh.uz` | маскировка; на время разбора `LOG_FULL_RECIPIENTS=true` |
| пустой список в `/services/<key>/logs` | фильтр `name=<key>*` не совпал с именем контейнера |

Логи:

```bash
docker logs -f gateway-auth-service-1     # авторизация, /verify, реестр
docker logs -f gateway-nginx-1            # маршрутизация
docker logs -f notification-service-notification-service-1
docker logs -f gateway-notification-bot
```

---

## 8. Что в системе сломано или отключено

Учитывать при работе, не принимать за норму:

- `/roles`, `/permissions`, `/admin-menu` — nginx их проксирует, роутов в
  auth-service нет (`routes/role_management.go.disabled`), отдают `404`.
- `POST /api/v1/security/alert` в notification-service **не реализован**, хотя
  `watchdog/watchdog.sh` шлёт туда алерты. Сам watchdog ни в одном
  compose-файле не описан, то есть не запущен.
- В `nginx/conf/quiz.inc` проверка `Origin`/`Referer` закомментирована
  «временно для тестирования», `$quiz_cors_origin` зеркалит любой Origin,
  у quiz-backend `BACKEND_CORS_ORIGINS=*`.
- `notification-service/README.md` называет порт `8082` — фактически `:80`.
- `getAllowedLogsServices` содержит захардкоженный список сервисов
  (`referal`, `client-service`, `notification`, `monitoring`); новый сервис
  туда надо дописывать руками.

---

## 9. Чеклист перед сдачей интеграции

```
[ ] Сервис заведён в /services/new: key == префикс URL
[ ] Контейнер слушает :80 и подключён к service_network
[ ] GET /health → 200, без чувствительных данных
[ ] GET /api/sync/permissions → {"success": true, "permissions": [...]}
[ ] permissions_setup.py объявляет права, включая <key>.*
[ ] Проверки прав идут через permission_granted()
[ ] ProxyFix(x_prefix=1) + PrefixMiddleware настроены
[ ] Свои страницы логина/логаута заменены редиректами на /login, /logout
[ ] Пользовательские загрузки лежат вне static/
[ ] Ассеты вендорены локально
[ ] INTERNAL_API_KEY задан; персональный ключ notification-service получен
[ ] Токена Telegram-бота в сервисе нет
[ ] Экземпляр виден в GET /api/registry/services
[ ] Права видны на /services/<key>, роли созданы и назначены
```
