# mongo

Хранилище auth-service: пользователи, роли, права, сервисы, реестр инстансов,
токены. Единственный сервис, который сюда ходит, — auth-service.

- **Образ:** `mongo:6`, запуск с `--auth`
- **Контейнер:** `gateway-mongo-1` (имя не задано явно, собирается из проекта)
- **Сеть:** только `data_network` (`internal: true`) — портов наружу нет,
  из `service_network` база недоступна
- **Том:** bind-mount `./mongo_data` → `/data/db`
- **База:** `authdb`
- **Healthcheck:** `mongosh --eval "db.runCommand({ping:1}).ok"`

## Пользователи

| Пользователь | Права | Кто использует |
|---|---|---|
| `mongoadmin` | root | healthcheck, административные скрипты |
| `authservice` | `readWrite` на `authdb` | auth-service |

`authservice` создаётся скриптом `scripts/mongo-init.js`, смонтированным в
`/docker-entrypoint-initdb.d/01-create-app-user.js`. Скрипт отрабатывает
**только при первой инициализации** — когда `/data/db` пуст. Пароль он берёт
из `process.env.MONGO_APP_PASSWORD` и падает с `quit(1)`, если переменная
не задана.

Для **уже существующей** базы (прод) init-скрипт бесполезен — там ручная
процедура: [MONGODB_AUTH_MIGRATION.md](../../!gateway/MONGODB_AUTH_MIGRATION.md).

## Пароли

Задаются в `!gateway/.env` (шаблон `.env.example`):

| Переменная | Что |
|---|---|
| `MONGO_ROOT_PASSWORD` | пароль `mongoadmin` |
| `MONGO_APP_PASSWORD` | пароль `authservice` |

`MONGO_APP_PASSWORD` **обязан совпадать** с паролем внутри `MONGO_URI` в
`auth-service/.env`:

```
MONGO_URI=mongodb://authservice:<MONGO_APP_PASSWORD>@mongo:27017/authdb?authSource=authdb
```

Генерация: `openssl rand -base64 32`.

## Коллекции

| Коллекция | Содержимое |
|---|---|
| `users` | пользователи, профиль, документы, привязка Telegram |
| `services` | сервисы платформы и их каталоги прав |
| `service_roles` | роли внутри сервисов |
| `user_service_roles` | назначения «пользователь × сервис × роль» |
| `service_instances` | реестр живых инстансов (service discovery) |
| `roles`, `permissions` | легаси-справочники до ADR-001 |
| `user_sessions` | сессии |
| `blacklisted_tokens` | отозванные JWT |
| `password_reset_tokens` | токены восстановления пароля |
| `telegram_link_tokens` | токены привязки Telegram (TTL) |
| `telegram_login_requests` | запросы входа через Telegram (TTL) |
| `document_types` | справочник типов документов |
| `activity_logs`, `import_logs`, `service_import_logs` | аудит и логи импорта |

Схема моделей — [auth-service/models/SCHEMA_DESIGN.md](../../!gateway/auth-service/models/SCHEMA_DESIGN.md).

## Доступ для отладки

Порт наружу закрыт намеренно. Для локального разбора есть оверрайд:

```bash
docker compose -f docker-compose.yaml -f docker-compose.test.yml up -d mongo
# mongo появится на localhost:27017 и дополнительно войдёт в service_network
```

Не использовать на проде.

Изнутри контейнера:

```bash
docker exec -it gateway-mongo-1 mongosh -u authservice -p '<пароль>' \
  --authenticationDatabase authdb authdb
```

## Скрипты обслуживания

В `scripts/` лежат разовые операции, запускаемые через `mongosh`:

| Скрипт | Что делает |
|---|---|
| `mongo-init.js` | создаёт `authservice` при первой инициализации |
| `create_users_temp.js` | заводит пользователей вручную |
| `migrate_referal_user_roles.js` | миграция ролей referal |
| `../cleanup_migration.js` | чистка после миграции ADR-001 |
| `../auth-service/restore_client_service_roles.js` | восстановление ролей client-service (см. [QUICKFIX](../../QUICKFIX_CLIENT_SERVICE_ROLES.md) и [RESTORE_CLIENT_SERVICE_ROLES.md](../../!gateway/auth-service/RESTORE_CLIENT_SERVICE_ROLES.md)) |

## Бэкап

```bash
docker exec gateway-mongo-1 mongodump \
  -u mongoadmin -p '<MONGO_ROOT_PASSWORD>' --authenticationDatabase admin \
  --db authdb --archive=/data/db/backup-$(date +%F).archive
```

Файл окажется в `mongo_data/` на хосте. Перед любой миграцией схемы бэкап
обязателен — миграции auth-service идемпотентны, но не откатываемы
автоматически (`POST /migration/rollback` покрывает не всё).
