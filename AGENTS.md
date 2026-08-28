# AGENTS.md — брифинг для ИИ-агента

Читай этот файл первым, если работаешь с платформой Golden House: со шлюзом
(`!gateway`) или с сервисом, который к нему подключается.

Этот файл **маршрутизирует, а не заменяет** документацию. Он короткий
намеренно: пересказ контракта здесь протух бы первым.

Всё сверено с кодом. Если код и документ расходятся — прав код, а документ
надо поправить.

---

## 0. Обязательный порядок чтения

**Одного этого файла недостаточно.** Прежде чем писать или менять код,
прочитай как минимум:

| Когда | Что обязательно прочитать |
|---|---|
| **всегда**, любая задача по интеграции | [integration/GATEWAY_SERVICE_INTEGRATION_API.md](integration/GATEWAY_SERVICE_INTEGRATION_API.md) — контракт целиком |
| код сервиса на Python/Flask | + [integration/AUTH_CONNECTOR_REFERENCE.md](integration/AUTH_CONNECTOR_REFERENCE.md) — что пакет умеет |
| подключение с нуля, локальный запуск, Dockerfile | + [integration/AUTH_CONNECTOR_SETUP_AND_LOCAL_DEV.md](integration/AUTH_CONNECTOR_SETUP_AND_LOCAL_DEV.md) |
| что-то отправляется пользователю | + [integration/TELEGRAM_NOTIFICATIONS_INTEGRATION_GUIDE.md](integration/TELEGRAM_NOTIFICATIONS_INTEGRATION_GUIDE.md) |
| правка самого шлюза | + профильный файл из [gateway/](gateway/README.md) |

Порядок: **AGENTS.md → integration/ → gateway/ → код**. Инварианты из §3
ниже — выжимка, а не полный список; отвечать «по памяти», не открыв
`GATEWAY_SERVICE_INTEGRATION_API.md`, нельзя.

Обязательства, действующие в **каждой** задаче: §7 (тесты), §8
(документация) и §10 (правила репозитория). Если правишь сам шлюз — плюс §9.
Их не пропускают, даже когда просили «просто поправить одну функцию».

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
| API пакета `auth-connector`: классы, декораторы, сигнатуры | [integration/AUTH_CONNECTOR_REFERENCE.md](integration/AUTH_CONNECTOR_REFERENCE.md) |
| подключить `auth-connector` к Flask, поднять сервис локально | [integration/AUTH_CONNECTOR_SETUP_AND_LOCAL_DEV.md](integration/AUTH_CONNECTOR_SETUP_AND_LOCAL_DEV.md) |
| слать уведомления, в том числе в Telegram | [integration/TELEGRAM_NOTIFICATIONS_INTEGRATION_GUIDE.md](integration/TELEGRAM_NOTIFICATIONS_INTEGRATION_GUIDE.md) |
| понять сети, путь запроса, service discovery | [gateway/architecture.md](gateway/architecture.md) |
| роуты, права, миграции, MongoDB auth-service | [gateway/auth-service.md](gateway/auth-service.md) |
| конфиги nginx, rate-limit, заголовки | [gateway/nginx.md](gateway/nginx.md) |
| страницы ошибок, перехват ответов сервисов | [gateway/nginx.md](gateway/nginx.md) §Обработка ошибок |
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
13. **Ошибки рисует шлюз, а не сервис.** `proxy_intercept_errors on` на всём
    server-блоке: ответ сервиса с кодом `4xx`/`5xx` теряет тело и заменяется
    страницей шлюза. Своя страница ошибки не нужна, а JSON с кодом ошибки под
    префиксом сервиса до `fetch()` не дойдёт. Причину передают заголовками
    `X-Error-Code` и `X-Error-Detail` — раздел 8.1
    [GATEWAY_SERVICE_INTEGRATION_API.md](integration/GATEWAY_SERVICE_INTEGRATION_API.md).
    Исключения: `<prefix>/static/`, `<prefix>/health`, `/api/` шлюза.

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

Эталон тестов для Python — `client_service/test_housing_backfill.py`, см. §7.

---

## 6. Что в системе сломано или отключено

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
- `AuthClient.get_user_permissions()` и `validate_token()` бьют в эндпоинты,
  которых в auth-service нет. Права — из заголовка, токен проверяет шлюз.
- `ServiceDiscoveryClient` по умолчанию целится в
  `http://auth-service:8080/api/registry`, а auth-service слушает `:80` —
  `registry_url` передавать явно.
