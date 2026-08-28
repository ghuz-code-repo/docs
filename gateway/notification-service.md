# notification-service

Единая очередь доставки уведомлений: email, Telegram (через
[notification-bot](notification-bot.md)), заготовки SMS и push. Вызывающие
сервисы не держат ни SMTP-паролей, ни контактов пользователей.

- **Каталог:** `notification-service/` (собственный `docker-compose.yml`)
- **Технология:** Go 1.23, Gin, GORM, PostgreSQL 15
- **Слушает:** `:80` (`PORT`), наружу не публикуется
- **Сети:** `notification_network` (БД, internal), `service_network` (приём
  запросов), `smtp_network` (исходящий SMTP)
- **Healthcheck:** `GET /api/v1/health`
- **Логи:** bind-mount `./logs` → `/var/log/notification-service`

Подробные справочники рядом с кодом:
[README.md](../../!gateway/notification-service/README.md),
[CONFIG.md](../../!gateway/notification-service/CONFIG.md),
[TELEGRAM_GUIDE.md](../../!gateway/notification-service/TELEGRAM_GUIDE.md),
[LOGGING_GUIDE.md](../../!gateway/notification-service/LOGGING_GUIDE.md),
[CHANGELOG_TELEGRAM.md](../../!gateway/notification-service/CHANGELOG_TELEGRAM.md).

## Модель адресации

Получателя описывает **ровно одно** именованное поле запроса:

| Поле | Режим | Что делает сервис |
|---|---|---|
| `login` | `login` | спрашивает адрес у auth-service (`POST /api/recipients/resolve`), кэш 10 мин |
| `external_recipient` | `external` | берёт значение как есть |
| — (только для `telegram_system`) | `system` | подставляет `system_telegram_chat_id` из конфига |

Два поля сразу или ни одного — `400`. Угадывать по виду строки («похоже на
email») сервис не будет: ошибка адресации молча уводит уведомление не туда.

В БД пишутся оба среза: `recipient` — фактический адрес доставки (для аудита),
`recipient_login` / `external_recipient` — чем адресовали, `failure_code` —
машинная причина отказа.

Таблица режимов задана `addressingFields` в
[recipients.go](../../!gateway/notification-service/recipients.go); добавление нового вида
адресации (например, «всем с правом X») не требует правки разбора запроса,
батчей и хендлеров.

## API

Все эндпоинты кроме `/api/v1/health` требуют `X-API-Key`.

| Метод | Путь | Назначение |
|---|---|---|
| GET | `/api/v1/health` | healthcheck, без аутентификации |
| POST | `/api/v1/notifications` | одно уведомление |
| POST | `/api/v1/notifications/batch` | пачка |
| GET | `/api/v1/notifications/:id` | статус уведомления |
| GET | `/api/v1/batches/:batch_id` | статус пачки |
| GET | `/api/v1/batches/:batch_id/notifications` | уведомления пачки |
| GET/POST | `/api/v1/config` | глобальная конфигурация (SMTP, системные получатели) |
| GET | `/api/v1/channels` | настройки всех каналов |
| POST | `/api/v1/channels/:channel` | правка одного канала |
| GET | `/`, `/config` | редирект на `/static/config.html` (веб-форма настроек) |

### Отправка

```json
POST /api/v1/notifications
X-API-Key: <ключ сервиса>

{
  "type": "email",
  "login": "ivanov",
  "subject": "Заявка одобрена",
  "content": "<p>Готово</p>",
  "content_type": "text/html",
  "priority": 0
}
```

`type`: `email` | `sms` | `push` | `telegram` | `telegram_system`.
`content_type`: `text/plain` (по умолчанию) | `text/html`.
Вложение — `attachment_filename` + `attachment_content` (base64).
`priority` необязателен и принимается только **не выше** потолка отправителя.

Батч резолвит всех получателей одного канала **одним** запросом к auth-service.
Нерезолвнутый получатель создаётся сразу со статусом `failed` и `failure_code`
— так вызывающий сервис видит причину, а отправки не происходит. Если
auth-service недоступен, пачка целиком отвергается с `503` и
`failure_code: auth_unavailable` — наполовину созданной пачки не бывает.

Машинные коды отказа: `auth_unavailable`, `unknown_channel`, `user_not_found`,
`send_failed`, плюс коды резолва от auth-service (`user_banned`,
`no_service_access`, `channel_not_linked`, `no_address`).

### Ханипоты

