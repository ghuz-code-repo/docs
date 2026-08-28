# Уведомления в Telegram через notification-service

Пошаговый гайд: как конечному сервису (пример — `apartment_finder`) слать сообщения в Telegram
со ссылками, кнопками и обратной связью, **не заводя собственного бота**.

Гайд состоит из двух частей:

* **Часть A (шаги 1–4)** — работает на текущем коде, ничего допиливать не надо.
* **Часть B (шаги 5–8)** — что именно нужно доработать в `notification-service`, `notification-bot`
  и `apartment_finder`, чтобы появились кнопки и обратная связь. Кода здесь нет — только
  описание изменений и контракты.

---

## 0. Кто есть кто

```
apartment_finder ──X-API-Key──► notification-service ──X-API-Key──► notification-bot ──► Telegram API
   (Flask)                        (Go, очередь+ретраи)               (Go, long polling)      ▲
                                                                            │               │
                                       auth-service ◄── адрес получателя ───┘               │
                                                                            └── getUpdates ─┘
```

| Компонент | Роль | Где |
|---|---|---|
| `notification-service` | очередь, ретраи, статусы, throttle, единая точка входа для всех сервисов | [!gateway/notification-service/main.go](../../!gateway/notification-service/main.go) |
| `notification-bot` | **единственный**, кто вызывает Telegram API (бот `@notification_analytics_gh_uz_bot`) | [!gateway/notification-bot/main.go](../../!gateway/notification-bot/main.go) |
| `auth-service` | единственный источник адресов доставки: резолвит логин портала → chat_id/email/телефон | [!gateway/auth-service/routes/recipient_handlers.go](../../!gateway/auth-service/routes/recipient_handlers.go) |
| конечный сервис | только формирует текст/кнопки и шлёт HTTP-запрос | [apartment_finder/app/services/notification_client.py](../../apartment_finder/app/services/notification_client.py) |

**Железное правило:** второй `getUpdates` с тем же токеном ломает long polling. Никакой сервис,
кроме `notification-bot`, не должен ходить в `api.telegram.org` и не должен хранить токен бота.

---

## 1. Матрица: что уже есть, что надо допилить

| Возможность | Сейчас | Что делать |
|---|---|---|
| Текст сообщения | ✅ `content` | — |
| Заголовок | ✅ `subject` → жирная первая строка | — |
| Markdown-разметка (жирный, курсив, код) | ✅ жёстко `parse_mode: Markdown` | — |
| **Ссылки** `[текст](url)` | ✅ через Markdown | — |
| **Получатель по логину портала** | ✅ поле `login`, резолв через auth-service | — |
| Получатель вне портала | ✅ поле `external_recipient` (email, телефон, chat_id канала, telegram-ник) | — |
| Пачки, ретраи (3 попытки), статусы | ✅ | — |
| **URL-кнопки** | ❌ | шаг 5 (`notification-service` + `notification-bot`) |
| **Callback-кнопки** | ❌ для прикладных сервисов (есть только `login:` для auth-service) | шаг 5 |
| **Обратная связь бот → сервис** | ❌ | шаг 6 |
| Свой `/start`-deep-link у сервиса | ❌ | шаг 7 |
| Картинки / медиа-группы | ❌ | шаг 8 (обходной путь — ссылка на страницу) |

---

# Часть A. Работает уже сегодня

## Шаг 1. Прописать сервис в notification-service

**1.1. Ключ сервиса.** В `.env` у `notification-service` в переменной `SERVICE_API_KEYS` перечисляются
пары `имя:ключ` через запятую:

```env
SERVICE_API_KEYS=referal:KEY1,apartment-finder:KEY2,client-service:KEY3
```

