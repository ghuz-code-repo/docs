# Документация шлюза (`!gateway`)

Шлюз — единственная точка входа в аналитическую платформу Golden House. Он
принимает внешний трафик, проверяет личность пользователя, раздаёт права
и маршрутизирует запросы в бизнес-сервисы, которые сами наружу не смотрят.

Здесь описан **сам шлюз**. Как подключить к нему свой сервис — в
[GATEWAY_SERVICE_INTEGRATION_API.md](../integration/GATEWAY_SERVICE_INTEGRATION_API.md)
и [AUTH_CONNECTOR_SETUP_AND_LOCAL_DEV.md](../integration/AUTH_CONNECTOR_SETUP_AND_LOCAL_DEV.md).

## Карта сервисов

| Сервис | Контейнер | Технология | Порт (внутр.) | Наружу | Документ |
|---|---|---|---|---|---|
| nginx | `gateway-nginx-1` | nginx:alpine | 80, 443 | **да** (80/443) | [nginx.md](nginx.md) |
| auth-service | `gateway-auth-service-1` | Go + Gin | 80 | нет | [auth-service.md](auth-service.md) |
| mongo | `gateway-mongo-1` | mongo:6 | 27017 | нет | [mongo.md](mongo.md) |
| notification-service | `notification-service-notification-service-1` | Go + Gin + GORM | 80 | нет | [notification-service.md](notification-service.md) |
| notification-postgres | `notification-service-notification-postgres-1` | postgres:15-alpine | 5432 | нет | [notification-service.md](notification-service.md#база-данных) |
| notification-bot | `gateway-notification-bot` | Go + Gin | 80 | нет (исходящий Telegram API) | [notification-bot.md](notification-bot.md) |
| monitoring-service | `gateway-monitoring-service` | Go + Gin | 80 | нет | [monitoring-service.md](monitoring-service.md) |
| dozzle | `gateway-dozzle` | amir20/dozzle | 8080 | через `/logs` | [dozzle.md](dozzle.md) |
| docker-socket-proxy | `gateway-docker-proxy` | tecnativa/docker-socket-proxy | 2375 | нет | [docker-socket-proxy.md](docker-socket-proxy.md) |
| guard-watchdog | — (не запущен) | docker:27-cli + bash | нет портов | нет | [guard-watchdog.md](guard-watchdog.md) |
| quiz (внешний) | `quiz-backend`, `quiz-widget`, `quiz-admin` | чужой compose | 8000 / 80 / 80 | через `/quiz/*` | [quiz.md](quiz.md) |

Сквозная картина — сети, путь запроса, контракт `auth_request`, service
discovery — в [architecture.md](architecture.md).

## Что где лежит

```
!gateway/
├── docker-compose.yaml          # nginx, auth-service, mongo, notification-bot,
│                                # monitoring-service, docker-socket-proxy, dozzle
├── docker-compose.test.yml      # оверрайд: пробрасывает mongo на localhost:27017
├── .env                         # пароли MongoDB для docker-compose (см. .env.example)
├── auth-service/                # ядро авторизации (Go)
├── nginx/                       # конфиги, сертификаты, статика шлюза
├── notification-service/        # доставка уведомлений (свой docker-compose.yml)
├── notification-bot/            # Telegram-бот портала
├── monitoring-service/          # опрос health бизнес-сервисов (свой docker-compose.yaml)
├── watchdog/                    # авто-карантин по индикаторам компрометации (не запущен)
├── scripts/                     # mongo-init.js, quarantine.sh, разовые миграции
├── mongo_data/                  # том MongoDB (bind-mount)
└── docs/                        # этот каталог
```

## Порядок запуска

Всё поднимает `../../start_all.sh` (или `start_all.ps1`) из корня AnalyticsRepo:

1. `git pull` корня и `!gateway`
2. `docker network create egress_network` (с `enable_icc=false`), если её нет
3. `docker compose up --build -d` в `!gateway` — nginx, auth-service, mongo,
   notification-bot, monitoring-service, docker-socket-proxy, dozzle
4. Ожидание healthcheck `gateway-nginx-1` и `gateway-auth-service-1`
5. `docker compose up --build -d` в `!gateway/notification-service`
6. Бизнес-сервисы: `client_service`, `apartment_finder`, `referal`

`monitoring-service` описан **и** в общем `docker-compose.yaml`, **и** в
собственном `monitoring-service/docker-compose.yaml`. Рабочий путь — общий
файл (шаг 3); отдельный compose оставлен для запуска сервиса в одиночку.

Шлюз стартует даже когда ни один бизнес-сервис не поднят: динамический конфиг
nginx создаётся пустым, а `proxy_pass` через переменную резолвится в рантайме.

## Существующие документы шлюза

Каталог `docs/` — описание сервисов. Остальные файлы в корне `!gateway/` —
решения, миграции и руководства, они не дублируются здесь:

| Файл | О чём |
|---|---|
| [ADR-001-service-based-authorization.md](../../!gateway/ADR-001-service-based-authorization.md) | базовое архитектурное решение: права принадлежат сервисам, а не глобальному списку |
| [SERVICE_PERMISSIONS_GUIDE.md](../../!gateway/SERVICE_PERMISSIONS_GUIDE.md) | гранулярные права `auth.<service>.*` |
| [SERVICE_MANAGER_ROLE.md](../../!gateway/SERVICE_MANAGER_ROLE.md) | роль `service-manager` |
| [SERVICE_ADMIN_DELEGATION_ANALYSIS.md](../../!gateway/SERVICE_ADMIN_DELEGATION_ANALYSIS.md) | анализ делегирования админов сервисов |
| [MONGODB_AUTH_MIGRATION.md](../../!gateway/MONGODB_AUTH_MIGRATION.md) | включение `--auth` в работающей MongoDB |
| [DEPLOYMENT_GUIDE.md](../../!gateway/DEPLOYMENT_GUIDE.md) | настройка почты и восстановления пароля |
| [REGISTER_AUTH_SERVICE.md](../../!gateway/REGISTER_AUTH_SERVICE.md) | ручная регистрация auth-service в реестре |
| [USER_DELETION_ENHANCEMENT.md](../../!gateway/USER_DELETION_ENHANCEMENT.md) | каскадное удаление пользователя |
| [auth-service/PROFILE_FEATURES.md](../../!gateway/auth-service/PROFILE_FEATURES.md) | личный кабинет |
| [auth-service/SERVICE_IMPORT_EXPORT.md](../../!gateway/auth-service/SERVICE_IMPORT_EXPORT.md) | Excel-импорт пользователей сервиса |
| [notification-service/CONFIG.md](../../!gateway/notification-service/CONFIG.md) | полный справочник настроек уведомлений |
| [notification-service/LOGGING_GUIDE.md](../../!gateway/notification-service/LOGGING_GUIDE.md) | чтение логов доставки |
| [notification-service/TELEGRAM_GUIDE.md](../../!gateway/notification-service/TELEGRAM_GUIDE.md) | Telegram-канал уведомлений |
| [notification-bot/README.md](../../!gateway/notification-bot/README.md) | API бота и лимиты Telegram |

## Известные расхождения

Собраны в одном месте, чтобы не искать их по коду:

- **`/roles`, `/permissions`, `/admin-menu`** — у nginx есть `location` для этих
  путей, но в auth-service соответствующих роутов нет: `routes/role_management.go`
  переименован в `.disabled` и не компилируется. Пути отдают 404.
- **`POST /api/v1/security/alert`** — `watchdog/watchdog.sh` шлёт туда алерты,
  но в notification-service такой роут не зарегистрирован. Плюс сам watchdog
  ни в одном compose-файле не описан. См. [guard-watchdog.md](guard-watchdog.md).
- **Проверка `Origin`/`Referer` для quiz отключена** — в `nginx/conf/quiz.inc`
  строки `if ($quiz_gh_uz_denied) { return 444; }` закомментированы «временно
  для тестирования», а `$quiz_cors_origin` зеркалит любой Origin.
  См. [quiz.md](quiz.md).
- **`notification-service/README.md` называет порт 8082** — фактически сервис
  слушает `:80` (`PORT` по умолчанию `80`, наружу не публикуется).
