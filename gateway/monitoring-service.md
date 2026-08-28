# monitoring-service

Опрашивает health-эндпоинты сервисов и шлёт алерты о падении и восстановлении
через [notification-service](notification-service.md).

- **Каталог:** `monitoring-service/`
- **Технология:** Go 1.23, Gin
- **Контейнер:** `gateway-monitoring-service`, слушает `:80` (`PORT`)
- **Сети:** `service_network` (в общем `docker-compose.yaml`)
- **Healthcheck:** `GET /health`
- **Веб-панель:** `static/index.html`

## Место в системе

Это **не** тот же мониторинг, что health-монитор внутри auth-service.
Они дополняют друг друга:

| | auth-service | monitoring-service |
|---|---|---|
| источник данных | heartbeat из service registry | активный HTTP-опрос health-URL |
| список сервисов | из MongoDB (`services` × `service_instances`) | из `MONITORED_SERVICES` в `.env` |
| что отдаёт | `GET /api/services/health` для карточек в UI | своя панель + `GET /api/v1/status` |
| алерты | по своим `MONITOR_*` настройкам | по `ALERT_*` настройкам |

Соответственно сервис, не умеющий регистрироваться в реестре, всё равно
попадёт под наблюдение — достаточно дописать его в `MONITORED_SERVICES`.

## Два compose-файла

`monitoring-service` описан **дважды**:

- в `!gateway/docker-compose.yaml` — только `service_network`,
  `container_name: gateway-monitoring-service`. **Это рабочий вариант**,
  его поднимает `start_all.sh`;
- в `monitoring-service/docker-compose.yaml` — `service_network` +
  `public_network`, `container_name: monitoring-service`. Для запуска сервиса
  в одиночку.

Оба поднимать не нужно: получатся два контейнера с одинаковой ролью.

## API

Доступ к панели и `/api/v1/*` закрыт `internalAuth`: пропускается запрос
либо с `X-API-Key: INTERNAL_API_KEY`, либо с внутреннего IP (loopback,
`172.16.0.0/12`, `10.0.0.0/8`, `192.168.0.0/16`). Иначе — `403`.

| Метод | Путь | Назначение |
|---|---|---|
| GET | `/` | панель мониторинга (`static/index.html`) |
| GET | `/health` | healthcheck, **без** аутентификации |
| GET | `/api/v1/status` | статусы всех наблюдаемых сервисов |
| GET | `/api/v1/status/:service` | статус одного |
| POST | `/api/v1/check` | принудительная проверка сейчас |

Статус сервиса: `healthy` | `unhealthy` | `unknown`; в записи есть
`last_check`, `last_healthy`, `error_count`, `last_error`, `alerted_down`,
`last_alert_at`.

## Список наблюдаемых

Формат `MONITORED_SERVICES` — `имя:health_url` через запятую:

```
MONITORED_SERVICES=auth-service:http://auth-service:80/health,notification-service:http://notification-service:80/api/v1/health,referral-service:http://referal:80/health,client-service:http://client-service-service:80/health
```

Разбор идёт по **первому** двоеточию, поэтому `http://...:80/...` не ломается.
Строка неверного формата пропускается с предупреждением в лог.

## Логика алертов

Проверка каждые `CHECK_INTERVAL_SECONDS` (30 с). Решение об алерте
принимается под мьютексом, отправка — уже без него (там HTTP).

- **Down-алерт** уходит только после `ALERT_FAILURE_THRESHOLD` (3)
  **подряд** неудачных проверок. Разовый сбой DNS или сети письма не породит.
- Повторные down-алерты по тому же сервису подавляются
  `ALERT_COOLDOWN_MINUTES` (15). Один инцидент = максимум одно письмо.
- **Recovery-алерт** отправляется только если down-алерт по этому инциденту
  реально был отправлен.
- Алерты вообще отключаются, если в конфиге notification-service выключены
  и email-, и Telegram-уведомления (`send_system_*`); конфиг забирается
  по `GET /api/v1/config` и кэшируется.

## Адресация алертов

Приоритет — логин портала:

```
SYSTEM_ALERT_LOGIN=ivanov      # → notification: {"login": "ivanov"}
```

Тогда адрес доставки (и email, и chat_id) держит auth-service, и менять его
в `.env` не нужно. Если логин пуст, работают прямые адреса из конфига
notification-service: `system_email_recipient` для почты и
`system_telegram_username` для Telegram (уходит в `external_recipient`).

Тип для почты — `email`, для Telegram — `telegram_system`. В письме markdown
вырезается (`*` удаляются), в Telegram уходит как есть.

Ключ авторизации: `NOTIFICATION_API_KEY`, фоллбек — `INTERNAL_API_KEY`.
Персональный ключ должен быть заведён в `SERVICE_API_KEYS` notification-service
записью `monitoring-service:…`.

## Переменные окружения

| Переменная | По умолчанию | Назначение |
|---|---|---|
| `PORT` | `80` | порт HTTP |
| `MONITORED_SERVICES` | пусто | что опрашивать; пусто = ничего |
| `CHECK_INTERVAL_SECONDS` | `30` | период опроса |
| `HEALTH_CHECK_TIMEOUT_SECONDS` | `10` | таймаут HTTP-клиента |
| `ALERT_FAILURE_THRESHOLD` | `3` | подряд неудач до down-алерта |
| `ALERT_COOLDOWN_MINUTES` | `15` | подавление повторов |
| `NOTIFICATION_SERVICE_URL` | `http://notification-service:80` | куда слать |
| `NOTIFICATION_API_KEY` | — | персональный ключ |
| `INTERNAL_API_KEY` | — | фоллбек-ключ и доступ к своему API |
| `SYSTEM_ALERT_LOGIN` | пусто | логин дежурного администратора |
| `SYSTEM_EMAIL_RECIPIENT`, `SYSTEM_TELEGRAM_USERNAME` | — | фоллбек-адреса |
| `SEND_SYSTEM_EMAIL_NOTIFICATIONS`, `SEND_SYSTEM_TELEGRAM_NOTIFICATIONS` | `true` | локальные тумблеры |
| `PERSISTENT_ALERT_THRESHOLD`, `ENABLE_PERSISTENT_ALERTS` | `20`, `false` | заготовка повторных напоминаний |
| `GIN_MODE` | — | `release` глушит debug-логи |

## Завершение работы

`SIGINT`/`SIGTERM` останавливают цикл опроса (`context.Cancel`) и дают
HTTP-серверу 10 секунд на graceful shutdown.