`/.env`, `/api/v1/internal/keys`, `/api/v1/admin/exec`, `/api/v1/debug/pprof`
отвечают обычным `404`, но пишут в лог маркер `GUARD TRIPWIRE: ip=…`.
Легитимного трафика на этих путях не существует, поэтому обращение —
бинарный индикатор компрометации источника; маркер ловит
[guard-watchdog](guard-watchdog.md).

## Каналы доставки

У каждого канала — своя очередь, свой темп, свои таймауты и свои повторы.
Очередь почты не задерживает Telegram. Настройки живут в таблице
`channel_configs` и правятся через `/api/v1/channels/:channel` либо через
страницу настроек auth-service (`/notification-settings`, секция «Каналы
доставки»). Переменные `MAX_RETRY_ATTEMPTS`, `BATCH_SIZE`,
`DELAY_BETWEEN_*` используются **только при первом создании строк** и на
отправку больше не влияют.

### Заводские значения

| Параметр | email | telegram |
|---|---|---|
| темп | 1 сообщение / 4 с (15/мин), burst 1 | 1800/мин (30/с), burst 30 |
| лимит на получателя | — | 60/мин, burst 1; класс `group` — 20/мин |
| приоритетная полоса | 75 % окна | 75 % окна |
| таймауты connect / send | 10 с / 30 с | 5 с / 15 с |
| параллельность | 1 (одно SMTP-соединение) | 8 |
| попыток | 3, backoff 5 с → 300 с | 3, backoff 2 с → 120 с |
| `max_inline_wait` / на получателя | 15 с / 1 с | 10 с / 0.25 с |

Темп почты продиктован администраторами почтового сервера (не более 20/мин;
держим запас 15). Темп Telegram — документированные лимиты Bot API.
**Завышать нельзя**: превышение даёт `429` на весь бот портала, включая
подтверждение входа.

### Транспорты

`transports.go`, интерфейс `Transport`: `Channel()`, `Types()`,
`LimiterKey()`, `RecipientClass()`, `Send()`, `Classify()`.
Реализованы `emailTransport` (SMTP), `telegramTransport` (через
notification-bot), заглушки `smsTransport` и `pushTransport`.
`Classify()` разбирает ошибку на «постоянная» (повторять бессмысленно),
«лимит темпа» (`retry_after`) и «временная».

Добавление канала = реализация `Transport` + строка в `channel_configs`;
лимитер, диспетчер и страница настроек подхватят его сами.

## Приоритетная полоса

Очередь канала упорядочена по приоритету, затем по времени создания.
Причина — жёсткая квота почты: рассылка на сотню адресов занимает семь минут,
и письмо со сбросом пароля всё это время ждало бы своей очереди.

- Потолок задаётся **сервису-отправителю**: `SERVICE_PRIORITIES` (общий для
  всех каналов) или `service_priorities` в настройках канала (переопределяет).
- Потолок он же значение по умолчанию — код вызывающего сервиса менять не надо.
- Запрос может попросить приоритет **не выше** потолка: так отправитель с
  полосой сам опускает вниз свои массовые рассылки.
- Уровни: `0` (bulk) и `100` (high). По умолчанию высокий приоритет только у
  `auth-service` — пользователь ждёт письмо перед экраном.
- Отправитель с общим `INTERNAL_API_KEY` неотличим от прочих легаси-сервисов
  и приоритета **не получит**.
- `priority_window_share_percent` (75 %) резервирует часть окна за обычной
  очередью — без резерва приоритетный поток вытеснил бы рассылки навсегда.

## Аутентификация вызывающих

`SERVICE_API_KEYS=referal:…,client-service:…,apartment-finder:…,monitoring-service:…,auth-service:…`
— персональный ключ на сервис. При компрометации отзывается только его ключ,
и в логах видно, кто отправитель. Легаси `INTERNAL_API_KEY` принимается, но
отправитель числится как `internal-shared-key`: без пер-сервисного лимита и
без приоритета.

Если **ни один** ключ не задан — аутентификация отключается (переходный режим,
в лог идёт предупреждение).

### `SERVICE_AUTH_KEY_MAP`

Имя сервиса в `SERVICE_API_KEYS` не всегда совпадает с `service_key` в
auth-service. Пример: ключ уведомлений `apartment-finder`, а `service_key`
у него — `finder`. Без маппинга проверка доступа получателя отвергла бы всех.
Пустое значение справа (`auth-service=`, `monitoring-service=`) отключает
проверку доступа — эти отправители шлют письма портала в целом, а не своего
сервиса.

