# auth-connector: подключение и локальная разработка

Как правильно подключить `auth-connector` к Flask-сервису и как настроить его так, чтобы сервис
можно было запускать и тестировать локально — в своём контейнере или вообще без контейнеров, без
шлюза, без auth-service, без MongoDB.

Парные документы:

* [GATEWAY_SERVICE_INTEGRATION_API.md](GATEWAY_SERVICE_INTEGRATION_API.md) — что шлюз даёт
  сервису и какие у него API.
* [AUTH_CONNECTOR_REFERENCE.md](AUTH_CONNECTOR_REFERENCE.md) — справочник по API пакета:
  классы, декораторы, сигнатуры, что устарело.

---

## 1. Что внутри пакета

Пакет — сабмодуль репозитория, [auth-connector/](../../auth-connector/), версия **2.0.0**.

| Модуль | Что даёт |
|---|---|
| `auth_middleware` | `AuthMiddleware`, `UserContext`, декораторы `require_permission`, `require_any_permission`, `require_role`, `get_current_user` |
| `permission_utils` | `permission_granted`, `any_permission_granted`, `extract_permissions` — единственная правильная проверка прав |
| `permissions` | `PermissionRegistry` — каталог прав сервиса |
| `auth_client` | `AuthClient` — вызовы `/api/*` auth-service с `X-API-Key` |
| `service_discovery` | `ServiceDiscoveryClient`, `init_service_discovery_flask`, `init_service_discovery_fastapi` |
| `exceptions` | `AuthError`, `PermissionDeniedError`, `InvalidTokenError`, `AuthServiceUnavailableError`, `ConfigurationError` |

Зависимости: `requests>=2.28`, `PyJWT>=2.4`, `Flask>=2.0`. Python ≥ 3.7.

### 1.1. Ломающие изменения 2.0.0

* `UserContext` **лишился** `is_admin`.
* Декораторы **потеряли** параметр `allow_admin`.
* Появился модуль `permission_utils`.

Признака администратора у сервиса больше нет вообще. Раньше сервисы делали ранний выход по
`X-User-Admin` **до** сравнения прав — из-за этого администратор проходил куда угодно, а выданное
роли в админке расходилось с тем, что реально действовало. Теперь администратор носит обычное
право `{service_key}.*`, которое проходит штатной проверкой шаблона.

---

## 2. Установка

### 2.1. Сабмодуль

```bash
git submodule update --init auth-connector
```

Обновлять — **по ветке**, не `git submodule update`: последний откатывает сабмодуль на записанный
в родительском репозитории коммит и роняет прод.

```bash
git -C auth-connector checkout master && git -C auth-connector pull
```

### 2.2. Локально

```bash
pip install -e ./auth-connector
```

Флаг `-e` важен: правки пакета сразу видны сервису без переустановки.

### 2.3. В Dockerfile

Порядок слоёв не косметика. `auth-connector` меняется часто, `requirements.txt` — почти никогда.
Обратный порядок сбрасывает кэш на каждой правке коннектора и тянет переустановку всех тяжёлых
пакетов (scipy, numpy, pandas, scikit-learn, matplotlib, lxml).

```dockerfile
# syntax=docker/dockerfile:1
FROM python:3.11-slim
WORKDIR /app

# 1. Тяжёлые зависимости — первым слоем
COPY apartment_finder/requirements.txt .
RUN --mount=type=cache,target=/root/.cache/pip pip install -r requirements.txt

# 2. auth-connector — отдельным слоем после
# --force-reinstall обязателен: версия пакета меняется не каждый раз,
# и pip иначе счёл бы уже установленную достаточной.
COPY auth-connector /tmp/auth-connector
RUN pip install --no-cache-dir --force-reinstall /tmp/auth-connector && rm -rf /tmp/auth-connector

# 3. Ломать сборку, а не рантайм
RUN python -c "import auth_connector as a, sys; v = a.__version__; \
    sys.exit(0 if int(v.split('.')[0]) >= 2 else 'auth-connector %s < 2.0.0 — обновите сабмодуль' % v)"

COPY apartment_finder/ .
EXPOSE 80
CMD ["gunicorn", "--bind", "0.0.0.0:80", "--worker-class", "gthread", \
     "--workers", "2", "--threads", "8", "--timeout", "120", "app_with_auth_connector:app"]
```

