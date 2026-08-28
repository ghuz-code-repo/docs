# quiz — изолированный сервис в шлюзе

Quiz живёт в **чужом репозитории** (`AnalyticsRepo/quiz/quiz_service/`,
`docker-compose.prod.yml`), но проксируется шлюзом и потому описан здесь:
он единственный сервис с ручной маршрутизацией и собственной моделью изоляции.

- **Контейнеры:** `quiz-backend` (FastAPI, `:8000`), `quiz-widget` (nginx,
  `:80`), `quiz-admin` (nginx, `:80`), `quiz-db` (postgres:15-alpine)
- **Сети:** `quiz_network` (общая **только** с gateway-nginx, `internal`),
  `quiz_data_network` (только backend ↔ postgres, `internal`)
- **Маршруты:** `nginx/conf/quiz.inc`, подключается внутри server-блока
  `gateway.inc`

## Почему отдельно

Виджет квиза встраивается на публичный сайт `gh.uz`, то есть его API вызывают
анонимные посетители. Пускать такой сервис в `service_network` — значит дать
ему сетевой путь к auth-service, notification-service и всем бизнес-сервисам.
Поэтому:

- quiz **не входит** в `service_network`; из `quiz_network` достижим только
  `gateway-nginx-1`;
- публичная часть намеренно **без `auth_request`** — авторизация шлюза для
  посетителей сайта не применяется;
- админка и admin-API, наоборот, закрыты обычным `auth_request /verify`.

## Маршруты (`quiz.inc`)

| Location | Куда | Авторизация |
|---|---|---|
| `^~ /quiz/api/v1/admin/` | `quiz-backend:8000/api/v1/admin/` | `auth_request /verify` + rate-limit `quiz_api` |
| `^~ /quiz/api/` | `quiz-backend:8000/api/` | нет, rate-limit `quiz_api` (10 r/s, burst 20) |
| `^~ /quiz/widget/` | `quiz-widget:80/` | нет; CORS + кэш 1 ч |
| `~ ^/quiz/(\d+)/?$` | `quiz-widget:80/index.html` | нет; `Cache-Control: no-store` — полноразмерная страница квиза |
| `= /quiz/admin` | `301 /quiz/admin/` | — |
| `^~ /quiz/admin/` | `quiz-admin:80/` | `auth_request /verify` |
| `/quiz/` (всё прочее) | — | `404` |

Regex-локация `^/quiz/(\d+)/?$` имеет приоритет над префиксным catch-all
`/quiz/`, поэтому страница квиза по числовому id открывается, а всё остальное
под `/quiz/` закрыто.

В backend и админку пробрасывается личность: `X-User-Id`, `X-User-Name`,
`X-User-Admin`, `X-User-Service-Roles`, `X-User-Service-Permissions`,
`X-Service-Key: quiz`. Сервис в `/verify` определяется по первому сегменту
пути, то есть `quiz`, — значит пользователю нужна роль или право в сервисе
`quiz`, иначе `403` → `/access-denied`.

## `unmanaged_routing`

Определение сервиса `quiz` в auth-service **обязано** иметь
`unmanaged_routing: true` **до первой регистрации инстанса**. Иначе
`GetActiveServicesForNginx()` сгенерирует `dynamic/service-quiz.conf` с
`location /quiz/`, который конфликтует с ручным `quiz.inc`.

При этом сам quiz-backend **регистрируется** в реестре и шлёт heartbeat —
ради карточки со статусом на `/menu` и `/services`. Регистрация идёт через
`gateway-nginx-1` (единственный достижимый из `quiz_network` компонент шлюза):

```
REGISTRY_URL=https://gateway-nginx-1/api/registry
REGISTRY_HOST_HEADER=analytics.gh.uz
REGISTRY_VERIFY_TLS=false      # SNI не совпадает с сертификатом gh.uz
SERVICE_KEY=quiz
SERVICE_INTERNAL_URL=http://quiz-backend:8000
INTERNAL_API_KEY=<как в !gateway/.env>
HEARTBEAT_INTERVAL=30
```

## Защита по Origin/Referer — сейчас отключена

Задумано так: запрос допускается, если `Origin` **или** `Referer` указывает
на `gh.uz` (fetch/XHR с сайта всегда шлёт `Origin`, загрузка `<script src>` —
`Referer`). Карты в `nginx.conf`:

```nginx
map $http_origin  $quiz_origin_ok    { default 0; "~^https://(www\.)?gh\.uz$" 1; }
map $http_referer $quiz_referer_ok   { default 0; "~^https://(www\.)?gh\.uz(/|$)" 1; }
map "$quiz_origin_ok$quiz_referer_ok" $quiz_gh_uz_denied { "00" 1; default 0; }
```

**Фактически проверка выключена:** во всех публичных блоках `quiz.inc` строка
`if ($quiz_gh_uz_denied) { return 444; }` закомментирована «временно для
тестирования», а `$quiz_cors_origin` зеркалит любой `Origin`:

```nginx
map $http_origin $quiz_cors_origin {
    default                      $http_origin;   # ← сейчас так
    # default                      "";
    # "~^https://(www\.)?gh\.uz$"  $http_origin;  # ← так задумано
}
```

Плюс `BACKEND_CORS_ORIGINS=*` в самом backend'е (в комментарии compose'а
указано вернуть `https://gh.uz,https://www.gh.uz`).

Чтобы вернуть защиту: раскомментировать `if ($quiz_gh_uz_denied)` в четырёх
публичных блоках, переключить `map $quiz_cors_origin` на закомментированные
строки, вернуть `BACKEND_CORS_ORIGINS`. Rate-limit `quiz_api` работает
независимо и остаётся единственным действующим ограничением.

## Порядок запуска

```bash
cd ~/AnalyticsRepo/!gateway && docker compose up -d      # создаёт quiz_network
cd ~/AnalyticsRepo/quiz/quiz_service
docker compose -f docker-compose.prod.yml up --build -d
```

`quiz_network` объявлена в quiz-compose как `external` — без поднятого шлюза
запуск упадёт.

Обязательные переменные: `QUIZ_DB_PASSWORD`, `INTERNAL_API_KEY`
(оба помечены `:?` — compose откажется стартовать без них).