- `!gateway/auth-service/tests/main_test.go` держит реквизиты MongoDB
  захардкоженной константой `testMongoURI`. Значение совпадает с локальным
  `!gateway/.env`, **в проде оно не используется** — это вопрос гигиены, а не
  инцидент. Заменить на чтение переменной окружения с безопасным значением по
  умолчанию; новые тесты так не писать (§7, правило 3).

---

## 7. Тесты и локальная проверяемость

**Обязательное требование, не пожелание.** Код, который нельзя запустить и
проверить на машине разработчика без прода, считается несданным.

### Правила

1. **Каждое изменение поведения приходит с тестом.** Новый роут, новая
   проверка прав, новая ветка обработки ошибки — значит новый тест.
   Исправление бага — сначала тест, воспроизводящий баг, потом фикс.
2. **Тесты не ходят в сеть.** Ни в auth-service, ни в notification-service,
   ни во внешние API. Личность подделывается заголовками, HTTP-клиенты
   мокаются. Заголовки шлюза — обычный `dict`, поэтому мок тривиален
   (пример — раздел «Тестируемость» в
   [integration/AUTH_CONNECTOR_REFERENCE.md](integration/AUTH_CONNECTOR_REFERENCE.md)).
3. **Тесты не требуют прода.** Боевые MySQL/MongoDB недоступны с машины
   разработчика; подставляется SQLite или фикстура. Схема подмены — раздел 7.1
   [integration/AUTH_CONNECTOR_SETUP_AND_LOCAL_DEV.md](integration/AUTH_CONNECTOR_SETUP_AND_LOCAL_DEV.md).
4. **Одна команда на запуск.** `pytest` для Python, `go test ./...` для Go —
   без предварительного поднятия docker-compose. Если для запуска нужны шаги,
   их описывает `README` сервиса (§8).
5. **Сервис должен подниматься локально без шлюза.** Уровни 0–3 локальной
   разработки описаны в разделе 7 гайда по auth-connector; уровень 0 (без
   Docker и без шлюза) обязан работать всегда.

### Минимальный обязательный набор для сервиса

```
[ ] GET /health → 200
[ ] GET /api/sync/permissions → success: true, в списке есть <key>.*
[ ] каждый защищённый роут: доступ разрешён / 403 / 401 без личности
[ ] wildcard <key>.* открывает роут  ← регресс на самую частую ошибку
[ ] url_for() под префиксом даёт /<key>/...
[ ] бизнес-логика: happy path + граничные случаи + обработка ошибок
```

### Что есть сейчас

| Где | Состояние |
|---|---|
| `!gateway/auth-service/tests/` | 9 файлов Go-тестов: роли, права, middleware, CRUD сервисов, внешние роли |
| `!gateway/auth-service/{models,routes}` | `recipients_test.go`, `recipient_handlers_test.go` |
| `!gateway/notification-service/` | `limiter_test.go`, `priority_test.go`, `recipients_test.go`, `resolve_test.go`, `transports_test.go` |
| `!gateway/nginx/error-pages/tests/` | pytest: страницы ошибок, карта `error_page`, обвязка nginx. Ни сети, ни докера |
| `client_service/test_housing_backfill.py` | **эталон для Python** — pytest, SQLite in-memory, `TestConfig`, моки |
| `client_service/test_housing_api.py` | не тест: ручной скрипт с `sys.path.insert('/app')`, требует боевую БД, печатает в stdout. Так не делать |
| `referal`, `apartment_finder`, `scriptovod` | тестов нет |

Эталон для Python-сервиса — `client_service/test_housing_backfill.py`:
фикстура приложения на `create_app(TestConfig)`, `SQLALCHEMY_DATABASE_URI =
'sqlite:///:memory:'`, `MAIL_SUPPRESS_SEND = True`, внешние вызовы под
`unittest.mock.patch`. Ни сети, ни боевой БД.

Для Go — тесты шлюза. Инфраструктуру (`pytest`, `conftest.py`, фикстуру
приложения) в сервисах без тестов заводить самому и описывать в `README`.
Не оставляй сервис без тестов на том основании, что их там не было.

### Тесты при работе со шлюзом

Два разных слоя, не путать:

| Где | Тип | Как запускать |
|---|---|---|
| `!gateway/notification-service/*_test.go` | **юнит**, БД и сеть не нужны | `cd '!gateway/notification-service' && go test ./...` |
| `!gateway/nginx/error-pages/tests/` | **юнит**, читают файлы репозитория | `cd '!gateway/nginx/error-pages' && pytest` |
| `!gateway/auth-service/models`, `routes` | юнит | `cd '!gateway/auth-service' && go test ./models/... ./routes/...` |
| `!gateway/auth-service/tests/` | **интеграционные**, нужна живая MongoDB на `localhost:27017` | поднять mongo оверрайдом, затем `go test ./tests/...` |