Проверка версии нужна, потому что со старым пакетом приложение импортируется, но падает воркером
при первом запросе — сервис отдаёт 502 целиком, а причина неочевидна.

Контекст сборки — **родительский каталог**, чтобы `COPY auth-connector` видел сабмодуль:

```yaml
build:
  context: ..
  dockerfile: apartment_finder/Dockerfile
```

---

## 3. Подключение к Flask

### 3.1. Точка входа

Файл `app_with_auth_connector.py` — то, что запускает gunicorn.

```python
"""Точка входа для gunicorn под шлюзом."""
import os
import threading
import time

from werkzeug.middleware.proxy_fix import ProxyFix

from app import create_app
from app.core.config import ProductionConfig
from prefix_middleware import PrefixMiddleware

try:
    from auth_connector import AuthMiddleware, AuthClient, init_service_discovery_flask
    from permissions_setup import permissions_registry
except ImportError:
    print("Warning: auth-connector не установлен. pip install -e ../auth-connector")
    AuthMiddleware = AuthClient = init_service_discovery_flask = None
    permissions_registry = None

SERVICE_KEY = "finder"
INTERNAL_URL = "http://apartment-finder-app:80"

# Служебные команды (`flask <cmd>`) импортируют этот же модуль.
# Флаг ставит сам flask-cli до импорта приложения.
RUNNING_FROM_CLI = os.environ.get('FLASK_RUN_FROM_CLI') == 'true'

app = create_app(ProductionConfig)

# Порядок обёрток важен: ProxyFix читает X-Forwarded-*, PrefixMiddleware
# ставит SCRIPT_NAME, чтобы url_for() выдавал ссылки с префиксом.
app.wsgi_app = ProxyFix(app.wsgi_app, x_for=1, x_proto=1, x_host=1, x_prefix=1)
app.wsgi_app = PrefixMiddleware(app.wsgi_app, app=app, prefix=f'/{SERVICE_KEY}')

if AuthMiddleware:
    AuthMiddleware(app, jwt_secret=os.getenv('JWT_SECRET'))


def start_background_jobs():
    """Регистрация в реестре и синхронизация прав.

    Отдельно от импорта модуля: служебной команде всё это не нужно, иначе она
    регистрирует в реестре второй экземпляр и завершается, оставляя там запись
    от уже умершего процесса.
    """
    auth_url = os.getenv('AUTH_SERVICE_URL', 'http://auth-service:80')
    api_key = os.getenv('INTERNAL_API_KEY', '')

    if AuthClient and permissions_registry and api_key:
        def _sync_delayed():
            # Дать gunicorn начать обслуживать запросы: auth-service в ответ
            # на POST сам пойдёт к нам за списком прав.
            time.sleep(5)
            client = AuthClient(auth_url, service_key=SERVICE_KEY, api_key=api_key)
            with app.app_context():
                try:
                    client.sync_permissions(permissions_registry.to_dict()['permissions'])
                except Exception as e:
                    print(f"[AUTH] Синхронизация прав не удалась: {e}")

        threading.Thread(target=_sync_delayed, daemon=True).start()

    if init_service_discovery_flask:
        try:
            init_service_discovery_flask(
                app,
                service_key=SERVICE_KEY,
                internal_url=INTERNAL_URL,
                registry_url=auth_url + '/api/registry',
                heartbeat_interval=30,
            )
        except Exception as e:
            print(f"[AUTH] Service discovery не запущен: {e}")


if not RUNNING_FROM_CLI:
    start_background_jobs()
```

Что здесь принципиально:

* `AuthMiddleware` вешает `before_request`, который кладёт в `g.user` объект `UserContext` или
  `None`. Без него **все** защищённые роуты отвечают 401.
* `jwt_secret` нужен только для разбора `Authorization: Bearer`. Основной путь под шлюзом —
  заголовки, он работает и без секрета.
* Регистрация в реестре живёт в фоновом потоке и не задерживает старт.

### 3.2. PrefixMiddleware