Логика разбора — [main.go:365](../../!gateway/notification-service/main.go#L365). Есть легаси-фоллбек
`INTERNAL_API_KEY` (общий ключ), но для нового сервиса заводите **персональный** ключ: по нему
работает rate-limit и алерты `ServiceGuard`, и в логах видно, кто именно шлёт.

Генерация: `openssl rand -base64 32`.

**1.2. Env конечного сервиса.** В `apartment_finder/.env`:

```env
NOTIFICATION_SERVICE_URL=http://notification-service:80
NOTIFICATION_API_KEY=KEY2
NOTIFICATION_SERVICE_TIMEOUT=10
```

**1.2.1. Маппинг ключа сервиса.** Имя сервиса в `SERVICE_API_KEYS` и его `service_key` в
auth-service — разные вещи: у apartment_finder это `apartment-finder` и `finder`. В `.env`
notification-service сопоставление задаётся явно, иначе проверка доступа получателей не найдёт
ролей ни у кого:

```env
SERVICE_AUTH_KEY_MAP=apartment-finder=finder
```

Имена переменных уже читает [notification_client.py](../../apartment_finder/app/services/notification_client.py) —
`NOTIFICATION_SERVICE_URL`, `NOTIFICATION_API_KEY`/`INTERNAL_API_KEY`, `NOTIFICATION_SERVICE_TIMEOUT`.

**1.3. Сеть.** Контейнер сервиса должен быть в `service_network` — там же живут `notification-service`
и `notification-bot`. У `finder` это уже так ([apartment_finder/docker-compose.yml](../../apartment_finder/docker-compose.yml)).
Выход в интернет (`egress_network`) конечному сервису для Telegram **не нужен** — наружу ходит только бот.

**1.4. Проверка доступности** (из контейнера сервиса):

```bash
docker exec apartment-finder-app wget -qO- http://notification-service:80/api/v1/health
```

## Шаг 2. Определить получателя

Получатель адресуется **логином портала**, а не адресом доставки. Фактический адрес
(`chat_id`, email, телефон) определяет auth-service в момент приёма запроса.

```json
{"type": "telegram", "login": "ivanov", "content": "..."}
```

Кому уведомление уйдёт по факту, решает привязка Telegram в личном кабинете портала
(Личный кабинет → «Безопасность» → «Подключить Telegram»). Сервису знать `chat_id`
не нужно и хранить его у себя не нужно: ник в Telegram пользователь меняет в один клик,
логин портала — нет.

**Получатель вне системы** (клиент, внешний подрядчик, канал Telegram) — отдельное поле:

```json
{"type": "email", "external_recipient": "client@mail.ru", "content": "..."}
```

Правила:

* Заполнено **ровно одно** из `login` / `external_recipient`. Оба сразу или ни одного — `400`.
* Угадывания нет: email в поле `login` — тоже `400`, а не «наверное, это внешний адрес».
  Ошибка адресации должна падать громко, а не молча уводить уведомление не туда.
* `telegram_system` — исключение: получателя можно не указывать, берётся из конфига сервиса.
* **Канал/группа.** Чтобы слать в канал, добавьте `@notification_analytics_gh_uz_bot`
  администратором канала и укажите его `chat_id` в `external_recipient` (для канала он
  отрицательный, вида `-100...`). Старый приватный канал `TELEGRAM_CHANNEL_ID` из
  `apartment_finder/.env` заработает только после добавления бота портала.

**Что происходит внутри.** notification-service зовёт у auth-service
`POST /api/recipients/resolve` — пачкой на всю рассылку, результат кэшируется на 10 минут
(отрицательный — на минуту). Матч идёт строго по `username`, поэтому в `login` нужен именно
логин портала, не email и не telegram-ник.

**Проверка доступа.** auth-service заодно смотрит, есть ли у получателя роли в
сервисе-отправителе. По умолчанию это только запись в лог (`RECIPIENT ACCESS`); жёсткая
блокировка включается флагом `RECIPIENT_ACCESS_ENFORCE=true` в auth-service — и только после
того, как логи покажут, что легитимных рассылок это не режет.

⚠️ Имя сервиса в `SERVICE_API_KEYS` и его `service_key` в auth-service **могут не совпадать**:
у apartment_finder это `apartment-finder` и `finder` соответственно. Сопоставление задаётся
в notification-service переменной `SERVICE_AUTH_KEY_MAP=apartment-finder=finder`. Без неё
проверка доступа не найдёт ролей ни у кого из получателей сервиса.

**Обработка отказа.** Резолв идёт на приёме запроса, поэтому отказ приходит сразу, синхронно,
машинным кодом в поле `failure_code` — а не оседает в очереди и всплывает потом в `last_error`:

| Ответ | `failure_code` | Что делать сервису |
|---|---|---|
| `400` | `user_not_found` | логина нет на портале — проверить свой справочник получателей |
| `400` | `user_banned` | пользователь заблокирован — не слать |
| `400` | `channel_not_linked` | Telegram не привязан либо бот заблокирован — fallback на email, в UI показать «Telegram не подключён» со ссылкой на личный кабинет |
| `400` | `no_address` | у пользователя не заполнен email/телефон |
| `400` | `no_service_access` | у получателя нет ролей в вашем сервисе |
| `503` | `auth_unavailable` | auth-service недоступен — повторить позже, **уведомление не создано** |

В пачке отказ по одному получателю не рушит остальные: такая запись создаётся сразу в статусе
`failed` с `failure_code`, а список проблемных логинов возвращается в ответе полем `unresolved`.

**Если пользователь заблокировал бота.** Telegram отвечает «bot was blocked by the user», и
notification-service сбрасывает привязку в auth-service, а свой кэш чистит. Дальше уведомления
этому получателю отваливаются на приёме с `channel_not_linked`, не доходя до Telegram.

Смысл в скорости: пачка обрабатывается последовательно, и каждая отправка заблокировавшему
бота тратит полный круг до `api.telegram.org`, задерживая всех остальных получателей пачки.
Один сброс привязки убирает эту задержку навсегда — до тех пор, пока пользователь не подключит
бота заново в личном кабинете.

**Что обязан делать вызывающий сервис:** увидев `channel_not_linked`, прекратить попытки по
Telegram для этого получателя. Повторять по расписанию бессмысленно — привязки больше нет.
Правильная реакция: отправить по email и показать в интерфейсе, что Telegram отключён.

Старое поле `recipient` из API удалено: запрос с ним отвергается как «не указан получатель».

### Кто на новом контракте

| Сервис | Чем адресует |
|---|---|
| `referal` | `login` получателей из auth-service и своей таблицы пользователей; служебные адреса — через `*_LOGIN` / `*_EMAIL` |
| `client_service` | `login` ответственного (резолв по `gateway_user_id`); если привязки нет — письмо уходит на `UNLINKED_RESPONSIBLE_EMAIL` с пометкой `[НЕТ ПРИВЯЗКИ]` |
| `auth-service` | `login` пользователя; письмо об удалении аккаунта и адреса из `ADMIN_EMAIL` — `external_recipient` |
| `monitoring-service` | `SYSTEM_ALERT_LOGIN`, иначе адреса из конфига через `external_recipient` |
| `apartment_finder` | `login` получателей |

Что ещё стоит проверить в `apartment_finder`: справочник получателей
(`/admin/notifications`, см. [settings_routes.py:128](../../apartment_finder/app/web/settings_routes.py#L128))
и подписки на уведомления о дебиторке
([debt_reminder_service.py:118](../../apartment_finder/app/services/debt_reminder_service.py#L118))
должны хранить **логины портала**, а не telegram-ники и chat_id. Канал Telegram
(`TELEGRAM_CHANNEL_ID`) — это `external_recipient`, а не `login`.

## Шаг 3. Отправить сообщение

Одно уведомление:

```http
POST http://notification-service:80/api/v1/notifications
Content-Type: application/json
X-API-Key: KEY2

{
  "type": "telegram",
  "login": "ivanov",
  "subject": "🏢 Новая квартира в подборке",
  "content": "ЖК *Golden House*\nПлощадь: 62.4 м²\nЦена: 1 240 у.е./м²"
}
```

Ответ: `{"id": 42, "status": "pending", "message": "..."}`.
Отказ резолва — синхронный `400`/`503` с `failure_code` (таблица в шаге 2).

Пачка (один запрос — много получателей):

```http
POST /api/v1/notifications/batch
{ "batch_id": "finder-daily-2026-08-25", "notifications": [ {...}, {...} ] }
```

Вся пачка резолвится одним запросом к auth-service (до 500 логинов), а нерезолвнутые
получатели возвращаются в ответе полем `unresolved` и создаются сразу в статусе `failed`.

Статусы: `GET /api/v1/notifications/42`, `GET /api/v1/batches/<batch_id>` →
`pending` → `sending` → `sent` | `failed` (+ `attempts`, `last_error`, `failure_code`).
В карточке уведомления `recipient` — фактический адрес доставки, `recipient_login` — логин,
если адресовали по нему.

Типы:

* `telegram` — прикладные уведомления пользователям. **Это то, что нужно `apartment_finder`.**
* `telegram_system` — системные алерты; получатель подменяется на `system_telegram_chat_id` из
  конфига, поэтому поля получателя можно не заполнять. Прикладным сервисам не использовать.

Что происходит с сообщением внутри:

* `subject` склеивается с телом как `*subject*\n\ncontent` — отдельного «заголовка» в Telegram нет.
* `parse_mode` жёстко зашит как `Markdown` (легаси, не MarkdownV2).
* Ретраи: `max_attempts` канала telegram (3) с экспоненциальным backoff от 2 с.
  Отказ по темпу (429) считается отдельным счётчиком и попытки доставки не съедает.
* Очередь у канала своя: telegram и telegram_system делят одну (общий бот, общие
  лимиты), почта — отдельную. Медленный SMTP телеграм не тормозит.
* Темп соблюдается по лимитам Telegram: 30 сообщений/с на бота, 1/с в личный чат,
  20/мин в группу. Настраивается в `POST /api/v1/channels/telegram`.
* Сообщение, упёршееся в лимит чата, откладывается в очередь и не задерживает
  сообщения другим получателям.
* Флаги `telegram_enabled` / `telegram_system_enabled` **больше не гейтят** отправку.

**Как встроить в код `apartment_finder`:** в `notification_client.py` добавляется метод-близнец
`send_email` — та же схема, только `type: "telegram"`, `login` = логин портала (или
`external_recipient` для получателя вне системы), без `content_type`. Ничего больше в сервисе
для отправки не нужно.

## Шаг 4. Ссылки и форматирование

Ссылка — обычный Markdown в `content`:

```
[Открыть карточку квартиры](https://analytics.gh.uz/finder/apartments/1234)
```

Правила:

* URL пишем **абсолютный, с публичным доменом портала**. Сервис сидит за nginx под префиксом `/finder`
  ([prefix_middleware.py](../../apartment_finder/prefix_middleware.py)), внутренние адреса вида
  `http://apartment-finder-app/...` из Telegram недоступны. Заведите в конфиге `PUBLIC_BASE_URL`
  и стройте ссылки от него, а не через `url_for(..., _external=True)`.
* Ссылка ведёт на страницу под авторизацией портала — это нормально: пользователь перейдёт,
  gateway попросит логин и вернёт на нужный URL.
* Легаси-Markdown ломается на несбалансированных `*`, `_`, `` ` ``, `[`. **Любой подставляемый текст
  (название ЖК, ФИО, комментарий) нужно экранировать.** Заведите одну функцию-экранировщик и
  прогоняйте через неё все динамические куски — иначе сообщение молча уйдёт в `failed` с
  `can't parse entities`.
* Эмодзи, переносы `\n` — работают как есть.

Если разметка мешает (много спецсимволов в данных) — это аргумент за доработку из шага 5,
где `parse_mode` становится настраиваемым (`HTML` экранируется втрое проще: только `&`, `<`, `>`).

---

# Часть B. Доработки

## Шаг 5. Кнопки: проброс через notification-service

Сейчас `notification-bot` кнопки **уже умеет** (`buttons` в `POST /api/v1/send`), но
`notification-service` их не принимает и не передаёт. Нужны три правки.

### 5.1. notification-bot: добавить URL-кнопки

[telegram.go:28](../../!gateway/notification-bot/telegram.go#L28) — структура `InlineButton` содержит только
`text` + `callback_data`. Для кнопки-ссылки нужно поле `url` (Telegram требует, чтобы у кнопки был
ровно один из `url` / `callback_data`).

* Добавить в `InlineButton` поле `url` с `omitempty`.
* В обработчике `POST /api/v1/send` ([main.go](../../!gateway/notification-bot/main.go)) валидировать:
  у каждой кнопки задан ровно один из двух атрибутов, `callback_data` ≤ 64 байт, разумный лимит рядов.

### 5.2. notification-service: новые поля уведомления

В модели `Notification` ([main.go:75](../../!gateway/notification-service/main.go#L75)) добавить:

| Поле | Тип | Смысл |
|---|---|---|
| `parse_mode` | строка | `Markdown` (дефолт, обратная совместимость) / `MarkdownV2` / `HTML` |
| `buttons` | JSON-строка в БД, массив массивов в API | инлайн-клавиатура, как у бота |

Замечания:

* В БД (Postgres, GORM) кнопки хранить сырым JSON-текстом — отдельная таблица не нужна.
  Поля должны быть nullable, чтобы существующие записи и вызывающие сервисы не сломались.
* В `sendTelegram` ([processors.go:452](../../!gateway/notification-service/processors.go#L452)) payload
  для бота собирается вручную — туда добавить `parse_mode` (из уведомления, дефолт `Markdown`)
  и `buttons` (если непусто).
* Валидацию делать **на приёме запроса**, а не при отправке: битые кнопки должны отдавать `400`
  сразу, а не оседать в очереди и падать в ретраях.
* Для `email`/`sms` поля игнорируются.

### 5.3. Контракт после доработки

```json
{
  "type": "telegram",
  "login": "ivanov",
  "subject": "🏢 Новая квартира в подборке",
  "parse_mode": "HTML",
  "content": "ЖК <b>Golden House</b>, 62.4 м²",
  "buttons": [
    [ {"text": "Открыть карточку", "url": "https://analytics.gh.uz/finder/apartments/1234"} ],
    [ {"text": "👍 Интересно", "callback_data": "af:like:1234"},
      {"text": "🚫 Скрыть",   "callback_data": "af:hide:1234"} ]
  ]
}
```

**Практический совет:** 80–90 % сценариев закрываются URL-кнопками. Они не требуют шага 6 вообще —
пользователь уходит на страницу сервиса, где уже есть авторизация, права и вся логика.
Callback-кнопки берите только там, где действие обязано выполняться, не выходя из Telegram.

## Шаг 6. Обратная связь: callback от бота в конечный сервис

Сегодня [updates.go:72](../../!gateway/notification-bot/updates.go#L72) знает ровно один формат
`login:<request_id>:<approve|reject>` и ходит только в auth-service. Нужен обобщённый роутер.

### 6.1. Формат callback_data

Первый сегмент — **ключ сервиса**, дальше — его личное дело:

```
<service_key>:<action>:<payload>
af:like:1234
af:hide:1234
```

Ограничение Telegram — 64 байта на весь `callback_data`. Длинные payload (UUID, JSON) не влезают:
класть короткий идентификатор записи, остальное сервис достаёт у себя. Значение `login` остаётся
зарезервированным за auth-service.

### 6.2. Таблица маршрутов в notification-bot

Env-переменная со списком «ключ = URL вебхука», например:

```env
CALLBACK_ROUTES=af=http://apartment-finder-app:80/api/telegram/callback,ref=http://referal-app:80/api/telegram/callback
```

* Неизвестный префикс → отвечать пользователю «Неизвестное действие» (как сейчас), в лог — WARN.
* Хождение по внутренним DNS-именам контейнеров в `service_network`; наружу такие URL не публикуются.
* Таймаут на вызов сервиса — 5–10 с; при недоступности показать пользователю
  «Сервис временно недоступен, попробуйте позже» и **не гасить кнопки**, чтобы можно было повторить.

### 6.3. Контракт вебхука (бот → сервис)

Запрос:

```http
POST /api/telegram/callback
X-API-Key: <ключ этого сервиса>
Content-Type: application/json

{
  "service": "af",
  "action": "like",
  "payload": "1234",
  "raw_data": "af:like:1234",
  "chat_id": 123456789,
  "telegram_username": "ivanov",
  "message_id": 55,
  "callback_id": "409...",
  "sent_at": 1756100000
}
```

Ответ:

```json
{
  "toast": "Добавлено в избранное",
  "edit_text": "👍 Квартира 1234 добавлена в избранное",
  "buttons": []
}
```

* `toast` → `answerCallbackQuery` (всплывашка вверху экрана, ≤200 символов).
  **Отвечать на callback обязательно**, иначе у пользователя кнопка «крутится» ~30 секунд.
* `edit_text` (опционально) → `editMessageText`: заменить текст сообщения на результат.
  Именно так гасятся кнопки, чтобы их нельзя было нажать дважды — приём уже используется
  в сценарии входа ([updates.go:99](../../!gateway/notification-bot/updates.go#L99)).
* `buttons` (опционально) → новая клавиатура; пустой массив — убрать кнопки.
  Потребует у бота метод правки клавиатуры (сейчас `EditMessageText` её просто сносит).

### 6.4. Безопасность вебхука

* Бот аутентифицируется у сервиса **своим X-API-Key** (тот же per-service ключ, что и в шаге 1,
  либо отдельный `TELEGRAM_CALLBACK_API_KEY`). Ключ у каждого сервиса свой — общий `INTERNAL_API_KEY`
  здесь только легаси-фоллбек.
* Эндпоинт вебхука в конечном сервисе **вне пользовательской авторизации** (`AuthMiddleware` его
  пропускает), поэтому проверка ключа — единственный барьер: без неё это открытый эндпоинт «сделай
  действие за пользователя».
* Личность пользователя определяется **только по `chat_id`**, а не по `telegram_username` из
  апдейта: ник в Telegram меняется в один клик, chat_id — нет. Маппинг chat_id → пользователь
  портала берётся у auth-service (или из локального справочника получателей из шага 2).
* Право на действие проверяет **сервис**, а не бот: «этот пользователь действительно может скрыть
  эту квартиру». `callback_data` приходит из клиента Telegram и подделывается тривиально —
  относиться к нему как к пользовательскому вводу.
* Идемпотентность: одна и та же кнопка может прилететь дважды (двойное нажатие, ретрай). Действие
  должно быть безопасно повторяемым либо помечаться как выполненное по `payload` + `chat_id`.

### 6.5. Что добавить в apartment_finder

* Роут `POST /api/telegram/callback` в отдельном blueprint (рядом с прочими служебными роутами),
  исключённый из auth-middleware и из CSRF.
* Проверка `X-API-Key` до любой другой логики; при несовпадении — `401` и запись в лог.
* Разбор `action` через явный whitelist-словарь «действие → обработчик»; всё незнакомое — `400`.
* Обработчик возвращает `toast`/`edit_text` — весь текст для пользователя формирует сервис,
  бот ничего про предметную область не знает.

## Шаг 7. Свои deep-link `/start` (если нужны подписки без портала)

Нужен только если получатели — люди **без аккаунта на портале** (например, клиенты-подписчики).
Сегодня `/start <token>` жёстко трактуется как токен привязки auth-service
([updates.go:36](../../!gateway/notification-bot/updates.go#L36)).

Что менять: префиксовать payload deep-link ключом сервиса (`/start af_<token>`) и в обработчике
маршрутизировать по тому же `CALLBACK_ROUTES` (`POST .../telegram/start` с токеном и chat_id).
Токен генерирует сам сервис: одноразовый, короткоживущий (≤15 мин), доставляется пользователю по
уже доверенному каналу (email/личный кабинет). Без префикса — прежнее поведение auth-service.

Если все получатели — пользователи портала, шаг 7 **не нужен**: привязка уже сделана в личном
кабинете, chat_id берётся из auth-service.

## Шаг 8. Убрать собственного бота из apartment_finder

Что сейчас в сервисе своё:

| Место | Что делает | Куда девать |
|---|---|---|
| [news_service.py:41](../../apartment_finder/app/services/news_service.py#L41) `send_to_telegram` | шлёт новость в канал напрямую через `api.telegram.org` (`sendMessage` / `sendMediaGroup`) | заменить на вызов notification-service (см. ниже про медиа) |
| [config.py:24-25](../../apartment_finder/app/core/config.py#L24) `TELEGRAM_BOT_TOKEN`, `TELEGRAM_CHANNEL_ID` | токен своего бота и id канала | токен — выпилить; id канала слать в `external_recipient` |
| [auth_utils.py:9](../../apartment_finder/app/core/auth_utils.py#L9) `verify_telegram_data` | валидация подписи Telegram Mini App **тем же токеном** | ⚠️ см. предупреждение ниже |
| [decorators.py:154](../../apartment_finder/app/core/decorators.py#L154) | проверка `X-Telegram-Init-Data` для TMA | зависит от предыдущего пункта |

**⚠️ Главная ловушка.** Токен используется не только для отправки, но и для проверки подписи
Telegram Mini App. Если просто удалить `TELEGRAM_BOT_TOKEN` из `.env` — TMA-авторизация молча
перестанет пускать пользователей. Варианты:

1. **Минимальный.** Удалить только отправку (`send_to_telegram`), токен оставить исключительно для
   валидации `init_data`. Требование гайда соблюдено: Telegram API сервис больше не дёргает.
2. **Чистый.** Перевести Mini App на бота портала и добавить в `notification-bot` эндпоинт
   `POST /api/v1/validate-init-data` (принимает `init_data`, возвращает `valid` + распарсенного
   пользователя). Тогда токен живёт ровно в одном месте — в `notification-bot/.env`.
   Учтите: сменится бот, внутри которого открывается Mini App, — старые ссылки на TMA перестанут
   работать, нужен `menu_button`/deep-link у бота портала.

**Медиа.** `notification-service` умеет только текст: `sendMediaGroup` через него не отправить.
Для новостей с фото:

* сейчас — слать текст + Markdown-ссылку «Смотреть на портале» (изображение подтянет превью
  по Open Graph, если страница отдаёт метатеги);
* при необходимости картинок в самом сообщении — отдельная доработка: поле `media` (список URL,
  доступных боту) в контракте `notification-service` → `notification-bot` → `sendPhoto`/`sendMediaGroup`.
  Учтите изолированную сеть: URL картинок должны быть достижимы для контейнера бота
  (он в `egress_network`, а внутренние адреса портала видит через `service_network`).

**После доработки** удалить из `.env` и `.env.example` строки `TELEGRAM_BOT_TOKEN` /
`TELEGRAM_CHANNEL_ID` (либо оставить только то, что реально требуется по выбранному варианту),
и **обязательно отозвать старый токен** через `@BotFather` (`/revoke`) — он лежал в репозитории
в открытом виде.

---

## 9. Чеклист внедрения

Часть A (сразу, без доработок):

1. [ ] Сгенерирован ключ, добавлен в `SERVICE_API_KEYS` у `notification-service`, сервис перезапущен.
2. [ ] `NOTIFICATION_SERVICE_URL` / `NOTIFICATION_API_KEY` прописаны в `.env` сервиса.
3. [ ] `/api/v1/health` отвечает из контейнера сервиса.
4. [ ] Решено, как берётся `chat_id` (справочник в сервисе или резолв через auth-service).
5. [ ] В `notification_client.py` добавлен метод отправки `type: "telegram"`.
6. [ ] Все динамические подстановки в тексте экранируются под Markdown.
7. [ ] Ссылки строятся от публичного `PUBLIC_BASE_URL`, а не от внутреннего адреса.
8. [ ] Есть fallback на email, если получатель не привязал Telegram.

Часть B (кнопки и обратная связь):

9. [ ] `InlineButton` в `notification-bot` умеет `url`; валидация «ровно одно из url/callback_data».
10. [ ] `Notification` в `notification-service` умеет `parse_mode` и `buttons`, поля пробрасываются в бот.
11. [ ] В `notification-bot` появился роутер callback по префиксу + `CALLBACK_ROUTES`.
12. [ ] `answerCallbackQuery` вызывается всегда, в том числе на ошибках.
13. [ ] В сервисе есть `POST /api/telegram/callback`: проверка X-API-Key, whitelist действий,
    идентификация по `chat_id`, проверка прав, идемпотентность.
14. [ ] Кнопки гасятся через `edit_text` после первого нажатия.
15. [ ] Из `apartment_finder` удалены прямые вызовы `api.telegram.org`; судьба TMA-токена решена;
    старый токен отозван в `@BotFather`.

## 10. Грабли

* **Два потребителя `getUpdates`** — если хоть один сервис снова начнёт опрашивать Telegram тем же
  токеном, апдейты начнут «делиться» между процессами и половина кнопок перестанет работать.
* **Кнопки в каналах.** В канале callback приходит от конкретного пользователя, а не «от канала»:
  проверять права по `chat_id` нажавшего, а не по чату сообщения.
* **64 байта на `callback_data`.** UUID + действие уже почти упираются в лимит — использовать
  короткие числовые id.
* **Легаси-Markdown.** Одиночный `_` в названии или `*` в тексте роняет всё сообщение целиком.
* **Тихие `failed`.** Уведомление уходит в очередь и падает асинхронно: без чтения
  `GET /api/v1/notifications/<id>` или логов сервис думает, что всё отправилось.
  Для важных рассылок логировать возвращённый `id` и проверять статус.
* **Ключи.** У `notification-service` есть внешний IP-whitelist (внутренние docker-сети пускаются
  и без ключа) — не полагайтесь на него как на защиту, ключ ставьте всегда.
* **Rate limit Telegram** — 30 сообщений/с суммарно, 1/с в один чат, 20/мин в одну группу.
  Соблюдаются автоматически: `notification-bot` выдерживает темп сам (он единственная точка,
  через которую идёт весь трафик бота), а `notification-service` дополнительно расставляет
  свою очередь. Отдельных действий от вызывающего сервиса не требуется — массовую рассылку
  просто отправить пачкой. Уведомления сверх темпа не теряются, а откладываются в очереди:
  для рассылки в сотни адресов это означает растянутую доставку, а не отказ.

## 11. Ссылки

* [!gateway/notification-bot/README.md](../../!gateway/notification-bot/README.md) — архитектура бота, привязка, вход через Telegram
* [!gateway/notification-service/TELEGRAM_GUIDE.md](../../!gateway/notification-service/TELEGRAM_GUIDE.md) — текущий контракт telegram-уведомлений
* [!gateway/notification-service/CONFIG.md](../../!gateway/notification-service/CONFIG.md) — переменные конфигурации