```bash
# MongoDB для интеграционных тестов auth-service
cd '!gateway'
docker compose -f docker-compose.yaml -f docker-compose.test.yml up -d mongo
cd auth-service && go test ./...
```

Правила для кода шлюза:

1. **Новый код покрывать юнит-тестами**, а не интеграционными. Образец —
   `notification-service/priority_test.go`: сервис собирается с заранее
   заполненным кэшем каналов, БД для проверки логики не нужна. Выноси чистую
   логику из хендлера в функцию, которую можно позвать без Mongo и без gin.
2. **Интеграционный тест — только когда без БД никак** (индексы, атомарные
   `FindOneAndUpdate`, миграции).
3. **Секретов в тестах быть не должно.** Реквизиты — из окружения с
   безопасным значением по умолчанию. Пароль, попавший в тест, попадает
   в историю git навсегда — см. §6.
4. **Меняешь одну из четырёх поверхностей совместимости (§9) — тест
   обязателен**: они ломают все сервисы разом, и регресс здесь дороже всего.

---

## 8. Документация — обязательна в каждой задаче

Правило одной строкой: **где менял код, там и документируй.**

| Что менял | Куда пишешь |
|---|---|
| конечный сервис | документы в репозитории **этого сервиса** (§8.1) |
| шлюз `!gateway` | сабмодуль **`docs/`** (§8.2) |
| контракт платформы | сабмодуль **`docs/`**, даже если правка была в коде сервиса |

Задача без обновлённой документации не считается сделанной. «Потом
задокументирую» отдельной задачей не оставляют.

### 8.1. Работаешь с конечным сервисом → документы в его репозитории

В `docs/` живёт платформа и контракт; как устроен конкретный сервис — его дело.

#### Что агент обязан поддерживать в репозитории сервиса

| Файл | Содержание |
|---|---|
| `README.md` | назначение, `service_key` и префикс URL, как поднять локально, **как прогнать тесты**, переменные окружения |
| `docs/permissions.md` (или раздел README) | каталог прав: имя, что открывает, кому выдаётся |
| `docs/architecture.md` (если сервис нетривиален) | модули, схема данных, внешние зависимости, фоновые задачи |
| `docs/integration.md` | чем сервис ходит наружу: вызовы auth-service, уведомления, чужие API |
| `CHANGELOG` или раздел в README | ломающие изменения — обязательно |

Минимум для маленького сервиса — один `README.md`, закрывающий первую строку
таблицы. Отдельные файлы заводить, когда README перестаёт читаться.

#### Правила

1. **Новое право — строка в каталоге прав.** Право, которого нет в документе,
   администратор не выдаст осознанно.
2. **Ломающее изменение — явной пометкой.** Смена `service_key`, формата
   заголовка, схемы БД, контракта API.
3. **Не дублируй контракт.** В сервисе — ссылка на
   [integration/GATEWAY_SERVICE_INTEGRATION_API.md](integration/GATEWAY_SERVICE_INTEGRATION_API.md),
   а не пересказ. Копия протухнет.

### 8.2. Работаешь со шлюзом → сабмодуль `docs/`

Любая правка в `!gateway` обязана дойти до этого сабмодуля. Документация
шлюза лежит **не в репозитории шлюза** — в `!gateway/docs/README.md` только
указатель сюда.

**Изменил в шлюзе → обнови в `docs/`:**