```python
class PrefixMiddleware:
    """Ставит SCRIPT_NAME, чтобы url_for() выдавал ссылки с префиксом сервиса."""

    def __init__(self, wsgi_app, app=None, prefix='/finder'):
        self.wsgi_app = wsgi_app
        self.prefix = prefix.rstrip('/')
        if app is not None:
            app.config['APPLICATION_ROOT'] = self.prefix
            app.static_url_path = self.prefix + '/static'

    def __call__(self, environ, start_response):
        script_name = environ.get('SCRIPT_NAME', '')
        path_info = environ.get('PATH_INFO', '')
        forwarded_prefix = environ.get('HTTP_X_FORWARDED_PREFIX', '').rstrip('/')

        # Статику nginx отдаёт отдельным блоком, префикс там уже срезан.
        if path_info.startswith('/static'):
            return self.wsgi_app(environ, start_response)

        if forwarded_prefix:
            environ['SCRIPT_NAME'] = script_name + forwarded_prefix
        elif path_info.startswith(self.prefix):
            environ['SCRIPT_NAME'] = script_name + self.prefix
            environ['PATH_INFO'] = path_info[len(self.prefix):] or '/'

        return self.wsgi_app(environ, start_response)
```

Ветка `elif` — та, что позволяет открыть сервис локально по `http://localhost:5000/finder/...`
без nginx.

### 3.3. Обязательные роуты

```python
@app.route('/health')
def health_check():
    return {'status': 'ok', 'service': 'apartment-finder'}, 200
```

`/{key}/health` в nginx идёт **без** `auth_request` — доступен анонимно. Версии, имена хостов,
строки подключения туда не кладите.

Синхронизация прав — отдельный blueprint под `url_prefix='/api/sync'`:

```python
from flask import Blueprint, jsonify
from permissions_setup import permissions_registry

sync_bp = Blueprint('sync', __name__)


@sync_bp.route('/permissions', methods=['GET', 'POST'])
def get_service_permissions():
    """Список прав сервиса. Дёргает auth-service при синхронизации."""
    try:
        all_perms = permissions_registry.get_all_permissions()
        return jsonify({
            "success": True,
            "service_key": "finder",
            "permissions": permissions_registry.to_dict()["permissions"],
            "total_permissions": len(all_perms),
            "categories": sorted({p.category for p in all_perms if p.category}),
        })
    except Exception as e:
        return jsonify({"success": False, "error": str(e)}), 500
```

---

## 4. Каталог прав

`permissions_setup.py` в корне сервиса:

```python
"""Каталог прав сервиса для шлюза.

Имена: {service_key}.{модуль}_{сущность}_{действие},
действие из view | create | update | delete | export | import | calculate.
"""
from auth_connector import PermissionRegistry

permissions_registry = PermissionRegistry('finder')

# Обязательное wildcard-право: им живёт роль admin, которую auth-service
# создаёт сам при старте.
permissions_registry.register(
    'finder.*', 'Все права сервиса', 'Полный доступ ко всем функциям finder', 'manage')

permissions_registry.register(
    'finder.selection_view', 'Подбор: просмотр', 'Доступ к странице подбора', 'view')
permissions_registry.register(
    'finder.discounts_versions_update', 'Скидки: редактирование', 'Правка версий скидок', 'manage')

# Локальные короткие имена -> шлюзовые. Роуты и шаблоны оперируют левой
# колонкой, шлюз присылает правую.
PERMISSION_MAP = {
    'selection_view': 'finder.selection_view',
    'discounts_versions_update': 'finder.discounts_versions_update',
}
```

`category` группирует права в админке. При пуле в `PermissionDef` она теряется — поле есть в
ответе, но auth-service его не сохраняет.

---

## 5. Проверка прав в коде

### 5.1. Единственно верный способ

```python
from auth_connector.permission_utils import permission_granted, extract_permissions
```

Формы пользователя разные: `AuthMiddleware` кладёт в `g.user` объект `UserContext`, а `to_dict()`
и служебные вызовы — словарь. `extract_permissions()` работает с обеими.

Проверка `if 'finder.selection_view' in user.permissions` **неверна**: право, выданное роли как
`finder.*`, приезжает строкой `finder.*` без разворачивания в список. Точное сравнение его не
увидит — пункт меню пропадёт, роут ответит 403 при формально выданном доступе.

### 5.2. Свои декораторы поверх

Готовые `require_permission` из пакета возвращают JSON. Для страниц удобнее `abort()`:

```python
from functools import wraps
from flask import abort, g

from auth_connector.auth_middleware import UserContext
from auth_connector.permission_utils import (any_permission_granted as _any_granted,
                                             extract_permissions,
                                             permission_granted as _granted)
from permissions_setup import PERMISSION_MAP


def _current_user():
    user = getattr(g, 'user', None)
    return user if isinstance(user, (dict, UserContext)) else None


def permission_granted(name, permissions):
    """Локальное имя права -> шлюзовое, дальше общая проверка шаблонов."""
    return _granted(PERMISSION_MAP.get(name, name), permissions)


def login_required(f):
    @wraps(f)
    def wrapper(*args, **kwargs):
        if not _current_user():
            abort(401)
        return f(*args, **kwargs)
    return wrapper


def permission_required(name):
    def decorator(fn):
        @wraps(fn)
        def wrapper(*args, **kwargs):
            user = _current_user()
            if not user:
                abort(401)
            if not permission_granted(name, extract_permissions(user)):
                abort(403)
            return fn(*args, **kwargs)
        return wrapper
    return decorator


def permission_required_any(*names):
    """Для страниц, открытых и на просмотр, и на правку.

    Право на правку подразумевает просмотр, но обратное неверно; один декоратор
    с правом на правку заставлял бы выдавать доступ к записи только ради чтения.
    """
    def decorator(fn):
        @wraps(fn)
        def wrapper(*args, **kwargs):
            user = _current_user()
            if not user:
                abort(401)
            if not any(permission_granted(n, extract_permissions(user)) for n in names):
                abort(403)
            return fn(*args, **kwargs)
        return wrapper
    return decorator
```

Импорты `auth_connector` стоит обернуть в `try/except ImportError` с локальными копиями
`permission_granted`/`extract_permissions` — тогда сервис поднимается и без установленного пакета,
и поведение проверок не расходится с продом.

### 5.3. Пользователь в шаблонах

Шаблоны привыкли к `current_user` из Flask-Login. Прокси поверх `g.user`:

```python
@app.context_processor
def inject_current_user():
    raw = getattr(g, 'user', None)
    if isinstance(raw, (dict, UserContext)):
        return {'current_user': GatewayUserProxy(raw), 'is_gateway_mode': True}
    return {'current_user': None, 'is_gateway_mode': False}
```

`GatewayUserProxy` отдаёт `is_authenticated`, `id`, `username`, `full_name`, `short_name`,
`avatar_url`, `role`, `.can(perm)`. Полная реализация —
[apartment_finder/app/__init__.py](../../apartment_finder/app/__init__.py).

Две детали:

* **ФИО приходит в base64.** `AuthMiddleware` декодирует его сам, но при чтении заголовка руками —
  `base64.b64decode(v).decode('utf-8')`.
* **Аватара в `UserContext` нет** — поля нет и `to_dict()` его не возвращает. Путь берётся прямо из
  `request.headers['X-User-Avatar']`, он абсолютный от корня домена, префикс сервиса к нему не
  дописывается.

---

## 6. Переменные окружения

| Переменная | Кто читает | Обязательна | Значение |
|---|---|---|---|
| `AUTH_SERVICE_URL` | ваш код | под шлюзом да | `http://auth-service:80` |
| `INTERNAL_API_KEY` | `AuthClient`, `ServiceDiscoveryClient` | под шлюзом да | ключ auth-service |
| `JWT_SECRET` | `AuthMiddleware` | только для Bearer | тот же, что в auth-service |
| `CONTAINER_NAME` | `ServiceDiscoveryClient` | желательно | имя контейнера; иначе берётся hostname |
| `NOTIFICATION_SERVICE_URL` | ваш код | если шлёте письма | `http://notification-service:80` |
| `NOTIFICATION_API_KEY` | ваш код | если шлёте письма | персональный ключ; фоллбек — `INTERNAL_API_KEY` |
| `SECRET_KEY` | Flask | да | для сессий/flash |

`ServiceDiscoveryClient` и `AuthClient` подхватывают `INTERNAL_API_KEY` из окружения сами, если
`api_key=` не передан явно.

Получателя уведомления адресуйте **логином портала** (`login`), а не email/chat_id: адрес
доставки резолвит auth-service. Для получателя вне системы — `external_recipient`. Детали и
коды отказов — [GATEWAY_SERVICE_INTEGRATION_API.md](GATEWAY_SERVICE_INTEGRATION_API.md),
раздел 6.

