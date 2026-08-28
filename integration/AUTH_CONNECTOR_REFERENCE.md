# auth-connector: справочник по пакету

Что лежит внутри пакета, какие классы и функции он экспортирует и что каждая
делает. Здесь **API-справочник**; как пакет подключать, настраивать и
запускать локально — в
[AUTH_CONNECTOR_SETUP_AND_LOCAL_DEV.md](AUTH_CONNECTOR_SETUP_AND_LOCAL_DEV.md).
Контракт шлюза целиком — в
[GATEWAY_SERVICE_INTEGRATION_API.md](GATEWAY_SERVICE_INTEGRATION_API.md).

Исходники: [auth-connector/](../../auth-connector/) (сабмодуль
`git@github.com-work:ghuz-code-repo/auth-connector.git`).

## Что это

Python-пакет, который избавляет конечный сервис от трёх обязанностей:
разбирать заголовки шлюза, сравнивать права с учётом wildcard и
регистрироваться в реестре сервисов. Своей аутентификации он не делает —
личность уже проверена шлюзом до того, как запрос дошёл до сервиса.

```
nginx ──X-User-*──► ваш Flask
                       │
                       ├ AuthMiddleware.before_request  → g.user: UserContext
                       ├ @require_permission('finder.x') → 401 / 403 / роут
                       ├ AuthClient                      → auth-service /api/*
                       └ ServiceDiscoveryClient          → register + heartbeat
```

- **Версия:** `2.0.0` (`auth_connector/__init__.py`, `setup.py` — держать
  синхронно: по этой строке сервисы проверяют поколение пакета)
- **Python:** ≥ 3.7
- **Зависимости:** `requests>=2.28`, `PyJWT>=2.4`, `Flask>=2.0`
- **Модули:** `auth_middleware`, `auth_client`, `permissions`,
  `permission_utils`, `service_discovery`, `exceptions`

## Ломающие изменения 2.0.0

| Что было | Что стало | Почему |
|---|---|---|
| `UserContext.is_admin` | поля нет | шлюз больше не шлёт `X-User-Admin`; клиент мог прислать его сам |
| `@require_permission(..., allow_admin=True)` | параметра нет | ветка «если админ — пустить» срабатывала до сравнения прав и скрывала, что реально выдано роли |
| проверка прав в каждом сервисе | модуль `permission_utils` | реализации расходились, шаблон и роут отвечали по-разному |

Администратор теперь носит обычное право `<service>.*` и проходит те же
проверки, что все остальные.

---

## `auth_middleware`

### `class UserContext`

Личность, разобранная из заголовков. Кладётся в `flask.g.user`.

| Атрибут | Тип | Источник |
|---|---|---|
| `user_id` | `str` | `X-User-Id` |
| `username` | `str` | `X-User-Name` |
| `full_name` | `str` | `X-User-Full-Name`, декодируется из base64 |
| `roles` | `List[str]` | `X-User-Service-Roles`, split по запятой |
| `permissions` | `List[str]` | `X-User-Service-Permissions`, split по запятой |
| `raw_headers` | `Dict[str, str]` | все заголовки запроса |

| Метод | Что делает |
|---|---|
| `has_permission(perm)` | проверка с учётом wildcard (делегирует `permission_granted`) |
| `has_any_permission([...])` | хватает любого из списка |
| `has_all_permissions([...])` | нужны все |
| `has_role(role)` | точное совпадение по имени роли, **без** wildcard |
| `to_dict()` | словарь для сериализации |

`is_admin` нет намеренно — см. таблицу ломающих изменений.

### `class AuthMiddleware`

```python
AuthMiddleware(app=None, auth_client=None, jwt_secret=None, verify_signature=True)
```

Вешает `before_request`, который на каждый запрос собирает `UserContext` и
кладёт его в `g.user`, а `auth_client` — в `g.auth_client`. Ошибка разбора не
роняет запрос: `g.user` становится `None`, а декораторы вернут `401`.

Три источника личности, проверяются по порядку:

1. `Authorization: Bearer <jwt>` — требует `jwt_secret`, иначе
   `InvalidTokenError`. Алгоритм `HS256`.
2. **Заголовки шлюза** — основной путь: есть `X-User-Id` и `X-User-Name`.
3. `X-Internal-Auth` — base64 от JSON, для служебных межсервисных вызовов.

Если ни один не подошёл — `g.user is None`.

### Декораторы

```python
@require_permission('finder.selection_view')
@require_any_permission(['finder.report_view', 'finder.*'])
@require_role('manager')           # для обратной совместимости
```

Поведение одинаковое: нет `g.user` → `401` с
`{"error": ..., "code": "AUTH_REQUIRED"}`; нет права → `403` с
`{"code": "PERMISSION_DENIED", "required_permission": ...}`.

Ответы **всегда JSON**. Для HTML-страниц нужен свой декоратор поверх —
см. раздел 5.2 [AUTH_CONNECTOR_SETUP_AND_LOCAL_DEV.md](AUTH_CONNECTOR_SETUP_AND_LOCAL_DEV.md).

### `get_current_user() -> Optional[UserContext]`

