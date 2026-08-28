# docs — документация платформы Golden House

Отдельный репозиторий, подключаемый сабмодулем в
[AnalyticsRepo](https://github.com/ghuz-code-repo) как `docs/`.

Здесь описан **шлюз** (`!gateway`) и **контракт подключения** конечных
сервисов к нему. Бизнес-логики самих сервисов тут нет — она в их
репозиториях.

## Начать отсюда

| Кто вы | Куда |
|---|---|
| ИИ-агент в новой сессии | **[AGENTS.md](AGENTS.md)** — брифинг: инварианты, маршрут по документам, диагностика |
| разработчик подключает свой сервис | [integration/GATEWAY_SERVICE_INTEGRATION_API.md](integration/GATEWAY_SERVICE_INTEGRATION_API.md) |
| разработчик пишет код на `auth-connector` | [integration/AUTH_CONNECTOR_REFERENCE.md](integration/AUTH_CONNECTOR_REFERENCE.md) |
| разработчик поднимает сервис локально | [integration/AUTH_CONNECTOR_SETUP_AND_LOCAL_DEV.md](integration/AUTH_CONNECTOR_SETUP_AND_LOCAL_DEV.md) |
| надо разобраться в самом шлюзе | [gateway/README.md](gateway/README.md) |

## Структура

```
docs/
├── AGENTS.md            # брифинг для ИИ-агента: инварианты, маршрут, грабли
├── gateway/             # по файлу на каждый сервис шлюза
│   ├── README.md        # карта сервисов, порядок запуска, известные расхождения
│   ├── architecture.md  # сети, путь запроса, auth_request, service discovery
│   ├── nginx.md
│   ├── auth-service.md
│   ├── mongo.md
│   ├── notification-service.md
│   ├── notification-bot.md
│   ├── monitoring-service.md
│   ├── dozzle.md
│   ├── docker-socket-proxy.md
│   ├── guard-watchdog.md
│   └── quiz.md
└── integration/         # подключение конечного сервиса
    ├── GATEWAY_SERVICE_INTEGRATION_API.md        # контракт шлюза целиком
    ├── AUTH_CONNECTOR_REFERENCE.md               # API пакета auth-connector
    ├── AUTH_CONNECTOR_SETUP_AND_LOCAL_DEV.md     # подключение и локальный запуск
    └── TELEGRAM_NOTIFICATIONS_INTEGRATION_GUIDE.md
```

## Про ссылки на код

Документы ссылаются на исходники соседних сабмодулей относительными путями
(`../../!gateway/...`, `../../apartment_finder/...`). Такие ссылки работают
**в рабочей копии AnalyticsRepo**, где `docs/` лежит рядом с остальными
сабмодулями. В standalone-просмотре этого репозитория на GitHub они не
разрешатся — это ожидаемо.

Внутренние ссылки (между файлами `docs/`) работают везде.

## Обязательный порядок чтения

`AGENTS.md` **не заменяет** документацию, а маршрутизирует по ней. Любая
задача по интеграции начинается так:

```
AGENTS.md → integration/GATEWAY_SERVICE_INTEGRATION_API.md → профильный документ → код
```

Код на Python/Flask требует ещё и
[integration/AUTH_CONNECTOR_REFERENCE.md](integration/AUTH_CONNECTOR_REFERENCE.md).
Требования к тестам и документации — разделы 7 и 8 [AGENTS.md](AGENTS.md);
они действуют в каждой задаче. Правишь сам шлюз — плюс раздел 9.

## Кто обязан править этот репозиторий

Не только тот, кто пишет документацию. **Агент или разработчик, меняющий код
шлюза `!gateway`, обязан обновить файлы здесь в том же заходе** — таблица
«изменил в шлюзе → обнови в `docs/`» в разделе 8.2 [AGENTS.md](AGENTS.md).

| Что менял | Куда пишешь |
|---|---|
| конечный сервис | документы в репозитории **этого сервиса** |
| шлюз `!gateway` | **сюда**, `docs/gateway/` и, если задет контракт, `docs/integration/` |
| контракт платформы | **сюда**, даже если правка была в коде сервиса |

Документации шлюза в репозитории `!gateway` больше нет: там остался
`!gateway/docs/README.md` — указатель на этот сабмодуль.

## Правила ведения

- **Прав код, а не документ.** Нашли расхождение — правьте документ и
  фиксируйте расхождение, а не выдумывайте, как «должно быть».
- **Известные поломки — в явный список**, а не в молчание: раздел «Известные
  расхождения» в [gateway/README.md](gateway/README.md) и раздел 6
  [AGENTS.md](AGENTS.md). Починили — вычеркните строку.
- **Ломающее контракт изменение** правится и в разделах 3–4 `AGENTS.md`:
  следующий агент читает выжимку оттуда.
- **Не дублировать.** Глубина — в профильном файле, в остальных — ссылка.
  `AGENTS.md` умышленно краток: он маршрутизирует, а не пересказывает.
- **Секретов здесь нет.** Ни паролей, ни ключей, ни токенов — только имена
  переменных окружения.
- Один файл на сервис; новый контейнер в шлюзе = новый файл в `gateway/`
  + строка в карте сервисов.

## Обновление сабмодуля

```bash
cd docs
git pull                        # тянуть по ветке, не через submodule update
# правки, коммит, push
cd ..
git add docs && git commit -m "docs: update"
```

`git submodule update` откатит `docs/` на записанный в родителе коммит —
в этом репозитории это безобидно, в сервисных сабмодулях роняет прод.