## Security guard

`security.go` — градуированный ответ на аномалии, без авто-отключения от сети
(ложное срабатывание не должно превращаться в полный отказ доставки):

| Событие | Порог | Реакция |
|---|---|---|
| много запросов от одного сервиса | `GUARD_MAX_REQUESTS_PER_WINDOW` (600 / 60 с) | `429` на `GUARD_THROTTLE_SECONDS` (300 с) + алерт |
| невалидные `X-API-Key` с одного IP | `GUARD_MAX_INVALID_KEYS_PER_WINDOW` (10 / 60 с) | временный бан IP (`401`) + алерт |
| обращение к ханипоту | первое же | лог-маркер `GUARD TRIPWIRE` + алерт |

Повторные алерты по одному источнику подавляются `GUARD_ALERT_COOLDOWN_SECONDS`
(900 с). Реальный карантин — только вручную, `scripts/quarantine.sh`.

## База данных

PostgreSQL 15 в `notification_network` (`internal: true`) — портов наружу нет,
том `notification_postgres_data`. Таблицы (GORM, автомиграция):

| Таблица | Содержимое |
|---|---|
| `notifications` | очередь и история: тип, адрес, статус, попытки, `failure_code`, приоритет, `next_attempt_at`, `rate_limit_hits` |
| `notification_batches` | пачки: общее число, статус |
| `notification_configs` | SMTP, системные получатели, флаги каналов, debug-режим |
| `channel_configs` | параметры каналов (см. выше) |

Составной индекс `idx_notifications_queue` по `status` + `next_attempt_at` +
`priority` — по нему диспетчер выбирает очередь.

Статусы: `pending` → `sending` → `sent` | `failed` | `cancelled`.

## Переменные окружения

Шаблон — `.env.example`, полный разбор — в
[CONFIG.md](../../!gateway/notification-service/CONFIG.md). Ключевое:

| Переменная | Назначение |
|---|---|
| `PORT` | по умолчанию `80` |
| `SERVICE_API_KEYS` | персональные ключи `name:key,…` |
| `INTERNAL_API_KEY` | легаси общий ключ; должен совпадать с auth-service |
| `DB_HOST/PORT/USER/PASSWORD/NAME/SSLMODE` | PostgreSQL |
| `SMTP_HOST/PORT/USERNAME/PASSWORD/FROM/USE_TLS/USE_AUTH/AUTH_METHOD/DEBUG` | почта (перекрываются записью в БД) |
| `NOTIFICATION_BOT_URL` | `http://notification-bot:80` |
| `AUTH_SERVICE_URL` | резолв получателей и легаси username → chat_id |
| `AUTH_SERVICE_TIMEOUT_MS` | 5000; резолв стоит на пути входящего запроса |
| `SERVICE_PRIORITIES` | общие потолки приоритета, `name=число` |
| `SERVICE_AUTH_KEY_MAP` | «имя ключа = service_key», пусто справа = не проверять доступ |
| `GUARD_*` | пороги детектора аномалий |
| `SECURITY_ALERT_EMAIL` | фоллбек-получатель алертов |
| `LOG_FULL_RECIPIENTS` | `true` снимает маскировку адресов в логах (`d.***@gh.uz`) — только на время разбора инцидента |
| `TELEGRAM_ENABLED`, `TELEGRAM_SYSTEM_ENABLED` | включение Telegram-каналов |
| `SYSTEM_EMAIL_RECIPIENT`, `SYSTEM_TELEGRAM_USERNAME`, `SEND_SYSTEM_*` | системные уведомления |

`TELEGRAM_BOT_TOKEN` / `TELEGRAM_SYSTEM_BOT_TOKEN` **устарели**: в Telegram
ходит только notification-bot.

## Грабли

- **Порт 8082 из старой документации не используется.** Сервис слушает `:80`
  и не публикует портов; обращаться по `http://notification-service:80`.
- **Сервис поднимается отдельным compose'ом** (`notification-service/`), а не
  общим `!gateway/docker-compose.yaml`.
- **Новый отправитель должен быть зарегистрирован дважды**: ключ в
  `SERVICE_API_KEYS` здесь и `NOTIFICATION_API_KEY` у самого сервиса.
  Ключ надо завести **до** выдачи его сервису, иначе первые запросы получат
  `401` и попадут в счётчик guard'а.
- **`POST /api/v1/security/alert` не существует**, хотя `watchdog.sh` в него
  пишет. См. [guard-watchdog.md](guard-watchdog.md).
