# auth-service

Ядро шлюза: единственный сервис, который знает пользователей, роли, права и
состав платформы. Всё остальное — nginx, бизнес-сервисы, боты — спрашивает у него.

- **Каталог:** `auth-service/`
- **Технология:** Go 1.23, Gin, MongoDB (официальный драйвер), JWT (`dgrijalva/jwt-go`)
- **Контейнер:** `gateway-auth-service-1`, слушает `:80`, наружу не публикуется
- **Сети:** `service_network`, `data_network`
- **Тома:** `auth-service/data` (аватарки, вложения), `auth-service/static`,
  `auth-service/templates`, том `nginx_dynamic_config` (на запись)
- **Healthcheck:** `GET /health`

## Зоны ответственности

1. **Аутентификация** — логин/пароль, JWT в cookie `token`, блэклист токенов,
   восстановление пароля, вход и сброс пароля через Telegram.
2. **Авторизация** — ADR-001: права принадлежат сервисам, у пользователя роли
   *внутри* сервиса. Точка принятия решения для nginx — `GET /verify`.
3. **Реестр сервисов** — регистрация инстансов, heartbeat, генерация конфигов
   nginx и его перезагрузка.
4. **Административная панель** — пользователи, сервисы, роли, права, документы,
   Excel-импорт/экспорт, настройки уведомлений.
5. **Личный кабинет** — профиль, аватар, документы, привязка Telegram.
6. **Резолв получателей уведомлений** — логин портала → адрес доставки для
   нужного канала.

## Структура каталога

```
auth-service/
├── main.go                  # старт: подключение к Mongo, миграции, роутер
├── notification_client.go   # клиент notification-service
├── notification_config.go   # синхронизация SMTP-настроек в notification-service
├── telegram_bot_client.go   # клиент notification-bot
├── permissions.json         # собственный каталог прав (отдаётся по /api/sync/permissions)
├── routes/                  # HTTP-слой (роуты, хендлеры, middleware)
├── models/                  # доменные модели + доступ к MongoDB
├── handlers/                # Excel-импорт/экспорт
├── migrations/              # разовые миграции схемы
├── utils/file_security.go   # защита от path traversal при отдаче файлов
├── templates/               # HTML (Go templates)
├── static/                  # css/js/шрифты/вендоренный Font Awesome
└── data/                    # аватарки и вложения (bind-mount)
```

## Что происходит при старте

`main.go` выполняет по порядку — все шаги идемпотентны:

1. Подключение к MongoDB (`MONGO_URI` обязателен, иначе `log.Fatal`).
2. `EnsureAdminExists()` — создаёт админа, если его нет.
3. `EnsureCriticalRolesIntegrity()` — роли `GOD` и `admin`, миграция
   `system/admin` → `auth/GOD`, починка легаси-форматов.
4. `EnsureExternalRolesForAllServices()` — внешние роли и права для всех
   зарегистрированных сервисов (важно после восстановления из бэкапа).
5. `EnsureServiceAdminRoles()` — выдаёт системным админам явную роль `admin`
   с правом `<service>.*` в каждом сервисе. **Без этого шага админ остался бы
   без доступа к сервисам**: обход по заголовку `X-User-Admin` отменён.
6. `MigrateToADR001Schema()` + `ValidateMigration()`.
7. `MigrateUserNamesFromFullName()` — расщепление ФИО на поля.
8. `MigrateDocumentsToServicesField()` — поле `allowed_services` у документов.
9. `CleanupOrphanedUserServiceRoles()` — чинит `client` → `client-service`,
   выбрасывает пустые и осиротевшие записи.
10. `checkAndCleanupAvatars()` — чистит пути к аватаркам, файлов которых нет.
11. Инициализация клиентов уведомлений и Telegram-бота, отправка текущих
    SMTP-настроек в notification-service.
12. `InitializeNginxConfig()` — создаёт пустой `dynamic/services.conf`,
    если его ещё нет.
13. `routes.SetupAllRoutes()` + фоновый health-монитор сервисов.

Ошибка миграции не валит старт — только пишется в лог.

## Модель прав (ADR-001)