| Изменение в `!gateway` | Что править |
|---|---|
| роут в `auth-service/routes/*.go` | таблицы API в [gateway/auth-service.md](gateway/auth-service.md); если роут вызывают сервисы — ещё раздел 5 [integration/GATEWAY_SERVICE_INTEGRATION_API.md](integration/GATEWAY_SERVICE_INTEGRATION_API.md) |
| middleware, модель прав, `permissions.json` | [gateway/auth-service.md](gateway/auth-service.md), разделы про middleware и права |
| **заголовки `X-User-*` в `verifyHandler`** | [gateway/architecture.md](gateway/architecture.md) §Контракт `auth_request` + §4 этого файла + раздел 3 контракта. **Ломающее для всех сервисов** |
| **коды ответа `/verify`** | там же; ломающее |
| шаблон конфига в `routes/nginx_config.go` | [gateway/auth-service.md](gateway/auth-service.md) §Генерация конфигов + [gateway/nginx.md](gateway/nginx.md) + [gateway/architecture.md](gateway/architecture.md) §Service discovery |
| **API реестра `/api/registry/*`** | [gateway/architecture.md](gateway/architecture.md) + контракт, раздел 5.1. Ломающее для `auth-connector` |
| `nginx/conf/*.inc`, `nginx.conf` | [gateway/nginx.md](gateway/nginx.md); блоки quiz — [gateway/quiz.md](gateway/quiz.md) |
| каналы, лимиты, приоритеты уведомлений | [gateway/notification-service.md](gateway/notification-service.md) + `!gateway/notification-service/CONFIG.md` |
| контракт `/api/v1/notifications` | там же + раздел 6 контракта + [integration/TELEGRAM_NOTIFICATIONS_INTEGRATION_GUIDE.md](integration/TELEGRAM_NOTIFICATIONS_INTEGRATION_GUIDE.md) |
| `notification-bot`: API, лимиты | [gateway/notification-bot.md](gateway/notification-bot.md) + `!gateway/notification-bot/README.md` |
| `docker-compose.yaml`: сервис, сеть, порт | карта сервисов в [gateway/README.md](gateway/README.md) + [gateway/architecture.md](gateway/architecture.md) §Сети |
| **новый контейнер в шлюзе** | новый файл `docs/gateway/<name>.md` + строка в карте сервисов |
| любой `.env.example` | таблица переменных в профильном файле |
| коллекции или схема MongoDB | [gateway/mongo.md](gateway/mongo.md) + [gateway/auth-service.md](gateway/auth-service.md) |
| фильтры логов, `getAllowedLogsServices` | [gateway/dozzle.md](gateway/dozzle.md) |
| разрешённые операции Docker API | [gateway/docker-socket-proxy.md](gateway/docker-socket-proxy.md) |
| `auth-connector` (сабмодуль) | [integration/AUTH_CONNECTOR_REFERENCE.md](integration/AUTH_CONNECTOR_REFERENCE.md); при смене версии — ещё раздел 1.1 setup-гайда |

**Правила для `docs/`:**

1. **Починил то, что перечислено в §6 — вычеркни строку оттуда.** Список
   «что сломано» врёт быстрее всего остального.
2. **Ломающее контракт изменение — правь §3 (инварианты) и §4 этого файла.**
   Иначе следующий агент будет работать по устаревшей выжимке.
3. **Новое расхождение кода и документов, которое не чинишь сейчас, — впиши
   в §6.** Молча оставлять нельзя.
4. **Правь документ, а не подгоняй код под текст.** Прав код.
5. **Не дублируй.** Глубина — в профильном файле, в остальных ссылка.
   `AGENTS.md` маршрутизирует, а не пересказывает.
6. **Секретов в документации нет.** Ни паролей, ни ключей, ни токенов —
   только имена переменных.
7. `docs/` — отдельный git-репозиторий. Правки в нём коммитит человек (§10),
   но **файлы обязан подготовить агент в том же заходе**.

---

## 9. Работа с самим шлюзом

Дополнительно к §7, §8 и §10 — для задач в `!gateway`.

### Четыре поверхности совместимости

Ломают **все** подключённые сервисы разом. Менять только осознанно, с тестом,
с правкой документации и с пометкой «ломающее»:

| Поверхность | Где | Что сломается |
|---|---|---|
| заголовки `X-User-*` | `routes/auth_handlers.go` → `verifyHandler` | личность и права во всех сервисах |
| коды `/verify` (`200/401/403`) | там же | редиректы на логин и отказ доступа |
| шаблон конфига nginx | `routes/nginx_config.go` | маршрутизация всех сервисов, статика, загрузки |
| API реестра `/api/registry/*` | `routes/service_registry.go` | регистрация из `auth-connector` |

### Правила

1. **Стартовые миграции идемпотентны.** Всё в `main.go` (`Ensure*`, `Migrate*`,
   `Cleanup*`) выполняется на каждом старте. Новый шаг обязан быть безопасным
   при повторном запуске и не должен ронять старт при ошибке — только лог.
2. **Не дерегистрировать сервис автоматически и не удалять его маршрут.**
   `unregister` намеренно не трогает конфиг nginx: иначе рестарт контейнера
   даёт пользователям `404` вместо `502`.
3. **Не ослаблять изоляцию контейнеров.** Не добавлять `ports` наружу, не
   монтировать `/var/run/docker.sock` (для этого есть
   [docker-socket-proxy](gateway/docker-socket-proxy.md)), не снимать
   `cap_drop: ALL` и `no-new-privileges`, не включать `EXEC=1` у прокси.
