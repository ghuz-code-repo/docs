# Подключение сервиса к шлюзу: полный справочник API

Что конечный сервис получает от шлюза, что обязан отдать взамен и какими HTTP-вызовами
может пользоваться. Всё описанное сверено с кодом на 2026-08-26.

Смежные документы:

* [AUTH_CONNECTOR_SETUP_AND_LOCAL_DEV.md](AUTH_CONNECTOR_SETUP_AND_LOCAL_DEV.md) — как подключить
  `auth-connector` и тестировать сервис локально.
* [AUTH_CONNECTOR_REFERENCE.md](AUTH_CONNECTOR_REFERENCE.md) — справочник по API пакета:
  классы, декораторы, сигнатуры, что устарело.
* [TELEGRAM_NOTIFICATIONS_INTEGRATION_GUIDE.md](TELEGRAM_NOTIFICATIONS_INTEGRATION_GUIDE.md) —
  Telegram-уведомления в подробностях.
* [!gateway/ADR-001-service-based-authorization.md](../../!gateway/ADR-001-service-based-authorization.md) —
  решение по модели прав.
* [../AGENTS.md](../AGENTS.md) — брифинг для ИИ-агента: инварианты, обязательные тесты,
  требования к документации в сервисе.

---

## 0. Кто есть кто

```
браузер ──HTTPS──► nginx ──auth_request──► auth-service (/verify)
                     │                          │  MongoDB: users, services,
                     │  заголовки X-User-*      │  roles, user_service_roles,
                     ▼                          │  service_instances
              ваш сервис :80 ◄─── pull прав ────┘
                     │
                     ├──X-API-Key──► auth-service /api/*        (пользователи, документы, реестр)
                     └──X-API-Key──► notification-service /api/v1 (email, Telegram)
```

| Компонент | Роль | Код |
|---|---|---|
| `nginx` | единственная точка входа, TLS, `auth_request`, генерация роутов сервисов | [!gateway/nginx/conf/gateway.inc](../../!gateway/nginx/conf/gateway.inc) |
| `auth-service` (Go) | пользователи, роли, права, документы, реестр сервисов, генератор nginx-конфигов | [!gateway/auth-service/routes/](../../!gateway/auth-service/routes/) |
| `notification-service` (Go) | очередь email/Telegram, ретраи, статусы | [!gateway/notification-service/main.go](../../!gateway/notification-service/main.go) |
| `notification-bot` (Go) | единственный владелец токена Telegram-бота | [!gateway/notification-bot/main.go](../../!gateway/notification-bot/main.go) |
| `monitoring-service` (Go) |health-опрос сервисов, алерты | [!gateway/monitoring-service/main.go](../../!gateway/monitoring-service/main.go) |
| `dozzle` | просмотр логов контейнеров через `/services/{key}/logs` | контейнер `gateway-dozzle` |
| ваш сервис | бизнес-логика; своей аутентификации не имеет | — |

---

## 1. Минимальный контракт сервиса

Четыре обязательства. Без них подключение не работает.

| # | Требование | Проверяется |
|---|---|---|
| 1 | Слушать **порт 80** внутри контейнера | `internal_url` при регистрации |
| 2 | Отдавать `GET /health` → `200` | nginx-роут, docker healthcheck, monitoring-service |
| 3 | Отдавать `GET /api/sync/permissions` → `{"success": true, "permissions": [...]}` | auth-service тянет права отсюда |
| 4 | Корректно работать под префиксом `/{service_key}` — читать `X-Forwarded-Prefix` | ссылки, редиректы, статика |

Плюс два соглашения:

* **`service_key` == внешний префикс URL.** `/verify` определяет сервис по первому сегменту
  `X-Original-URI` ([auth_handlers.go:180-196](../../!gateway/auth-service/routes/auth_handlers.go#L180-L196)).
  Сервис с ключом `finder` живёт на `/finder/...` и никак иначе.
* **Контейнер в сети `service_network`** (внутренняя, без выхода наружу). Нужен интернет —
  добавить `egress_network`.

### 1.1. Формат `/api/sync/permissions`

Тянет [service_management.go:1234](../../!gateway/auth-service/routes/service_management.go#L1234) — обычным
`GET` **без** `X-API-Key`, таймаут 30 с, по адресу `{internal_url}/api/sync/permissions`
(путь от корня приложения, **без** префикса `/finder`).

```json
{
  "success": true,
  "service_key": "finder",
  "permissions": [
    {"name": "finder.*",             "displayName": "Все права сервиса", "description": "Полный доступ", "category": "manage"},
    {"name": "finder.selection_view","displayName": "Подбор: просмотр",  "description": "Доступ к подбору", "category": "view"}
  ]
}
```

Парсер читает только `name`, `displayName`, `description` — `category` в `PermissionDef` при пуле
теряется. При `success: false` берётся поле `error`.

---

## 2. Порядок подключения нового сервиса

Шаги строго по порядку — регистрация экземпляра падает, если сервиса нет в базе.

**Шаг 1. Завести сервис в админке.**
`https://analytics.gh.uz/services/new` (нужна роль системного администратора). Заполнить:

| Поле | Смысл |
|---|---|
| `key` | ключ и префикс URL, например `finder`. Менять потом нельзя — это путь. |
| `name` | заголовок карточки в `/menu` |
| `description` | подпись карточки |
| `icon` | имя иконки Font Awesome (`building`, `gift`, `users`) |
| `menu_url` | нестандартная ссылка карточки; пусто — `/{key}/` |
| `unmanaged_routing` | `true` — nginx-роут **не** генерировать (маршрут описан руками в `.inc`, как у `quiz`) |

Без записи в коллекции `services` вызов `POST /api/registry/register` вернёт 500
`service {key} not found in services collection`
([service_registry.go:34](../../!gateway/auth-service/models/service_registry.go#L34)).

**Шаг 2. Получить `INTERNAL_API_KEY`.** Тот же ключ, что в `.env` у auth-service. Им подписаны
все вызовы `/api/*`.

**Шаг 3. Объявить права.** `permissions_setup.py` с `PermissionRegistry` — см.
[AUTH_CONNECTOR_SETUP_AND_LOCAL_DEV.md](AUTH_CONNECTOR_SETUP_AND_LOCAL_DEV.md), раздел 4.

**Шаг 4. Поднять контейнер** в `service_network`. При старте `auth-connector` сам:
регистрирует экземпляр, шлёт heartbeat раз в 30 с, дёргает синхронизацию прав.

**Шаг 5. Проверить регистрацию.**

```bash
curl -H "X-API-Key: $INTERNAL_API_KEY" http://auth-service:80/api/registry/services
```

**Шаг 6. Создать роли и раздать их людям.** `https://analytics.gh.uz/services/{key}` — вкладки
Permissions / Roles / Users.

При старте auth-service сам создаёт роль `admin` с правом `{key}.*` для каждого сервиса и
навешивает её всем системным администраторам
([migration.go:122](../../!gateway/auth-service/models/migration.go#L122)).

---

## 3. Что шлюз кладёт в каждый запрос к сервису

Формирует [auth_handlers.go verifyHandler](../../!gateway/auth-service/routes/auth_handlers.go#L154),
пробрасывает шаблон nginx [nginx_config.go:22](../../!gateway/auth-service/routes/nginx_config.go#L22).

### 3.1. Доходят до сервиса

| Заголовок | Значение | Формат |
|---|---|---|
| `X-User-ID` | id пользователя | hex ObjectID MongoDB, 24 символа |
| `X-User-Name` | логин | plain |
| `X-User-Full-Name` | ФИО | **base64** UTF-8 |
| `X-User-Full-Name-Encoding` | `base64` | признак кодировки |
| `X-User-Email` | email | plain, только если заполнен |
| `X-User-Phone` | телефон | plain, только если заполнен |
| `X-User-Avatar` | путь к аватару | абсолютный путь от корня домена, префикс сервиса не нужен |
| `X-User-Service-Roles` | роли **в этом** сервисе | CSV: `admin,manager` |
| `X-User-Service-Permissions` | права **в этом** сервисе | CSV: `finder.*` или `finder.selection_view,finder.discounts_view` |
| `X-User-Roles` | легаси-дубль `X-User-Service-Roles` | CSV |
| `X-Service-Key` | ключ сервиса | константа из конфига |
| `X-Service-Prefix` | внешний префикс | `/finder` |
| `X-Forwarded-Prefix` | то же, для `ProxyFix`/`PrefixMiddleware` | `/finder` |
| `X-Original-URI` | исходный URI **с** префиксом | `/finder/reports/plan-fact?year=2026` |
| `X-Forwarded-For`, `X-Real-IP`, `X-Forwarded-Proto`, `Host` | стандартные | |

Пустое значение nginx не передаёт вовсе — заголовка с пустой строкой не будет.

### 3.2. НЕ доходят — распространённая ошибка

`/verify` вычисляет и отдаёт nginx ещё девять заголовков, но шаблон их **не пробрасывает**:

```
X-User-Passport-Number   X-User-Passport-Giver   X-User-Passport-Date
X-User-Passport-Address  X-User-PINFL            X-User-Bank-Name
X-User-Bank-Card         X-User-Bank-MFO         X-User-Bank-Account
```

Паспорт, ПИНФЛ и банковские реквизиты берутся только через
`GET /api/users/{id}/documents/for-service/{key}` (раздел 5.4).

Ещё два заголовка всегда пустые:

* `X-User-Permissions` — `/verify` его не устанавливает никогда.
* `X-User-Admin` — отменён намеренно; nginx явно затирает его пустой строкой, чтобы клиентский
  заголовок не прошёл насквозь. **Признака администратора у сервиса нет**: администратор носит
  обычное право `{key}.*` в `X-User-Service-Permissions`.

### 3.3. Когда запрос до сервиса не доходит

| Ситуация | Ответ `/verify` | Что видит пользователь |
|---|---|---|
| нет cookie `token` / токен невалиден / в чёрном списке | 401 | редирект `302 /login?redirect=...` |
| токен валиден, но у пользователя **ноль** ролей и **ноль** прав в этом сервисе | 403 | редирект `302 /access-denied?service=...` |
| всё в порядке | 200 + заголовки | запрос уходит в сервис |

---

## 4. Модель прав

Три коллекции MongoDB: `services` (каталог прав), `roles` (роль = набор прав внутри сервиса),
`user_service_roles` (кто какую роль имеет).

```
users ──user_service_roles──► roles ──permissions[]──► services.availablePermissions
        (user, service_key,     (service_key, name,
         role_name, is_active)   permissions[])
```

Права пользователя = объединение `permissions` всех его **активных** ролей в этом сервисе
([service.go:797](../../!gateway/auth-service/models/service.go#L797)). Ни ролевой иерархии, ни
наследования нет.

### 4.1. Шаблоны прав

Разворачивания в список не происходит: право `finder.*`, выданное роли, приезжает в заголовок
строкой `finder.*` как есть. Сравнение —
[permission_utils.py](../../auth-connector/auth_connector/permission_utils.py):

| Право у пользователя | Покрывает |
|---|---|
| `finder.selection_view` | ровно себя |
| `finder.*` | всё, что начинается с `finder.` |
| `*` | всё |

Отсюда правило: **проверяйте права только через `permission_granted()` из `auth_connector`**.
Проверка `if perm in user.permissions` пропустит владельца `finder.*` и отдаст 403 при формально
выданном доступе — ровно эта ошибка и породила общий модуль.

### 4.2. Именование

Соглашение флота: `{service_key}.{модуль}_{сущность}_{действие}`, действие из
`view | create | update | delete | export | import | calculate`. Плюс обязательное `{key}.*` —
им живёт роль `admin`.

Внутри сервиса удобно держать короткие локальные имена (`selection_view`) и `PERMISSION_MAP`
локальное → шлюзовое (`finder.selection_view`); так сделано в
[apartment_finder/permissions_setup.py:180](../../apartment_finder/permissions_setup.py#L180).

### 4.3. Внешние права и внешние роли

У `PermissionDef` есть флаг `external`. Внешние права живут в сервисе `auth` с именами
`auth.{service_key}.*` и управляют не доступом к вашему сервису, а правом **администрировать**
его из панели шлюза:

| Право | Даёт |
|---|---|
| `auth.{key}.roles.assign` | назначать роли вашего сервиса пользователям |
| `auth.{key}.logs.view` | смотреть логи контейнеров вашего сервиса |
| `auth.{key}.*` | всё вышеперечисленное |
| `auth.*` | системный администратор |

Отдельно есть роль `service-manager` (в вашем сервисе или глобально в `auth`) — управление
сервисом без прав системного администратора.

---

## 5. HTTP API auth-service для сервисов

База: `http://auth-service:80` (внутри `service_network`).
Аутентификация: заголовок **`X-API-Key: $INTERNAL_API_KEY`** на всех `/api/*`.
Без ключа — `401 {"error": "Неверный или отсутствующий API-ключ"}`.
Если `INTERNAL_API_KEY` не задан в окружении auth-service, он падает при старте (`log.Fatal`) —
незащищённых `/api/*` не бывает.

Все ответы — JSON, тексты ошибок на русском.

### 5.1. Реестр сервисов (Service Discovery)

`auth-connector` делает это сам; ручные вызовы нужны для диагностики.

| Метод | Путь | Назначение |
|---|---|---|
| `POST` | `/api/registry/register` | зарегистрировать экземпляр, перегенерировать nginx-конфиг, послать SIGHUP |
| `POST` | `/api/registry/heartbeat` | продлить жизнь экземпляра |
| `DELETE` | `/api/registry/unregister/{key}?container_name=...` | снять экземпляр (nginx-роут **остаётся**) |
| `GET` | `/api/registry/services` | все активные экземпляры |
| `GET` | `/api/registry/services/{key}` | экземпляры одного сервиса |

```bash
curl -X POST http://auth-service:80/api/registry/register \
  -H "X-API-Key: $INTERNAL_API_KEY" -H "Content-Type: application/json" \
  -d '{
    "service_key": "finder",
    "container_name": "apartment-finder-app",
    "internal_url": "http://apartment-finder-app:80",
    "health_check_path": "/health",
    "metadata": {"version": "2.3.0"}
  }'
```

Ответ `200`:

```json
{"message":"Экземпляр сервиса успешно зарегистрирован",
 "instance":{"id":"...","service_key":"finder","container_name":"apartment-finder-app",
   "internal_url":"http://apartment-finder-app:80","external_prefix":"/finder",
   "health_check_path":"/health","status":"active",
   "registered_at":"2026-08-26T10:00:00+05:00","last_heartbeat":"2026-08-26T10:00:00+05:00"}}
```

Поведение реестра:

* Повторная регистрация с тем же `container_name` **или** тем же `internal_url` обновляет запись,
  дубля не создаёт.
* Без heartbeat 2 минуты — статус `unhealthy`; 10 минут — запись удаляется
  ([service_registry.go:141](../../!gateway/auth-service/routes/service_registry.go#L141)).
* Heartbeat на несуществующий экземпляр отвечает `404`, и клиент авторегистрируется заново.
* **Дерегистрация nginx-конфиг не трогает намеренно**: при рестарте контейнера пользователь
  получит временный 502, а не постоянный 404. `auth-connector` по SIGTERM/SIGINT тоже только
  гасит heartbeat и не дерегистрируется.

### 5.2. Синхронизация прав

| Метод | Путь | Назначение |
|---|---|---|
| `POST` | `/api/services/{key}/permissions/sync` | запустить синхронизацию |
| `GET` | `/api/sync/permissions` | собственные права auth-service |

**Ключевая деталь: это триггер, а не загрузка.** Тело запроса игнорируется — auth-service в ответ
сам идёт `GET {internal_url}/api/sync/permissions` и берёт список оттуда
([service_management.go:1147](../../!gateway/auth-service/routes/service_management.go#L1147)).
`AuthClient.sync_permissions()` шлёт тело только ради совместимости.

Адрес сервиса берётся: реестр → жёсткая таблица (`referal`, `client-service`) →
`http://{service_key}:80`. Если контейнер называется не как ключ и в реестре пусто —
синхронизация уйдёт «в никуда».

Та же кнопка есть в UI: `POST /services/{key}/permissions/sync` под cookie-авторизацией.

### 5.3. Пользователи сервиса

| Метод | Путь | Возвращает |
|---|---|---|
| `GET` | `/api/services/{key}/users` | все пользователи с ролями в сервисе |
| `GET` | `/api/services/{key}/users-by-role/{roleName}` | пользователи с конкретной ролью |
| `GET` | `/api/services/{key}/users-by-permission/{permName}` | пользователи с конкретным правом |
| `GET` | `/api/users/{userId}/profile` | профиль пользователя |
| `GET` | `/api/auth/roles/{roleName}` | описание роли auth-сервиса |
| `GET` | `/api/auth/roles/{roleName}/users` | носители роли auth-сервиса |

`/api/services/{key}/users` — массив (не объект), только базовые поля, без документов:

```json
[{"id":"665f...","username":"ivanov","email":"i@gh.uz","phone":"+998...",
  "last_name":"Иванов","first_name":"Иван","middle_name":"Петрович",
  "full_name":"Иванов Иван Петрович","avatar_path":"/avatar/665f...",
  "roles":["manager"]}]
```

`users-by-role` и `users-by-permission` дают более узкий набор — `user_id`, `username`, `email`,
`full_name`, `short_name`. Пустой результат — `[]`, не `null`.

`/api/users/{userId}/profile`:

```json
{"user_id":"665f...","username":"ivanov","email":"i@gh.uz",
 "full_name":"Иванов Иван Петрович","first_name":"Иван","last_name":"Иванов",
 "middle_name":"Петрович","suffix":"","phone":"+998901234567",
 "avatar_path":"/avatar/665f...","passport_number":"AA1234567",
 "passport_issued_by":"ИИБ","passport_issued_date":"2020-01-01T00:00:00Z",
 "address":"...","birth_date":"1990-01-01T00:00:00Z"}
```

Типовое применение: рассылка по владельцам права.

```python
users = requests.get(
    f"{AUTH_SERVICE_URL}/api/services/finder/users-by-permission/finder.reports_plan_fact_view",
    headers={"X-API-Key": INTERNAL_API_KEY}, timeout=10,
).json()
```

### 5.4. Документы пользователя

Единственный способ получить паспорт, ПИНФЛ и банковские реквизиты — заголовками они не приходят
(см. 3.2).

| Метод | Путь | Назначение |
|---|---|---|
| `GET` | `/api/users/{userId}/documents` | все документы |
| `GET` | `/api/users/{userId}/documents/grouped` | сгруппированные по `document_group` |
| `GET` | `/api/users/{userId}/documents/for-service/{key}` | **рекомендуется** — по одному документу из группы, отфильтровано по сервису |
| `POST` | `/api/users/{userId}/documents` | создать документ |
| `GET` | `/api/users/{userId}/documents/{docId}/attachments/{attachmentId}/download` | скачать вложение |

Документ:

```json
{"id":"...","document_type":"passport","title":"Паспорт РУз",
 "fields":{"passport_number":"AA1234567","passport_giver":"ИИБ","passport_date":"2020-01-01"},
 "attachments":[{"id":"...","file_name":"...","original_name":"скан.pdf","content_type":"application/pdf","size":123456}],
 "allowed_services":["referal","finder"],
 "status":"completed","created_at":"...","updated_at":"..."}
```

`for-service` отдаёт только документы, у которых ваш `service_key` есть в `allowed_services`, и
берёт по одному на группу. Для группы `identity` приоритет: `passport` (РУз) > `passport_ru` >
`pinfl`; при равном приоритете — свежий по `created_at`. Для остальных групп — просто свежий
([routes.go getUserDocumentsForServiceAPIHandler](../../!gateway/auth-service/routes/routes.go)).

Ответ:

```json
{"user_id":"665f...","service_key":"finder",
 "documents_for_service":{"identity":{"group":"identity","document":{...}},
                          "bank":{"group":"bank","document":{...}}}}
```

Пользователь без документов даёт `200` с пустым объектом, не `404`.

Создание:

```bash
curl -X POST http://auth-service:80/api/users/665f.../documents \
  -H "X-API-Key: $INTERNAL_API_KEY" -H "Content-Type: application/json" \
  -d '{"document_type":"bank_details","title":"Реквизиты",
       "fields":{"bank_name":"Ипотека-банк","card_number":"8600...","mfo":"00123"},
       "allowed_services":["finder"],"status":"draft"}'
```

Обязательны `document_type`, `title`, `fields`; `status` по умолчанию `draft`. Тип документа
должен существовать в `document_types`, иначе `400 Неверный тип документа`.
Список типов: `GET /document-types` (cookie-авторизация).

### 5.5. Telegram

| Метод | Путь | Назначение |
|---|---|---|
| `GET` | `/api/telegram/chat-id?username={telegram_username}` | telegram username → chat_id |
| `POST` | `/api/telegram/link/confirm` | служебный, вызывает notification-bot |
| `POST` | `/api/telegram/login/decision` | служебный, вызывает notification-bot |

```json
{"success": true, "chat_id": 123456789}
```

При отсутствии привязки — `200` с `{"success": false, "message": "..."}`, а не 404. Проверяйте
`success`, не только код ответа.

### 5.6. Резолв получателей уведомлений

`POST /api/recipients/resolve` — логины портала → адреса доставки. Вызывает
`notification-service`, когда уведомление адресовано полем `login`. Прикладным сервисам
дёргать напрямую не нужно: шлите `login` в notification-service, резолв он сделает сам.

```json
{"service": "finder", "channel": "telegram", "logins": ["ivanov", "petrov"]}
```
```json
{"success": true, "channel": "telegram", "access_enforced": false,
 "results": {
   "ivanov": {"found": true, "address": "123456789", "has_service_access": true},
   "petrov": {"found": false, "reason": "channel_not_linked", "has_service_access": true}}}
```

* `channel` — `telegram` (chat_id), `email`, `sms` (телефон).
* `service` — **`service_key` в auth-service**, а не имя ключа в `SERVICE_API_KEYS`.
  Имена расходятся: у apartment_finder ключ уведомлений `apartment-finder`, а `service_key` —
  `finder`; маппинг задаётся в notification-service переменной `SERVICE_AUTH_KEY_MAP`.
* Матч строго по `username`. Email и telegram-ник как идентификатор не принимаются:
  email не уникален как ключ адресации, telegram-ник меняется в один клик.
* `logins` — до 500 за запрос: пачка резолвится одним вызовом, а не по одному на уведомление.
* `reason`: `user_not_found`, `user_banned`, `no_service_access`, `channel_not_linked`,
  `no_address`, `unknown_channel`.
* `has_service_access` — есть ли у получателя роли в сервисе-отправителе. При
  `RECIPIENT_ACCESS_ENFORCE=false` (умолчание) отсутствие ролей только пишется в лог
  строкой `RECIPIENT ACCESS`; при `true` такой получатель отдаётся как
  `found: false, reason: no_service_access`. Включать флаг только после того, как логи
  покажут, что легитимных рассылок это не режет.

Эндпоинт раздаёт контактные данные — живёт только внутри `service_network` под `X-API-Key`,
наружу через nginx не публикуется.

### 5.7. Здоровье сервисов

`GET /api/services/health` — **cookie-авторизация** (`authRequired`), не `X-API-Key`. Для панели,
не для сервис-к-сервису.

```json
{"services":[{"service_key":"finder","service_name":"Подбор квартир","external_prefix":"/finder",
  "status":"healthy","last_heartbeat":"2026-08-26T10:00:00+05:00",
  "has_active_instance":true,"health_check_url":"/health"}],"count":1}
```

`healthy` — heartbeat моложе минуты, `unhealthy` — до двух минут, `offline` — дальше.

---

## 6. notification-service: email и Telegram

База: `http://notification-service:80`. Ключ — заголовок `X-API-Key`, но **свой**, из
`SERVICE_API_KEYS` вида `имя:ключ,имя:ключ` (легаси-фоллбек — общий `INTERNAL_API_KEY`).
Персональный ключ нужен: по нему считается rate-limit и видно в логах, кто шлёт.

| Метод | Путь | Назначение |
|---|---|---|
| `GET` | `/api/v1/health` | без аутентификации |
| `POST` | `/api/v1/notifications` | одно уведомление |
| `POST` | `/api/v1/notifications/batch` | пачка |
| `GET` | `/api/v1/notifications/{id}` | статус уведомления |
| `GET` | `/api/v1/batches/{batch_id}` | статус пачки |
| `GET` | `/api/v1/batches/{batch_id}/notifications` | уведомления пачки |
| `GET`/`POST` | `/api/v1/config` | конфигурация SMTP/Telegram |

Тело одиночного — получатель адресуется **логином портала**:

```json
{"type":"email","login":"ivanov","subject":"Отчёт готов",
 "content":"<p>Готово</p>","content_type":"text/html",
 "attachment_filename":"report.xlsx","attachment_content":"<base64>"}
```

Получателю вне портала (клиент, внешний подрядчик, канал Telegram) — `external_recipient`:

```json
{"type":"email","external_recipient":"client@mail.ru","content":"..."}
```

| Поле | Обяз. | Допустимые значения |
|---|---|---|
| `type` | да | `email`, `sms`, `push`, `telegram`, `telegram_system` |
| `login` | см. ниже | логин пользователя портала; адрес доставки определит auth-service |
| `external_recipient` | см. ниже | адрес получателя, которого нет в системе: email, телефон, chat_id |
| `content` | да | текст; для Telegram — Markdown |
| `subject` | нет | у Telegram становится жирной первой строкой |
| `content_type` | нет | `text/plain` (умолчание) или `text/html` |
| `attachment_content` | нет | base64 |

**Ровно одно** из `login` / `external_recipient`. Два сразу или ни одного — `400`.
Сервис не угадывает по виду строки: email в поле `login` — тоже `400`, а не «наверное, внешний».
Исключение — `telegram_system`: там получателя можно не указывать, он берётся из конфига.

Куда уведомление ушло на самом деле, видно в ответе `GET /api/v1/notifications/{id}`:
`recipient` — фактический адрес, `recipient_login` — логин, если адресовали по нему.

Резолв логина идёт через auth-service (`POST /api/recipients/resolve`, раздел 2) на **приёме**
запроса, а не при отправке. Поэтому отказ приходит сразу, синхронно, с машинным кодом:

| Ответ | `failure_code` | Что делать вызывающему |
|---|---|---|
| `400` | `user_not_found` | логина нет на портале — проверить справочник получателей |
| `400` | `user_banned` | пользователь заблокирован — не слать |
| `400` | `channel_not_linked` | Telegram не привязан или бот заблокирован — fallback на email или ссылка на личный кабинет |
| `400` | `no_address` | у пользователя не заполнен email/телефон |
| `400` | `no_service_access` | у пользователя нет ролей в вашем сервисе |
| `503` | `auth_unavailable` | auth-service недоступен — **повторить позже**, уведомление не создано |

В пачке отказ по одному получателю не рушит остальные: такая запись создаётся сразу в статусе
`failed` с `failure_code`, а список проблемных логинов возвращается в ответе полем `unresolved`.

**Заблокированный бот отключает привязку.** Если Telegram отвечает «bot was blocked by the
user», «chat not found» или «user is deactivated», notification-service сбрасывает привязку
в auth-service (`POST /api/telegram/link/broken`) и чистит свой кэш. Следующее уведомление
этому получателю отвалится уже на приёме с `channel_not_linked` — не дойдя ни до бота, ни до
Telegram. Это не наказание пользователя, а защита очереди: до сброса каждое такое уведомление
тратило полный круг до `api.telegram.org` и задерживало всю пачку. Пользователь восстанавливает
привязку в личном кабинете, там же видно, что бот заблокирован.

Вызывающему сервису отсюда следует: получив `channel_not_linked`, **не повторять** отправку по
тому же каналу, а переключаться на email или показывать в интерфейсе «Telegram отключён».

Все сервисы переведены на этот контракт: `referal`, `client_service`, `auth-service`,
`monitoring-service` и `apartment_finder`. Старое поле `recipient` из API удалено —
запрос с ним будет отвергнут как «не указан получатель».

Очередь сама делает до 3 попыток. Кнопок и callback пока нет — что именно допилить, расписано в
[TELEGRAM_NOTIFICATIONS_INTEGRATION_GUIDE.md](TELEGRAM_NOTIFICATIONS_INTEGRATION_GUIDE.md), часть B.

**Токен Telegram-бота в конечном сервисе держать нельзя.** Второй `getUpdates` с тем же токеном
ломает long polling бота на весь портал.

Ханипоты `/.env`, `/api/v1/internal/keys`, `/api/v1/admin/exec`, `/api/v1/debug/pprof` отдают 404 и
поднимают тревогу `GUARD TRIPWIRE` — не ходите туда даже из любопытства.

---

## 7. Что шлюз даёт пользователю вокруг сервиса

Готовые страницы — переиспользуйте, не пишите свои.

| URL | Что | Кому |
|---|---|---|
| `/login`, `/logout` | вход и выход, включая вход через Telegram | всем |
| `/forgot-password`, `/reset-password` | восстановление пароля (email или Telegram) | всем |
| `/menu` | плитка доступных сервисов | авторизованным |
| `/profile` | профиль, аватар, документы | авторизованным |
| `/access-denied?service=...` | «доступ запрещён» | всем |
| `/services/{key}` | админка сервиса: права, роли, пользователи, импорт/экспорт | админ сервиса |
| `/services/{key}/logs` | логи контейнеров сервиса (Dozzle) | по правам |
| `/services/{key}/import`, `/export`, `/template` | Excel-обмен пользователями и ролями | админ сервиса |
| `/avatar/{...}` | аватары | авторизованным |
| `/_shared/js/...`, `/_shared/css/...` | общие вендорные библиотеки на весь портал | всем |
| `/settings` | системные настройки | `auth.settings.view` |

Из шаблонов сервиса ведите логотип на `/menu`, а «выход» — на `/logout`. Свою страницу логина
делать не нужно и нельзя: `auth_bp` в сервисе должен быть заглушкой с редиректами
(пример — [apartment_finder/app/web/auth_routes.py](../../apartment_finder/app/web/auth_routes.py)).

**Общая статика.** Портал в изолированной сети, CDN недоступен. Общие библиотеки вендорьте либо в
`/_shared/` шлюза, либо в `static/vendor/` своего сервиса. CSP шлюза разрешает `default-src 'self'`
плюс `cdn.jsdelivr.net`, `cdnjs.cloudflare.com`, `fonts.googleapis.com`/`gstatic.com` — но в
изолированной сети эти хосты всё равно не резолвятся.

**Логи.** Фильтр контейнеров по умолчанию — `name={service_key}*`
([auth_handlers.go:732](../../!gateway/auth-service/routes/auth_handlers.go#L732)). Если имя контейнера
не начинается с ключа (как `apartment-finder-app` при ключе `finder`), надёжнее добавить явную
запись в `containerPatterns`.

---

## 8. Что nginx генерирует за вас

При регистрации auth-service пишет `dynamic/service-{key}.conf` и шлёт nginx SIGHUP через
docker-socket-proxy. Руками файл не править — перезапишется. Шаблон —
[nginx_config.go:22](../../!gateway/auth-service/routes/nginx_config.go#L22).

Три location на сервис:

| location | auth_request | Поведение |
|---|---|---|
| `= /{key}/health` | **нет** | точное совпадение выигрывает у префикса — health публичен. Ничего чувствительного туда не кладите |
| `/{key}/static/` | **нет** | статика кэшируется на час; нужна и странице логина |
| `/{key}/static/uploads/` | — | жёсткий `404` |
| `/{key}/` | **да** | всё остальное: `/verify`, проброс `X-User-*`, `rewrite ^/{key}/(.*) /$1` |

Отсюда два следствия:

* **Загрузки пользователей нельзя класть в `static/`.** Блок статики идёт без `auth_request`, всё
  внутри качалось бы анонимно; `static/uploads/` поэтому и закрыт наглухо. Отдавайте файлы своим
  роутом `/{key}/media/...` с проверкой прав.
* **Префикс срезается до приложения.** Внутри сервиса пути начинаются с `/`. Обратно префикс в
  ссылки возвращает `ProxyFix(x_prefix=1)` + `PrefixMiddleware` через `X-Forwarded-Prefix`.

Таймауты по 60 с на connect/send/read. Долгие отчёты либо укладывайте в минуту, либо делайте
асинхронными: 504 придёт от nginx, и ответ сервиса уже никто не увидит.

`client_max_body_size 100M` задан на весь сервер.

---

## 9. Грабли

| Симптом | Причина | Лечение |
|---|---|---|
| 500 при регистрации: `service not found in services collection` | сервис не заведён в админке | шаг 1 раздела 2 |
| `/menu` не показывает карточку | у пользователя нет ни одной активной роли в сервисе | назначить роль |
| Роут даёт 403 при выданном `finder.*` | проверка через `in`, а не `permission_granted()` | раздел 4.1 |
| Все страницы 401 | нет `AuthMiddleware`, либо запрос идёт мимо nginx | раздел 3 |
| Права в админке не появились | auth-service не смог достучаться до `/api/sync/permissions` | проверить реестр и имя контейнера |
| Ссылки без префикса `/finder` | нет `ProxyFix(x_prefix=1)` / `PrefixMiddleware` | раздел 8 |
| Аватар не грузится | путь из `X-User-Avatar` дописан после префикса сервиса | путь абсолютный от корня домена |
| ФИО в кракозябрах | не декодирован base64 из `X-User-Full-Name` | `base64.b64decode(v).decode('utf-8')` |
| Паспорт/ПИНФЛ пустые | ждали заголовков | раздел 5.4 |
| 404 на всём сервисе после рестарта | кто-то дерегистрировал экземпляр и перегенерировал конфиг | перерегистрировать; штатный рестарт даёт 502, а не 404 |
| 502 при живом контейнере | `internal_url` не резолвится, контейнер не в `service_network` | проверить сеть и имя |
| 504 на тяжёлом отчёте | лимит nginx 60 с | асинхронная выгрузка |
| Файл из `static/uploads/` даёт 404 | так и задумано | отдавать своим роутом с проверкой прав |
| Логи сервиса пустые в `/services/{key}/logs` | фильтр `name={key}*` не совпал с именем контейнера | раздел 7 |
| Уведомление даёт 400 `user_not_found` на живом сотруднике | в `login` ушёл email или telegram-ник вместо логина портала | раздел 6 |
| Уведомления сервиса разом дают `no_service_access` | `SERVICE_AUTH_KEY_MAP` не сопоставил имя ключа с `service_key` | раздел 5.6 |
| 503 `auth_unavailable` на отправке | auth-service недоступен, уведомление НЕ создано | повторить; проверить auth-service |
| В логе `Email sent`, но письмо не пришло | включён debug-режим — письма уходят на `debug_email` | `GET /api/v1/config`, поле `debug_mode` |
| По логу не понять, чей адрес: `d.***@gh.uz` | адреса в логах маскируются | на время разбора `LOG_FULL_RECIPIENTS=true` у notification-service |

---

## 10. Чеклист подключения

```
[ ] Сервис заведён в /services/new: key == префикс URL
[ ] Контейнер слушает :80 и подключён к service_network
[ ] GET /health отвечает 200 и не содержит чувствительных данных
[ ] GET /api/sync/permissions отдаёт {"success": true, "permissions": [...]}
[ ] permissions_setup.py объявляет права, включая {key}.*
[ ] Проверки прав идут через permission_granted() из auth_connector
[ ] ProxyFix(x_prefix=1) + PrefixMiddleware настроены
[ ] Свои страницы логина/логаута заменены редиректами на /login, /logout
[ ] Пользовательские загрузки лежат вне static/
[ ] Ассеты вендорены (CDN в изолированной сети недоступен)
[ ] INTERNAL_API_KEY задан; персональный ключ notification-service получен
[ ] Токена Telegram-бота в сервисе нет
[ ] Экземпляр виден в GET /api/registry/services
[ ] Права видны на /services/{key}
[ ] Роли созданы и назначены
```