Полное решение — в [ADR-001](../../!gateway/ADR-001-service-based-authorization.md),
руководство по гранулярным правам — в
[SERVICE_PERMISSIONS_GUIDE.md](../../!gateway/SERVICE_PERMISSIONS_GUIDE.md).
Кратко:

- **Сервис** (`services`) владеет каноническим списком своих прав
  (`availablePermissions`), которые сам же и присылает по
  `GET /api/sync/permissions`.
- **Роль** (`service_roles`) — именованный набор прав внутри одного сервиса.
- **Назначение** (`user_service_roles`) связывает пользователя, сервис и роль.
- Право именуется `<service>.<область>.<действие>`, wildcard —
  `<service>.*` и `auth.*`.

### Уровни администратора

| Кто | Признак | Может |
|---|---|---|
| GOD | роль `auth/GOD` | всё; создаётся при старте |
| системный админ | право `auth.*` (или легаси `system/admin`, `auth/admin`) | всё в админке |
| админ сервиса | роль `admin` в сервисе | управлять своим сервисом |
| service-manager | роль `service-manager` в сервисе или в `auth` | назначать пользователей на роли своего сервиса (см. [SERVICE_MANAGER_ROLE.md](../../!gateway/SERVICE_MANAGER_ROLE.md)) |
| внешняя роль | роль в `auth` с правами `auth.<service>.*` | ограниченное управление чужим сервисом |

### Собственные права auth-service (`permissions.json`)

| Категория | Права |
|---|---|
| dashboard | `auth.dashboard.view` |
| users | `view`, `create`, `edit`, `delete`, `reset_password`, `assign_roles` |
| services | `view`, `create`, `edit`, `delete`, `sync_permissions` |
| roles | `view`, `create`, `edit`, `delete`, `assign_permissions` |
| permissions | `view`, `manage` |
| documents | `view`, `manage` |
| document_types | `view`, `manage` |
| logs | `auth.logs.view`, `auth.logs.export` |
| settings | `view`, `edit` |
| notifications | `receive`, `send` |
| import_export | `auth.import.execute`, `auth.export.execute` |
| api | `auth.api.access` |
| system | `auth.system.monitor` |

Дополнительно проверяются, но в каталоге не объявлены:
`auth.logs.system.view` и `auth.<service>.logs.view` — доступ к логам
(см. [dozzle.md](dozzle.md)).

## Middleware

| Middleware | Что проверяет | Отказ |
|---|---|---|
| `authRequired()` | валидный JWT в cookie `token`, пользователь существует | `302 /login?redirect=…` |
| `adminAuthRequired()` | у пользователя есть **хоть одно** право `auth.*`; кладёт в контекст `authPermissions` и `isSystemAdmin` | `403` HTML |
| `serviceAdminAuthRequired()` | системный админ **или** админ / service-manager / внешняя роль / `auth.<key>.*` для сервиса из `:serviceKey` (или `:id`) | `403` HTML |
| `internalAPIKeyRequired()` | заголовок `X-API-Key` == `INTERNAL_API_KEY` | `401` JSON |
| `RateLimitMiddleware()` | лимит попыток логина | `429` |
| `telegramLoginRateLimit()` | лимит стартов входа через Telegram | `429` |

Роли пользователя внутри одного запроса читаются один раз и кэшируются в
`gin.Context` (`getUserRolesCached`) — иначе цепочка проверок давала бы N+1
запросов в Mongo.

## HTTP API

### Публичные страницы и аутентификация

| Метод | Путь | Назначение |
|---|---|---|
| GET | `/` | редирект на `/menu` |
| GET | `/health` | healthcheck |
| GET/POST | `/login` | вход по логину и паролю (rate-limit) |
| GET | `/logout` | сброс cookie |
| GET/POST | `/forgot-password` | запрос ссылки восстановления (email или Telegram) |
| GET/POST | `/reset-password` | установка нового пароля по токену |
| GET | `/login/telegram` | страница входа через Telegram |
| POST | `/login/telegram` | старт запроса подтверждения (rate-limit) |
| GET | `/login/telegram/status` | опрос статуса подтверждения |
| GET | `/access-denied` | страница отказа |

