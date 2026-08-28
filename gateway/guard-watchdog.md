# guard-watchdog

Авто-карантин контейнеров по **бинарным индикаторам компрометации** —
событиям, которых в нормальной работе не бывает никогда.

> **Статус: не запущен.** Образ и скрипт лежат в `watchdog/`, но сервиса нет
> ни в `!gateway/docker-compose.yaml`, ни в отдельном compose-файле, и
> `start_all.sh` его не поднимает. Ниже — как он устроен и что нужно, чтобы
> его включить.

- **Каталог:** `watchdog/`
- **Образ:** `docker:27-cli` + `bash`, `curl`, `jq`, `coreutils`
- **Портов не слушает вообще** — нулевая входящая поверхность атаки
- **Требует настоящий `/var/run/docker.sock`** (не прокси): карантин — это и
  есть точка принуждения

## Два уровня реакции

Система защиты разделена по обратимости:

| Уровень | Где | Реакция | Обратима |
|---|---|---|---|
| поведенческие пороги | `security.go` в [notification-service](notification-service.md) | `429`, временный бан IP | да |
| бинарные индикаторы | этот watchdog | карантин контейнера | нет, снимается вручную |

Поведенческий порог может сработать ложно (массовая рассылка после простоя,
ретраи, рассинхрон ключей при деплое) — поэтому там только обратимые меры.
Бинарный индикатор ложно сработать не должен, поэтому здесь реакция жёсткая.

## Индикаторы

1. **Контейнер не из allowlist обратился к docker-socket-proxy.**
   Источник — haproxy-логи `gateway-docker-proxy`, клиентский IP стоит в
   начале строки. IP разрешается в имя контейнера через
   `docker network inspect`. Разрешены `auth-service` и `dozzle`
   (`WATCHDOG_PROXY_ALLOWED`); разрешённые IP кэшируются на 60 с.
2. **Срабатывание ханипота в notification-service.**
   Источник — строка `GUARD TRIPWIRE: ip=…` в его логах
   (пути `/.env`, `/api/v1/internal/keys`, `/api/v1/admin/exec`,
   `/api/v1/debug/pprof`).

Оба монитора работают параллельно через `docker logs -f` и переподключаются
через 5 с, если поток оборвался.

## Что происходит при инциденте

```
индикатор
   │
   ├─ кулдаун по контейнеру (WATCHDOG_INCIDENT_COOLDOWN, 900 с) → пропуск
   │
   ├─ контейнер в WATCHDOG_PROTECTED?
   │     ДА  → только алерт «нужен ручной карантин»
   │     НЕТ ↓
   │
   ├─ WATCHDOG_DRY_RUN=true → только алерт «[DRY-RUN]»
   │
   └─ scripts/quarantine.sh <container>
         1. сохранить логи и сетевую конфигурацию (улики)
         2. отключить от ВСЕХ docker-сетей
         3. docker pause  (память сохранена для форензики)
         + алерт с инструкцией восстановления
```

Контейнер **не удаляется и не останавливается** — улики сохраняются.

### Почему core-инфраструктура защищена

`WATCHDOG_PROTECTED` по умолчанию: `nginx`, `auth-service`, `mongo`,
`docker-socket-proxy`, `dozzle`, `monitoring-service`, `notification-service`,
`notification-postgres`, `guard-watchdog`.

Авто-pause этих контейнеров — готовый DoS-рычаг для атакующего и полный отказ
платформы при ложном срабатывании. Для них только алерт с текстом
«проверьте немедленно» и командой ручного карантина.

## Снятие карантина

```bash
./scripts/quarantine.sh --release <container>
# сети автоматически НЕ восстанавливаются:
cd <директория сервиса> && docker compose up -d --force-recreate <container>
```

Улики по умолчанию складываются в `/var/log/quarantine`
(`QUARANTINE_EVIDENCE_DIR`).

## Переменные окружения

| Переменная | По умолчанию | Назначение |
|---|---|---|
| `WATCHDOG_PROXY_CONTAINER` | `gateway-docker-proxy` | чьи логи слушать (индикатор 1) |
| `WATCHDOG_PROXY_ALLOWED` | `auth-service,dozzle` | кому доступ к Docker API положен |
| `WATCHDOG_PROTECTED` | см. выше | никогда не карантинить автоматически |
| `WATCHDOG_TRIPWIRE_SERVICE` | `notification-service` | чьи логи слушать (индикатор 2) |
| `WATCHDOG_DRY_RUN` | `false` | `true` — только алерты, обкатка |
| `WATCHDOG_NOTIFY_URL` | `http://notification-service:80/api/v1/security/alert` | куда слать алерты |
| `WATCHDOG_API_KEY` | пусто | без него алерт идёт только в лог |
| `WATCHDOG_INCIDENT_COOLDOWN` | `900` | не обрабатывать тот же контейнер чаще, сек |
| `WATCHDOG_QUARANTINE_SCRIPT` | `/scripts/quarantine.sh` | путь к скрипту внутри контейнера |
| `QUARANTINE_EVIDENCE_DIR` | `/var/log/quarantine` | куда складывать улики |

## Что нужно, чтобы включить

1. **Реализовать `POST /api/v1/security/alert` в notification-service.**
   Сейчас такого роута нет — алерты уйдут в `404`, и watchdog запишет
   «Не удалось отправить алерт». Ожидаемое тело:
   `{"subject": "...", "content": "..."}`, заголовок `X-API-Key`.
2. **Добавить сервис в compose** с монтированием
   `/var/run/docker.sock:/var/run/docker.sock` (на запись — нужен `pause` и
   `network disconnect`) и `./scripts:/scripts:ro`.
3. **Обкатать с `WATCHDOG_DRY_RUN=true`**, пока логи не покажут отсутствие
   ложных срабатываний.
4. Завести ключ `watchdog` в `SERVICE_API_KEYS` notification-service и
   передать его как `WATCHDOG_API_KEY`.

## Грабли

- **Watchdog держит полный docker.sock.** Его собственная компрометация
  эквивалентна компрометации хоста. Отсюда требование «ни одного слушающего
  порта»: наружу он только ходит сам.
- **`quarantine.sh` запускается на хосте или из контейнера с сокетом** — ему
  нужны полные права на docker.
- **Разрешение IP в имя контейнера — по всем сетям.** Если контейнер уже
  отключён от сетей, `resolve_ip` вернёт пустоту, и придёт алерт
  «неопознанный клиент».