`getattr(g, 'user', None)`. Единственный правильный способ добраться до
личности из кода роута и из контекст-процессора шаблонов.

---

## `permission_utils`

Одна реализация сравнения прав на весь флот. **Дублировать её в сервисе
нельзя** — расхождения дают разный ответ в шаблоне и на роуте.

### `permission_granted(permission_name, permissions) -> bool`

Совпадением считается:

| Запись в списке | Покрывает |
|---|---|
| `finder.selection_view` | точно это право |
| `*` | всё |
| `finder.*` | всё с префиксом `finder.` |

```python
permission_granted('finder.projects_info_update', ['finder.*'])  # True
'finder.projects_info_update' in ['finder.*']                    # False  ← так ломается
```

Шлюз отдаёт право **как оно выдано роли**: `finder.*` приходит строкой со
звёздочкой, в список не разворачивается. Сервисы, сравнивавшие точным `in`,
такое право не видели — пункт меню пропадал, роут отвечал `403` при формально
выданном доступе. Это самая частая ошибка интеграции.

### `any_permission_granted(permission_names, permissions) -> bool`

Хватает любого права из списка; список прав разворачивается один раз.

### `extract_permissions(user) -> List[str]`

Достаёт права из пользователя любой формы — `UserContext`, словарь из
`to_dict()`, `None`. Нужен там, где неизвестно, что именно лежит в
`g.user`: в контекст-процессорах и служебных хелперах.

---

## `auth_client`

### `class AuthClient`

```python
AuthClient(auth_service_url, service_key, timeout=10, api_key=None)
```

`api_key` по умолчанию берётся из `INTERNAL_API_KEY`. Клиент держит
`requests.Session` с заголовком `X-API-Key` на всех запросах.

| Метод | Эндпоинт auth-service | Возврат |
|---|---|---|
| `api_headers` (property) | — | `{'X-API-Key': ...}` для собственных `requests.get()` |
| `get_user_document(user_id, document_type=None)` | `GET /api/users/{id}/documents` | документ или `None` (404 → `None`) |
| `sync_permissions(permissions)` | `POST /api/services/{key}/permissions/sync` | `bool` |
| `validate_token(token)` | `POST /api/validate-token` | dict; `401` → `InvalidTokenError` |
| `health_check()` | `GET /health` | `bool` |
| `clear_cache()` | — | сброс внутреннего кэша |

Сетевые ошибки не пробрасываются наружу у `get_user_document` и
`sync_permissions` — пишется лог, возвращается `None` / `False`.
`validate_token` при недоступности поднимает `AuthServiceUnavailableError`.

**Устарело:**

- `get_user_permissions()` — дёргает
  `/api/users/{id}/permissions/{service_key}`, которого **в auth-service
  нет**. Метод оставлен ради совместимости сигнатуры, пишет
  `DeprecationWarning`. Права берутся из заголовка
  `X-User-Service-Permissions`, а не запросом.
- `validate_token()` — `POST /api/validate-token` в текущем auth-service
  тоже отсутствует. Токен проверяет шлюз через `auth_request`, сервису
  проверять нечего.

---

## `permissions`

### `@dataclass Permission`

`name`, `display_name`, `description`, `category`.

### `class PermissionRegistry`

Каталог прав сервиса. Из него собирается ответ `GET /api/sync/permissions`.

```python
registry = PermissionRegistry('finder')
registry.register('finder.*', 'Все права сервиса', 'Полный доступ', category='manage')
registry.register('finder.selection_view', 'Подбор: просмотр', 'Доступ к подбору', category='view')

# в роуте:
return jsonify({'success': True, **registry.to_dict()})
```

| Метод | Что делает |
|---|---|
| `register(name, display_name, description, category=None)` | добавляет право; пустое `name` → `ConfigurationError` |
| `get_permission(name)` | одно право или `None` |
| `get_all_permissions()` | список `Permission` |
| `get_permissions_by_category(category)` | права одной категории |
| `to_dict()` | `{"service_key", "permissions": [{name, displayName, description, category}], "categories"}` |
| `to_json()` | то же строкой, `ensure_ascii=False` |

**Wildcard `<key>.*` объявлять обязательно** — им живёт роль `admin`,
которую auth-service заводит сам при старте.

Парсер auth-service читает только `name`, `displayName`, `description`;
`category` при пуле теряется — категории видны в вашем ответе, но в
`PermissionDef` не сохраняются.

### `class CommonPermissions`

Генераторы шаблонных наборов, возвращают список кортежей
`(name, display_name, description)`:

- `crud_permissions(resource)` → `.view`, `.create`, `.edit`, `.delete`
- `admin_permissions(service)` → `.admin.manage_users`, `.admin.view_logs`,
  `.admin.export_data`, `.admin.manage_settings`

---

## `service_discovery`

### `class ServiceDiscoveryClient`

```python
ServiceDiscoveryClient(
    service_key, internal_url,
    registry_url='http://auth-service:8080/api/registry',
    container_name=None,          # по умолчанию CONTAINER_NAME, иначе hostname
    health_check_path='/health',
    heartbeat_interval=30,
    metadata=None,
    api_key=None,                 # по умолчанию INTERNAL_API_KEY
)
```