### Внутренние проверки для nginx (`internal`)

| Метод | Путь | Что делает |
|---|---|---|
| GET | `/verify` | основной `auth_request`: сервис определяется по `X-Original-URI`, в ответ уходят заголовки `X-User-*` |
| GET | `/verify-admin` | проверка «только админ» |
| GET | `/verify-logs-access` | доступ к логам + заголовки `Remote-User/Email/Name/Filter` для Dozzle |

### Пользовательские страницы

`/menu`, `/settings`, `/profile` (+ `/profile/avatar`, `/profile/password`,
`/profile/documents/*`, `/profile/telegram/link|unlink`),
`/document-types`, `/available-services` — всё под `authRequired()`.
Подробности личного кабинета — в
[PROFILE_FEATURES.md](../../!gateway/auth-service/PROFILE_FEATURES.md).

### Администрирование пользователей — `/users/*` (`adminAuthRequired`)

Список и карточка, создание/редактирование/удаление, бан/разбан, сброс пароля,
аватарки, документы и вложения, назначение и снятие ролей
(`/users/:id/roles/data|assign|remove`), Excel-импорт/экспорт
(`/users/import`, `/users/export`, `/users/template`, `/users/import/logs`).

### Администрирование сервисов — `/services/*` (`serviceAdminAuthRequired`)

| Метод | Путь | Назначение |
|---|---|---|
| GET | `/services/` | список сервисов с доступом |
| GET/POST | `/services/new`, `POST /services/` | создание сервиса |
| GET/POST | `/services/:serviceKey` | карточка и редактирование |
| POST | `/services/:serviceKey/delete` \| `/restore` \| `/hard-delete` | мягкое удаление, восстановление, полное удаление (только системный админ) |
| POST/PUT/POST | `/services/:serviceKey/permissions[/:permName[/delete]]` | правка каталога прав |
| POST | `/services/:serviceKey/permissions/sync` | подтянуть права из самого сервиса |
| POST/GET/POST | `/services/:serviceKey/roles[/:roleId[/delete]]` | роли сервиса |
| POST | `/services/:serviceKey/assign-role` | назначить пользователя на роль |
| GET/POST/PUT/DELETE | `/services/:serviceKey/external-roles/…` | внешние роли |
| GET/POST/PUT | `/services/:serviceKey/users[/:userId/roles]` | пользователи сервиса |
| GET/POST | `/services/:serviceKey/import` \| `/export` \| `/template` \| `/import/logs` | Excel по сервису ([SERVICE_IMPORT_EXPORT.md](../../!gateway/auth-service/SERVICE_IMPORT_EXPORT.md)) |
| GET | `/check-user-exists` | проверка существования пользователя |

Настройки уведомлений: `GET/POST /notification-settings`,
`POST /notification-settings/test`, `GET /notification-settings/channels`,
`POST /notification-settings/channels/:channel` — все под `adminAuthRequired()`.

Миграции: `/migration/` (`run`, `validate`, `rollback`, `cleanup`).

### Межсервисный API — `/api/*` (требует `X-API-Key`)

| Метод | Путь | Назначение |
|---|---|---|
| GET | `/api/test` | проверка ключа |
| GET | `/api/sync/permissions` | каталог прав самого auth-service |
| POST | `/api/services/:serviceKey/permissions/sync` | приём каталога прав от сервиса |
| GET | `/api/services/:serviceKey/users` | пользователи сервиса |
| GET | `/api/services/:serviceKey/users-by-role/:roleName` | пользователи с ролью |
| GET | `/api/services/:serviceKey/users-by-permission/:permissionName` | пользователи с правом |
| GET | `/api/users/:userId/profile` | профиль (ФИО, контакты, паспорт, адрес) |
| GET | `/api/users/:userId/documents` \| `/grouped` \| `/for-service/:serviceKey` | документы пользователя |
| POST | `/api/users/:userId/documents` | создать документ |
| GET | `/api/users/:userId/documents/:docId/attachments/:attachmentId/download` | скачать вложение |
| GET | `/api/auth/roles/:roleName` \| `/users` | роль auth-service и её носители |
| POST | `/api/telegram/link/confirm` | подтверждение привязки (от бота) |
| POST | `/api/telegram/login/decision` | решение по входу (от бота) |
| POST | `/api/telegram/link/broken` | бот заблокирован пользователем |
| GET | `/api/telegram/chat-id` | легаси-резолв username → chat_id |
| POST | `/api/recipients/resolve` | логин портала → адрес доставки |
| POST | `/api/registry/register` | регистрация инстанса |
| DELETE | `/api/registry/unregister/:serviceKey` | снятие с учёта |
| POST | `/api/registry/heartbeat` | heartbeat |
| GET | `/api/registry/services[/:serviceKey]` | список инстансов |