4. **Не расширять сети.** Сервис из `service_network` не должен получать
   `data_network`; выход в интернет — только `egress_network`.
   Схема — [gateway/architecture.md](gateway/architecture.md) §Сети.
5. **Секреты только в `.env`.** В коде, тестах и документации — имена
   переменных. `.env.example` обновлять вместе с кодом, который читает новую
   переменную.
6. **Обязательные переменные проверять на старте.** `MONGO_URI` и
   `INTERNAL_API_KEY` роняют сервис при отсутствии намеренно: молчаливый
   старт без ключа оставил бы `/api/*` открытым.
7. **Перезагрузка nginx — сигналом**, `POST /containers/<name>/kill?signal=HUP`
   через docker-socket-proxy. Не `exec`: `EXEC=0` держится специально.
8. **Правишь `!gateway` — правишь два репозитория.** Код в сабмодуле
   `!gateway`, документация в сабмодуле `docs`. Коммитов два.

### Локальная проверка шлюза

```bash
cd '!gateway'
docker compose up -d --build            # весь шлюз
docker compose logs -f auth-service
docker exec gateway-nginx-1 nginx -t    # конфиг валиден
docker kill -s HUP gateway-nginx-1      # применить правку conf/*.inc и страниц ошибок
curl -k https://localhost/health

# после правки .env — restart НЕ перечитывает env_file
docker compose up -d --force-recreate auth-service
```

Конфиги в `nginx/conf/dynamic/` генерируются автоматически — руками не править,
изменения перетрутся при следующей регистрации сервиса.

---

## 10. Правила работы в этом репозитории

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
  подстановок съест восклицательный знак. В PowerShell проблемы нет, но там
  нет `rm -rf`, `&&` и `mv`-переименования в существующий каталог.
- **Мобильная вёрстка шлюза**: `static/css/mobile.css` грузится последним и
  одноэкранная раскладка включена только при `min-width: 769px`. Остальные
  сервисы этого слоя пока не имеют.
- **Библиотеку предпочитать самописному**: для сложного UI и алгоритмов брать
  готовое решение, а не писать с нуля. С поправкой на инвариант №9 — вендорить
  локально, не с CDN.

---

## 11. Диагностика: симптом → причина

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
| `fetch()` получил HTML вместо JSON | шлюз перехватил `4xx`/`5xx` и подменил тело страницей ошибки |
| на странице ошибки «шлюз портала» вместо имени сервиса | запрос не дошёл до сервиса: префикс не зарегистрирован либо ошибка на уровне шлюза |
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

## 12. Чеклист перед сдачей

### Интеграция

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

Блок «Интеграция» — для задач в конечном сервисе. Правишь шлюз — вместо него
блок «Шлюз» ниже. Блоки «Тесты» и «Документация» обязательны всегда.

### Тесты (§7)

```
[ ] Тесты запускаются одной командой, без сети и без прода
[ ] Сервис поднимается локально без шлюза (уровень 0)
[ ] Покрыты /health и /api/sync/permissions
[ ] Покрыт каждый защищённый роут: разрешён / 403 / 401
[ ] Покрыт wildcard <key>.*
[ ] Изменённое поведение покрыто новым тестом; исправленный баг — регрессом
```

### Шлюз (§9) — если правил `!gateway`

```
[ ] Ни одна из четырёх поверхностей совместимости не сломана молча
[ ] Новый стартовый шаг идемпотентен и не роняет старт при ошибке
[ ] Маршрут сервиса не удаляется при дерегистрации
[ ] Изоляция не ослаблена: нет новых ports, docker.sock, cap_add, EXEC=1
[ ] Сети не расширены
[ ] Новая переменная окружения добавлена в .env.example
[ ] Секретов в коде и тестах нет
[ ] docker exec gateway-nginx-1 nginx -t проходит
[ ] go test ./... зелёный в затронутом модуле
```

### Документация (§8)

```
[ ] Правил сервис  → README сервиса: запуск, тесты, переменные окружения
[ ] Правил сервис  → каталог прав актуален, каждое право объяснено
[ ] Правил шлюз    → обновлены файлы docs/ по таблице §8.2
[ ] Правил шлюз    → новый контейнер получил свой файл в docs/gateway/
[ ] Сломал контракт → правлены §3 и §4 AGENTS.md, помечено как ломающее
[ ] Починил пункт из §6 → строка оттуда вычеркнута
[ ] Новое расхождение кода и документов вписано в §6
[ ] Секретов в документации нет
```