> Значение `registry_url` по умолчанию указывает на порт **8080**, а
> auth-service слушает **:80**. Передавайте `registry_url` явно:
> `http://auth-service:80/api/registry`.

| Метод | Что делает |
|---|---|
| `register(max_retries=10, retry_delay=3)` | `POST /register`; повторяет при недоступности auth-service |
| `send_heartbeat()` | `POST /heartbeat`; при `404` пытается перерегистрироваться |
| `start_heartbeat()` / `stop_heartbeat()` | фоновый поток-демон с периодом `heartbeat_interval` |
| `deregister()` | `DELETE /unregister/{key}` — **вызывать вручную почти никогда не нужно** |

**Дерегистрации при остановке нет намеренно.** `atexit` и обработчики
`SIGTERM`/`SIGINT` только останавливают heartbeat. Дерегистрация заставила бы
auth-service перегенерировать конфиг nginx, и на время рестарта контейнера
все пользователи получали бы `404` вместо `502`. Протухший heartbeat сам
пометит экземпляр как `unhealthy`.

### `init_service_discovery_flask(app, service_key, internal_url, **kwargs)`

Создаёт клиент и в фоновом потоке-демоне (после паузы 2 с, чтобы Flask успел
подняться) регистрируется и запускает heartbeat. Возвращает клиент.

### `init_service_discovery_fastapi(app, service_key, internal_url, **kwargs)`

То же через `@app.on_event("startup")`. **Внимание:** в отличие от
Flask-варианта, здесь на `shutdown` вызывается `deregister()` — с описанным
выше последствием (`404` вместо `502` во время рестарта). Для сервиса, чей
маршрут генерируется шлюзом, обработчик стоит убрать или использовать
`ServiceDiscoveryClient` напрямую.

---

## `exceptions`

```
AuthError
├── PermissionDeniedError(permission, user_id=None)
├── InvalidTokenError
├── AuthServiceUnavailableError
└── ConfigurationError
```

Декораторы `require_*` эти исключения **не** поднимают — они возвращают
JSON-ответы `401`/`403`. Исключения приходят из `AuthClient`,
`AuthMiddleware._extract_from_jwt` и `PermissionRegistry.register`.

---

## Минимальное подключение

```python
from flask import Flask
from auth_connector import AuthMiddleware, AuthClient, init_service_discovery_flask

app = Flask(__name__)

auth_client = AuthClient(
    auth_service_url='http://auth-service:80',
    service_key='finder',
)
AuthMiddleware(app, auth_client=auth_client)

init_service_discovery_flask(
    app, 'finder', 'http://finder:80',
    registry_url='http://auth-service:80/api/registry',
    health_check_path='/health',
)
```

Порядок обёрток (`ProxyFix`, `PrefixMiddleware`), обязательные роуты
`/health` и `/api/sync/permissions`, переменные окружения и локальный запуск —
раздел 3 и далее в
[AUTH_CONNECTOR_SETUP_AND_LOCAL_DEV.md](AUTH_CONNECTOR_SETUP_AND_LOCAL_DEV.md).

---

## Тестируемость

Пакет разбирает обычный `dict` заголовков, поэтому личность в тестах
подделывается без шлюза и без сети:

```python
def test_permission_wildcard():
    from auth_connector import permission_granted
    assert permission_granted('finder.selection_view', ['finder.*'])
    assert not permission_granted('client.view', ['finder.*'])


def test_route_requires_permission(client):
    # заголовки шлюза подделываются напрямую
    r = client.get('/selection', headers={
        'X-User-Id': '507f1f77bcf86cd799439011',
        'X-User-Name': 'ivanov',
        'X-User-Service-Permissions': 'finder.selection_view',
    })
    assert r.status_code == 200

    r = client.get('/selection', headers={
        'X-User-Id': '507f1f77bcf86cd799439011',
        'X-User-Name': 'ivanov',
        'X-User-Service-Permissions': 'finder.other',
    })
    assert r.status_code == 403
```

Что обязательно покрывать в своём сервисе:

- каждый роут — доступ разрешён / запрещён / нет личности (`401`);
- wildcard `<key>.*` открывает роут (регресс на самую частую ошибку);
- `GET /health` → `200`;
- `GET /api/sync/permissions` → `success: true` и наличие `<key>.*`;
- работа под префиксом: `url_for()` даёт `/<key>/...`.

Правила по тестам для всей платформы — раздел «Тесты и локальная
проверяемость» в [../AGENTS.md](../AGENTS.md).

## Обновление пакета

`auth-connector` подключается сабмодулем и ставится `pip install`. Версия
меняется не каждый релиз, поэтому pip может счесть уже установленную
достаточной — в `Dockerfile` ставить с `--force-reinstall` отдельным слоем
после тяжёлых зависимостей. Подробности и готовый `Dockerfile` — раздел 2.3
[AUTH_CONNECTOR_SETUP_AND_LOCAL_DEV.md](AUTH_CONNECTOR_SETUP_AND_LOCAL_DEV.md).