Отдельно: `GET /api/services/health` — **не** требует `X-API-Key`, но требует
залогиненного пользователя (`authRequired`); её дёргают карточки сервисов в UI.

### `POST /api/recipients/resolve`

```json
{ "service": "referal", "channel": "email", "logins": ["ivanov", "petrov"] }
```

Ответ:

```json
{
  "success": true,
  "channel": "email",
  "access_enforced": false,
  "results": {
    "ivanov": { "found": true, "address": "i.ivanov@gh.uz", "has_service_access": true },
    "petrov": { "found": false, "reason": "no_address", "has_service_access": true }
  }
}
```

Каналы: `email`, `telegram` (адрес = `chat_id`), `sms` (адрес = телефон).
Матч идёт **строго по `username`** — email не уникален как ключ адресации,
а telegram-ник пользователь меняет в один клик.

Коды отказа (`reason`): `user_not_found`, `user_banned`, `no_service_access`,
`channel_not_linked`, `no_address`, `unknown_channel`.

Ограничения и режимы:

- максимум **500** логинов в запросе — безлимитный `$in` даёт скан коллекции;
- `RECIPIENT_ACCESS_ENFORCE=false` (по умолчанию) — **режим наблюдения**:
  получатель без ролей в сервисе-отправителе всё равно резолвится, но в лог
  идёт `⚠️ RECIPIENT ACCESS`. Включать только после того, как логи покажут,
  что легитимные рассылки (алерты админам, письма колл-центру) от этого
  не пострадают;
- эндпоинт раздаёт контакты пользователей и наружу не публикуется.

## Генерация конфигов nginx

`routes/nginx_config.go`. При регистрации инстанса:

1. `GetActiveServicesForNginx()` — все сервисы со статусом ≠ `deleted`,
   у которых есть хоть один известный инстанс, кроме помеченных
   `unmanaged_routing`.
2. Старые `service-*.conf` в `NGINX_DYNAMIC_CONFIG_DIR` удаляются.
3. На каждый сервис пишется `service-<key>.conf` из шаблона: блок
   `location <prefix>/` с `auth_request /verify`, `location <prefix>/static/`
   **без** авторизации (нужен странице логина), `location <prefix>/static/uploads/`
   → `404` (загрузки отдаются отдельным роутом сервиса через общую проверку),
   и `location = <prefix><health_path>` без логов.
4. Пишется master-файл `services.conf` с `include` всех сервисных конфигов.
5. `POST http://docker-socket-proxy:2375/containers/<NGINX_CONTAINER_NAME>/kill?signal=HUP`
   — мягкая перезагрузка nginx. Через `kill?signal=`, а не `exec`, чтобы у
   прокси можно было держать `EXEC=0`.

В имени nginx-переменной дефисы заменяются на подчёркивания
(`client-service` → `$backend_client_service`).

## MongoDB

База `authdb`, пользователь `authservice` с `readWrite` (создаётся
`scripts/mongo-init.js` при первой инициализации тома). Коллекции:

| Коллекция | Содержимое |
|---|---|
| `users` | пользователи, профиль, документы, привязка Telegram |
| `roles`, `permissions` | легаси-справочники |
| `services` | сервисы и их каталоги прав |
| `service_roles` | роли внутри сервисов |
| `user_service_roles` | назначения «пользователь × сервис × роль» |
| `service_instances` | реестр живых инстансов (service discovery) |
| `user_sessions` | сессии |
| `blacklisted_tokens` | отозванные JWT |
| `password_reset_tokens` | токены восстановления пароля |
| `telegram_link_tokens`, `telegram_login_requests` | привязка и вход через Telegram (TTL-индексы) |
| `activity_logs`, `import_logs`, `service_import_logs` | аудит и логи импорта |
| `document_types` | справочник типов документов |

