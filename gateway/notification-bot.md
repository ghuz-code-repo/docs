# notification-bot

Telegram-бот портала ([@notification_analytics_gh_uz_bot](https://t.me/notification_analytics_gh_uz_bot))
и **единственная точка обращения к Telegram Bot API** во всей системе.

- **Каталог:** `notification-bot/`
- **Технология:** Go, Gin, long polling (`getUpdates`)
- **Контейнер:** `gateway-notification-bot`, слушает `:80`
- **Сети:** `service_network` (приём запросов и вызовы auth-service),
  `egress_network` (выход в Telegram API)
- **Healthcheck:** `GET /health`
- **Состояние:** нет. Токены привязки, запросы входа и счётчики отклонений
  лежат в MongoDB auth-service.

Полное описание с примерами — [notification-bot/README.md](../../!gateway/notification-bot/README.md).
Здесь — место бота в архитектуре шлюза.

## Четыре сценария

1. **Системные уведомления.** notification-service пересылает сюда типы
   `telegram` и `telegram_system` (алерты мониторинга, security-алерты).
2. **Привязка Telegram к аккаунту.** Пользователь в личном кабинете вводит
   свой username, получает на почту deep-link
   `https://t.me/notification_analytics_gh_uz_bot?start=<токен>`; переход по
   ссылке подтверждает привязку.
3. **Вход через Telegram.** На `/login/telegram` бот присылает сообщение с
   кнопками «Подтвердить вход» / «Отклонить вход».
4. **Сброс пароля через Telegram.** На `/forgot-password` можно выбрать
   доставку ссылки в Telegram вместо email.

```
Браузер ──► nginx ──► auth-service ─────► notification-bot ──► Telegram API
                          ▲                    ▲   │ (long polling)
                          │                    │   │
                          │    notification-service (telegram, telegram_system)
                          │                        │
                          └────────────────────────┘
                        /api/telegram/* (X-API-Key)
```

**Никакой другой сервис не должен вызывать `getUpdates` с этим токеном** —
long polling двух потребителей конфликтует, и обновления начнут теряться.
Вебхук и открытые наружу порты боту не нужны.

## API

### `POST /api/v1/send`

Требует `X-API-Key` (равен `INTERNAL_API_KEY` auth-service).

```json
{
  "chat_id": 123456789,
  "text": "Текст сообщения",
  "parse_mode": "Markdown",
  "buttons": [[
    {"text": "✅ Подтвердить вход", "callback_data": "login:<request_id>:approve"},
    {"text": "🚫 Отклонить вход",  "callback_data": "login:<request_id>:reject"}
  ]]
}
```

| Код | Значение |
|---|---|
| `200` | `{"success": true, "message_id": 42}` |
| `429` | темп превышен: `{"error": "...", "retry_after": 5}` + заголовок `Retry-After`. Сообщение **не отправлено** — вернуть его в свою очередь |
| `502` | Telegram отверг сообщение (нет чата, бот заблокирован, битая разметка) |

### `GET /health`

Healthcheck для docker-compose.

## Вызовы в auth-service

| Эндпоинт | Когда |
|---|---|
| `POST /api/telegram/link/confirm` | пришло `/start <токен>` — подтверждение привязки |
| `POST /api/telegram/login/decision` | нажата кнопка подтверждения/отклонения входа |
| `POST /api/telegram/link/broken` | пользователь заблокировал бота |

## Почему лимиты живут здесь

Через одного бота шлют notification-service, auth-service и цикл ответов на
входящие сообщения. Ни один из них не видит трафик остальных — единственная
точка, где считается общий темп, это сам бот.

| Ограничение | Значение | Переменные |
|---|---|---|
| суммарно на бота | 30 сообщений/с | `TELEGRAM_RATE_PER_SECOND`, `TELEGRAM_BURST` |
| в один личный чат | 1 сообщение/с | `TELEGRAM_CHAT_PER_MINUTE`, `TELEGRAM_CHAT_BURST` |
| в одну группу | 20 сообщений/мин | `TELEGRAM_GROUP_PER_MINUTE`, `TELEGRAM_GROUP_BURST` |

Группа отличается от личного чата знаком `chat_id`: у групп, супергрупп и
каналов он отрицательный.

Отправка, укладывающаяся в лимит, ждёт очереди внутри бота (до
`TELEGRAM_MAX_WAIT_MS`). Дольше — вызывающий получает `429` с точным
`retry_after`. Если `429` пришёл от самого Telegram, его `retry_after`
прокидывается без изменений и сдвигает внутреннее окно бота.

Значения по умолчанию равны документированным лимитам Telegram. Занижать
безопасно, **завышать нет**: превышение даёт `429` на весь бот портала,
включая подтверждение входа.

## Переменные окружения

| Переменная | Описание |
|---|---|
| `TELEGRAM_BOT_TOKEN` | токен от @BotFather (обязательно) |
| `INTERNAL_API_KEY` | должен совпадать с auth-service (обязательно) |
| `AUTH_SERVICE_URL` | по умолчанию `http://auth-service:80` |
| `PORT` | по умолчанию `80` |
| `ENVIRONMENT` | `production` отключает debug-логи gin |
| `TELEGRAM_RATE_PER_SECOND`, `TELEGRAM_BURST` | 30 / 30 |
| `TELEGRAM_CHAT_PER_MINUTE`, `TELEGRAM_CHAT_BURST` | 60 / 1 |
| `TELEGRAM_GROUP_PER_MINUTE`, `TELEGRAM_GROUP_BURST` | 20 / 1 |
| `TELEGRAM_MAX_WAIT_MS` | 10000 |

Со стороны auth-service обязательны `NOTIFICATION_BOT_URL=http://notification-bot:80`
и `TELEGRAM_BOT_USERNAME` (без `@`, нужен для deep-link).

## Безопасность

- Токен привязки: 48 hex-символов, одноразовый, живёт 15 минут, доставляется
  только на подтверждённый email пользователя.
- Запрос входа: 64 hex-символа, живёт 3 минуты, потребляется ровно один раз
  (атомарный `FindOneAndUpdate`); решение принимается только с привязанного
  `chat_id`.
- Неизвестные идентификаторы при входе и сбросе получают неотличимый
  «generic» ответ — перебор аккаунтов невозможен.
- ≥3 отклонений входа замораживают вход через Telegram до успешного входа
  по паролю.
- TTL-индексы MongoDB сами удаляют истёкшие токены и запросы.