---

## 7. Локальная разработка

Четыре уровня. Берите самый левый, который покрывает задачу — он же самый быстрый.

| Уровень | Что поднято | Когда |
|---|---|---|
| **0** | только Flask, без Docker, фейковый пользователь | вся обычная разработка фич |
| **1** | свой `docker-compose` без шлюза | нужны Postgres/Redis/воркеры |
| **2** | то же + настоящие заголовки через curl | проверка матрицы прав |
| **3** | полный шлюз локально | отладка nginx, `/verify`, реестра |

Все они опираются на один факт, зафиксированный в коде: режим `gateway-headers` в
[auth_middleware.py:135-160](../../auth-connector/auth_connector/auth_middleware.py#L135-L160) читает
**обычные HTTP-заголовки без подписи**. Ни JWT, ни ключей, ни обращений к auth-service.

> **Это и есть граница доверия.** Единственное, что не даёт постороннему прислать
> `X-User-Service-Permissions: finder.*` напрямую, — то, что сервис доступен только из
> `service_network`, а nginx перезаписывает эти заголовки своими на каждом запросе.
> Отсюда два правила без исключений: порт сервиса **никогда** не публикуется наружу
> (`ports:` в compose быть не должно), и любой код, подставляющий пользователя, включается
> только явным флагом окружения и отказывается работать в production.

### 7.1. Уровень 0 — без Docker и без шлюза

Файл `dev.py` рядом с боевой точкой входа. В образ он не попадает.

```python
"""Локальный запуск без шлюза. НЕ для прода."""
import os
import sys

# --- Предохранитель -------------------------------------------------------
# Подмена включается ТОЛЬКО явным флагом, значения по умолчанию нет намеренно.
# В проде переменную никто не выставляет, поэтому хук ниже остаётся мёртвым,
# даже если файл случайно попадёт в образ.
#
# Флаг именно свой, а не FLASK_ENV/ENVIRONMENT: .env сервисов выставляет
# FLASK_ENV=production и на машине разработчика тоже, так что проверка по
# нему даёт ложное срабатывание при первом же перезапуске от reloader'а.
LOCAL_DEV_AUTH = os.environ.get('LOCAL_DEV_AUTH') == '1'
# --------------------------------------------------------------------------

os.environ.setdefault('JWT_SECRET', 'local-dev')
# Проектная деталь: у finder модели estate_*/finance_* висят на bind
# 'mysql_source', и без него db.create_all() падает. Подставляем SQLite —
# таблицы создадутся пустыми.
#
# setdefault здесь ПЕРЕБИВАЕТ .env, а не уступает ему: строка выполняется до
# импорта app, то есть до load_dotenv(), а python-dotenv существующие
# переменные не перезаписывает. Локально это и нужно — в боевой MySQL с
# машины разработчика хода нет. Нужен настоящий источник — экспортируйте
# SOURCE_MYSQL_URI в оболочке перед запуском.
os.environ.setdefault('SOURCE_MYSQL_URI',
                      'sqlite:///' + os.path.abspath('instance/src_stub.db'))

from flask import g

from app import create_app
from app.core.config import DevelopmentConfig
from app.core.extensions import db
from auth_connector import AuthMiddleware

app = create_app(DevelopmentConfig)

# PrefixMiddleware намеренно не ставим: локально сервис живёт в корне,
# ссылки в шаблонах от этого не страдают.
AuthMiddleware(app, jwt_secret=os.environ['JWT_SECRET'])

DEV_USER = {
    'id': 1,
    'username': 'dev',
    'full_name': 'Локальный Разработчик',
    'roles': ['admin'],
    'permissions': os.environ.get('LOCAL_DEV_PERMISSIONS', 'finder.*').split(','),
}


@app.before_request
def fake_gateway_user():
    """Подставляет пользователя, когда заголовков шлюза нет.

    AuthMiddleware отрабатывает раньше и кладёт в g.user None — здесь только
    добиваем пустое значение, чтобы настоящие заголовки продолжали работать
    и уровень 2 не требовал правок кода.
    """
    if not LOCAL_DEV_AUTH:
        return
    if getattr(g, 'user', None) is None:
        g.user = dict(DEV_USER)


if __name__ == '__main__':
    if not LOCAL_DEV_AUTH:
        sys.exit('Задайте LOCAL_DEV_AUTH=1, иначе все страницы ответят 401')
    with app.app_context():
        db.create_all()
    print('[DEV] Авторизация подменена. Права:', DEV_USER['permissions'])
    app.run(host='127.0.0.1', port=5000, debug=True)
```

Запуск:

```bash
LOCAL_DEV_AUTH=1 PYTHONIOENCODING=utf-8 python dev.py
```

`PYTHONIOENCODING` не опция: в сервисных `print()` есть эмодзи, а консоль Windows на cp1251
валится с `UnicodeEncodeError: 'charmap' codec can't encode character '❌'`.

Проверка матрицы прав без перезапуска кода:

```bash
LOCAL_DEV_PERMISSIONS=finder.selection_view python dev.py   # только подбор
LOCAL_DEV_PERMISSIONS=                        python dev.py   # ничего: должны быть 403
```

**Проверено на `apartment_finder`** (Python 3.13, корневой `.venv`, без шлюза, без MySQL):

```
$ LOCAL_DEV_AUTH=1 PYTHONIOENCODING=utf-8 python dev.py
[DEV] Авторизация подменена. Права: ['finder.*']
/health                     -> 200
/                           -> 302 /selection
/selection                  -> 200
/home                       -> 200
/reports/inventory-summary  -> 200

$ python dev.py
Задайте LOCAL_DEV_AUTH=1, иначе все страницы ответят 401
```

Без `AuthMiddleware` те же адреса дают 401 — это и есть проверка, что предохранитель не сломан.

### 7.2. Уровень 1 — свой контейнер без шлюза

Отдельный `docker-compose.dev.yml`. Отличия от боевого: сети создаются локально, а не берутся как
`external`, и добавлен свой `.env.dev`.

```yaml
services:
  finder:
    build:
      context: ..
      dockerfile: apartment_finder/Dockerfile
    container_name: finder-dev
    env_file: [.env.dev]
    environment:
      - LOCAL_DEV_AUTH=1
      - FLASK_ENV=development
      - PYTHONIOENCODING=utf-8
    # Публикуем на loopback, не на 0.0.0.0: снаружи машины порт не виден,
    # а с подменённой авторизацией это принципиально.
    ports:
      - "127.0.0.1:5000:80"
    volumes:
      - ./instance:/app/instance
      - ./uploads:/app/uploads
    command: >
      gunicorn --bind 0.0.0.0:80 --worker-class gthread
      --workers 1 --threads 4 --timeout 120 --reload dev:app

networks:
  default:
    name: finder_dev_network
```

```bash
docker compose -f docker-compose.dev.yml up --build
```

Если удобнее держать один compose-файл, боевые сети создаются вручную и всё поднимается как в
проде, только без шлюза:

```bash
docker network create --driver bridge --internal service_network
docker network create --driver bridge --opt com.docker.network.bridge.enable_icc=false egress_network
```

Регистрация в реестре при этом будет падать в лог каждые несколько секунд — это нормально:
`ServiceDiscoveryClient.register()` делает 10 попыток с паузой 3 с и сдаётся, приложение работает
дальше.

### 7.3. Уровень 2 — настоящие заголовки

Убираем подмену (`LOCAL_DEV_AUTH=0`) и шлём то же, что шлёт nginx.

```bash
FULLNAME=$(printf 'Иванов Иван Петрович' | base64 -w0)

curl -s -o /dev/null -w '%{http_code}\n' http://localhost:5000/selection \
  -H "X-User-Id: 665f1a2b3c4d5e6f7a8b9c0d" \
  -H "X-User-Name: ivanov" \
  -H "X-User-Full-Name: $FULLNAME" \
  -H "X-User-Full-Name-Encoding: base64" \
  -H "X-User-Email: i@gh.uz" \
  -H "X-User-Service-Roles: manager" \
  -H "X-User-Service-Permissions: finder.selection_view,finder.discounts_view"
```

| Что проверяем | Заголовок прав | Ожидание |
|---|---|---|
| администратор | `finder.*` | 200 везде |
| узкая роль | `finder.selection_view` | 200 на подборе, 403 на скидках |
| нет прав | заголовок отсутствует | 403 |
| не авторизован | нет `X-User-Id`/`X-User-Name` | 401 |

Оба заголовка — `X-User-Id` **и** `X-User-Name` — обязательны: без любого из них
`extract_user_context()` вернёт `None`, и пользователь будет считаться неавторизованным.

Для ручной проверки в браузере подойдёт любое расширение-модификатор заголовков (ModHeader и
подобные) с тем же набором.

### 7.4. Уровень 3 — полный шлюз локально

Нужен, только когда отлаживаете сам nginx-роут, `/verify` или реестр.

```bash
docker network create --driver bridge --opt com.docker.network.bridge.enable_icc=false egress_network
cd '!gateway'
cp .env.example .env                      # MONGO_ROOT_PASSWORD, MONGO_APP_PASSWORD
cp auth-service/.env.example auth-service/.env   # MONGO_URI, JWT_SECRET, INTERNAL_API_KEY
docker compose up -d
```

Затем: завести сервис в `https://localhost/services/new`, поднять свой контейнер в
`service_network`, дождаться регистрации. Сертификат самоподписанный — браузер будет ругаться.

Порядок: сначала `!gateway`, потом сервисы. `service_network` создаёт compose шлюза; сервисы берут
её как `external`.

---

## 8. Что ломается локально и как чинить

Реальные ошибки, все воспроизведены на `apartment_finder`.

| Ошибка | Причина | Лечение |
|---|---|---|
| `ModuleNotFoundError: No module named 'auth_connector'` при `create_app()` | `permissions_setup` импортирует пакет жёстко, а импортирует его `sync_routes` | `pip install -e ./auth-connector` |
| Все роуты 401, `/health` 200 | точка входа не ставит `AuthMiddleware` | раздел 7.1 |
| `UnboundExecutionError: Bind key 'mysql_source' is not in 'SQLALCHEMY_BINDS' config` | bind добавляется только при непустом `SOURCE_MYSQL_URI`, а модели его требуют | `SOURCE_MYSQL_URI=sqlite:///instance/src_stub.db` — таблицы создадутся пустыми |
| `UnicodeEncodeError: 'charmap' codec can't encode character '❌'` | эмодзи в `print()`, консоль Windows на cp1251 | `PYTHONIOENCODING=utf-8` |
| `network service_network declared as external, but could not be found` | сети создаёт compose шлюза | раздел 7.2 |
| Registration attempt 1/10 failed: Cannot connect to registry | auth-service не поднят | штатно для локали; после 10 попыток клиент сдаётся |
| `url_for()` без префикса `/finder` | нет `ProxyFix(x_prefix=1)`/`PrefixMiddleware` | локально префикс не нужен вовсе |
| 404 на `/finder/...` локально | `PrefixMiddleware` подключён, но `X-Forwarded-Prefix` не приходит | либо снять middleware, либо ходить с префиксом — ветка `elif` его срежет |
| Пункт меню пропал у владельца `finder.*` | проверка через `in`, а не `permission_granted()` | раздел 5.1 |
| ФИО в кракозябрах | заголовок читается без декодирования base64 | раздел 5.3 |

---

## 9. Чеклист

**Подключение**

```
[ ] Сабмодуль auth-connector на нужной ветке, версия >= 2.0.0
[ ] Dockerfile: requirements.txt слоем ВЫШЕ auth-connector
[ ] Dockerfile: --force-reinstall + проверка мажорной версии
[ ] AuthMiddleware поставлен в точке входа
[ ] ProxyFix(x_prefix=1) + PrefixMiddleware
[ ] GET /health без чувствительных данных
[ ] GET /api/sync/permissions отдаёт success + permissions
[ ] permissions_setup.py: есть {key}.*
[ ] Проверки прав только через permission_granted()
[ ] Service discovery запускается вне пути flask-cli
[ ] Свои /login и /logout заменены редиректами на шлюз
```

**Локальная разработка**

```
[ ] dev.py в .dockerignore и не попадает в образ
[ ] Подмена пользователя включается только LOCAL_DEV_AUTH=1, значения по умолчанию нет
[ ] Порт публикуется на 127.0.0.1, а не 0.0.0.0
[ ] PYTHONIOENCODING=utf-8
[ ] Матрица прав проверена на трёх наборах: {key}.*, узкое право, пусто
[ ] Перед коммитом сервис проверен с настоящими заголовками (уровень 2)
```