## Переменные окружения

Файл `auth-service/.env` (шаблон — `.env.example`). Обязательные помечены **\***.

| Переменная | Назначение |
|---|---|
| `MONGO_URI` **\*** | строка подключения с логином/паролем; без неё `log.Fatal` |
| `MONGO_DB` | база, по умолчанию `authdb` |
| `JWT_SECRET` **\*** | ≥32 символов; **должен совпадать** с `client_service` и `referal` |
| `INTERNAL_API_KEY` **\*** | ключ `/api/*`; пустое значение → `log.Fatal` при первом запросе |
| `ENVIRONMENT` | `production` включает `Secure` у cookie и глушит подробные логи |
| `BASE_URL` | базовый URL для ссылок в письмах (без порта) |
| `ALLOWED_ORIGIN` | список разрешённых CORS-origin через запятую |
| `NOTIFICATION_SERVICE_URL` | по умолчанию `http://notification-service:80` |
| `NOTIFICATION_API_KEY` | персональный ключ в notification-service; без него шлётся `INTERNAL_API_KEY` и пер-сервисный лимит не работает |
| `NOTIFICATION_BOT_URL` | по умолчанию `http://notification-bot:80` |
| `TELEGRAM_BOT_USERNAME` | имя бота без `@` для deep-link |
| `RECIPIENT_ACCESS_ENFORCE` | блокировать ли отправку получателю без ролей в сервисе-отправителе (по умолчанию `false`) |
| `NGINX_CONTAINER_NAME` | кого перезагружать (по умолчанию `gateway-nginx-1`) |
| `NGINX_DYNAMIC_CONFIG_DIR` | по умолчанию `/etc/nginx/conf.d/dynamic` |
| `DOCKER_HOST` | по умолчанию `tcp://docker-socket-proxy:2375` |
| `MONITORING_ENABLED`, `MONITOR_SEND_NOTIFICATIONS`, `MONITOR_CHECK_INTERVAL_SECONDS`, `MONITOR_ALERT_COOLDOWN_MINUTES`, `MONITOR_ADMIN_EMAIL`, `ADMIN_TELEGRAM_CHAT_ID` | встроенный мониторинг (отдельно от [monitoring-service](monitoring-service.md)) |
| `ADMIN_EMAIL`, `SUPPORT_EMAIL`, `SUPPORT_TELEGRAM` | контакты в письмах и на страницах |

`docker-compose.yaml` дополнительно задаёт `TZ=Asia/Tashkent` и перекрывает
`NOTIFICATION_SERVICE_URL`, `NOTIFICATION_BOT_URL`, `NGINX_*`, `DOCKER_HOST`.

## Грабли

- **`env_file` не перечитывается при `docker compose restart`.** После правки
  `.env` нужен `docker compose up -d --force-recreate auth-service`.
- **`X-User-Admin` больше не работает.** Сервис, ожидающий этот заголовок,
  получит пустую строку: шаблон nginx явно его затирает.
- **Пустой список ролей = 403.** Прежняя аварийная ветка «системному админу
  выдать роль admin на лету» убрана — она подставляла роль с пустым списком
  прав, и сервис показывал пустое приложение вместо честного отказа.
  Доступ админа обеспечивает `EnsureServiceAdminRoles()` при старте.
- **Смена `service_key` осиротит назначения.** `CleanupOrphanedUserServiceRoles()`
  чинит только известный случай `client` → `client-service`.
- **`/roles`, `/permissions`, `/admin-menu`** отдают 404: nginx их проксирует,
  но `routes/role_management.go.disabled` не компилируется.
- **Шрифтовые MIME-типы регистрируются вручную** в `main.go` — в alpine нет
  `/etc/mime.types`, и вендоренный Font Awesome иначе уходил бы как
  `application/octet-stream`.
