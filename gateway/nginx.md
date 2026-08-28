# nginx

Единственный контейнер платформы, доступный извне. Терминирует TLS, проверяет
доступ через `auth_request` к auth-service и проксирует запросы в сервисы,
которые сами наружу не смотрят.

- **Каталог:** `nginx/`
- **Образ:** `nginx:alpine` + `tzdata`
- **Контейнер:** `gateway-nginx-1`, порты `80:80` и `443:443`
- **Сети:** `public_network`, `service_network`, `quiz_network`
- **Healthcheck:** `nginx -t`
- **Домены:** `analytics.gh.uz`, `test_analytics.gh.uz`, `localhost`;
  весь HTTP редиректится в HTTPS (`301`)

## Файлы конфигурации

| Файл | Где смонтирован | Роль |
|---|---|---|
| `conf/nginx.conf` | `/etc/nginx/nginx.conf` | глобальные настройки, `resolver`, `map`-переменные, зоны rate-limit |
| `conf/default.conf` | `/etc/nginx/conf.d/default.conf` | точка входа: `include gateway.inc` |
| `conf/gateway.inc` | `/etc/nginx/conf.d/gateway.inc` | server-блок 443: весь шлюз |
| `conf/quiz.inc` | `/etc/nginx/conf.d/quiz.inc` | маршруты quiz, включается внутри server-блока |
| `conf/errors.inc` | `/etc/nginx/conf.d/errors.inc` | страницы ошибок: перехват, раздача, заголовки; включается внутри server-блока |
| `conf/errors-pages.inc` | `/etc/nginx/conf.d/errors-pages.inc` | **автогенерация** `error-pages/generate.py`: карта `error_page` код → файл |
| `conf/errors-preview.inc` | `/etc/nginx/conf.d/errors-preview.inc` | **автогенерация**: витрина `/errors-preview/` для администратора |
| `conf/errors-contact.inc` | `/etc/nginx/conf.d/errors-contact.inc` | **автогенерация** entrypoint из `SUPPORT_TELEGRAM`; в репозитории файла нет |
| `dynamic/services.conf` + `dynamic/service-*.conf` | том `nginx_dynamic_config` | **автогенерация** auth-service, руками не править |
| `certs/` | `/etc/nginx/certs` (ro) | `gh.uz.chain.pem`, `key.pem` |
| `html/` | образ | `favicon.ico`, `favicon.svg`, `_shared/` |
| `html/errors/` | образ + монтирование `:ro` | 38 страниц ошибок и `errors.css`; **автогенерация**, исходники в `nginx/error-pages/` |

`conf/legacy.inc/` — пустой каталог, оставшийся от ручных конфигов сервисов;
нигде не подключается, маршруты давно генерируются динамически.

## Entrypoint

`docker-entrypoint.sh` перед стартом nginx создаёт
`/etc/nginx/conf.d/dynamic/services.conf`, если файла нет, — иначе `include`
упал бы и шлюз не поднялся бы без единого зарегистрированного сервиса.
`nginx -t` выполняется только для лога: недоступный upstream на этом этапе
нормален.

Там же генерируется `/etc/nginx/conf.d/errors-contact.inc` — одна директива
`set $gw_support_telegram "<логин>";` из переменной окружения. nginx не читает
окружение в конфиге, поэтому значение приходится превращать в конфиг на старте.
Значение чистится до `[A-Za-z0-9_]`: кавычка или перевод строки в переменной
сломали бы конфиг и не дали шлюзу подняться.

## Переменные окружения

| Переменная | Значение | Что делает |
|---|---|---|
| `TZ` | `Asia/Tashkent` | часовой пояс в логах и на страницах ошибок |
| `SUPPORT_TELEGRAM` | логин без `@`, по умолчанию пусто | ссылка «написать в Telegram» на страницах ошибок. Пусто — блок не выводится вовсе. Не секрет: значение видит любой пользователь портала |

Переменная приходит из `!gateway/.env` через `docker-compose.yaml`. После её
правки нужен `docker compose up -d --force-recreate nginx`: `restart` не
перечитывает `env_file`.

## Почему `proxy_pass` через переменную

```nginx
resolver 127.0.0.11 valid=10s ipv6=off;   # DNS докера
set $backend_referal http://referal:80;
proxy_pass $backend_referal;
```

С литеральным адресом nginx резолвит имя **при загрузке конфига** и падает,
если контейнера ещё нет. С переменной + `resolver` резолв происходит в
рантайме: шлюз стартует и живёт, даже когда половина сервисов не поднята,
а пользователь видит `502` вместо отказа старта.

## Что раздаёт сам nginx

| Путь | Источник |
|---|---|
| `/_shared/` | `html/_shared` — вендоренные JS/CSS-библиотеки, общие для всех сервисов (`Cache-Control: immutable`, 30 дней) |
| `/favicon.svg` | `html/favicon.svg` |
| `/_errors/<код>.html` | `html/errors/` — `internal`, только по редиректу `error_page`, SSI включён |
| `/_errors/errors.css` | `html/errors/errors.css` — публично, кэш 1 час |
| `/errors-preview/` | витрина всех кодов, `auth_request /verify-admin` |

`sub_filter` вставляет `<link rel="icon" href="/vite.svg?v=2">` перед `</head>`
всех HTML-ответов; сам `/vite.svg` проксируется в auth-service.
Запросы к `/static/img/favicon.(ico|png)` возвращают `404`, а
`/static/img/favicon.svg` — `301` на `/favicon.svg`: единый favicon на портал.

## Маршруты в auth-service

Аутентификация: `/`, `/login`, `/logout`, `/forgot-password`,
`/reset-password`, `/access-denied`.
Интерфейс: `/menu`, `/profile`, `/settings`, `/admin-menu`.
Админка: `/services`, `/service/`, `/users`, `/roles`, `/permissions`,
`/notification-settings`, `/migration`, `/document-types`,
`/available-services`, `/check-user-exists`.
Статика и данные: `/static/` (кэш 1 ч), `/data/`, `/avatar/` (`no-store`).
API: `^~ /api/`.

> `/admin-menu`, `/roles`, `/permissions` в auth-service **не реализованы** —
> вернётся 404. См. «Известные расхождения» в [README.md](README.md).

## Внутренние `auth_request`-эндпоинты

| Location | Проксирует | Кто использует |
|---|---|---|
| `= /verify` | `auth-service/verify` | все динамические сервисные конфиги и quiz |
| `= /verify-admin` | `auth-service/verify-admin` | зарезервировано |
| `= /verify-logs-auth` | `auth-service/verify-logs-access?service=$logs_referer_service` | `/logs` |
| `= /verify-service-logs-auth` | `auth-service/verify-logs-access?service=$service_key` | `/services/<key>/logs` |

Все помечены `internal` — снаружи не вызываются.

## Rate limiting

| Зона | Ключ | Темп | Где применяется |
|---|---|---|---|
| `auth_login` | IP | 5 r/s, burst 10 | `/login`; `/reset-password` (burst 5) |
| `auth_recovery` | IP, **только POST** | 3 r/min, burst 5 | `/forgot-password` |
| `quiz_api` | IP | 10 r/s, burst 20 | `/quiz/api/` |

`auth_recovery` использует `map $request_method $recovery_limit_key`: для GET
ключ пустой, и nginx лимит не применяет. Иначе повторный заход на страницу
восстановления съедал бы лимит и упирался в `503`.

## Заголовки безопасности

На всех ответах server-блока:

```
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
X-Frame-Options: DENY
X-Content-Type-Options: nosniff
X-XSS-Protection: 1; mode=block
Referrer-Policy: strict-origin-when-cross-origin
Content-Security-Policy: default-src 'self'; script-src 'self' 'unsafe-inline'
  'unsafe-eval' https://cdn.jsdelivr.net; style-src 'self' 'unsafe-inline'
  https://cdnjs.cloudflare.com https://fonts.googleapis.com https://cdn.jsdelivr.net;
  img-src 'self' data:; font-src 'self' https://cdnjs.cloudflare.com
  https://fonts.gstatic.com https://cdn.jsdelivr.net; connect-src 'self';
  frame-ancestors 'none'
```

`X-Frame-Options` переопределяется на `SAMEORIGIN` в блоках Dozzle — иначе
логи не открылись бы в iframe карточки сервиса.

CSP разрешает CDN, но **прод CDN не видит**: ассеты вендорятся в
`static/vendor` и `/_shared/`. Разрешения в CSP — наследие, не разрешение
подключать CDN.

## Обработка ошибок

| Код | Поведение |
|---|---|
| `401` | `302 /login?redirect=$request_uri` |
| `403` | `302 /access-denied?service=$request_uri` |
| `403` на `/logs` | `302 /menu` |
| `403` на `/services/<key>/logs` | `302 /services/<key>` |
| остальные 4xx и 5xx | стилизованная страница `html/errors/<код>.html` |

Кодов со страницей 38: `400`, `402`, `404`–`418`, `421`–`431`, `451`,
`500`–`511`. Каталог с текстами — `nginx/error-pages/codes.py`, там же
генератор и тесты; подробности и порядок правки — `nginx/error-pages/README.md`.

`401` и `403` страниц не получают намеренно: это часть контракта авторизации,
пользователя надо увести на вход или на `/access-denied`, а не показать ему
красивый тупик.

### Перехват ответов сервисов

`proxy_intercept_errors on` объявлен на уровне server-блока, поэтому действует
и на автогенерируемые конфиги сервисов. Без него `500` от сервиса уходил бы
пользователю как есть — голой страницей Flask или пустым телом.

**Тело ответа сервиса при перехвате выбрасывается.** Разбирать JSON nginx не
умеет, поэтому на странице остаётся то, что видно снаружи: код, `$upstream_status`,
`$upstream_addr`, адрес запроса, время и `$request_id`. Сервис, которому есть
что сказать, кладёт текст в заголовки ответа `X-Error-Code` и `X-Error-Detail` —
шлюз читает их как `$upstream_http_*` и показывает в блоке подробностей.

Перехват отключён (`proxy_intercept_errors off`) там, где HTML вместо ответа
ломает вызывающий код:

| Где | Почему |
|---|---|
| `/static/`, `/data/`, `/avatar/`, `/vite.svg`, `<prefix>/static/` | ассеты: браузеру нужен код, а не страница на несколько килобайт |
| `^~ /api/`, `/check-user-exists` | JSON-API портала, ответы читает JS админки и `auth-connector` |
| `= /verify`, `= /verify-admin`, `= /verify-logs-auth`, `= /verify-service-logs-auth` | подзапросы `auth_request` |
| `/logs`, `/services/<key>/logs` | вебсокеты Dozzle |
| `<prefix>/health` | опрашивают monitoring-service и docker |

Обратная сторона решения: **JSON-ответы сервисов с кодом 4xx и 5xx под их
префиксом до клиента больше не доходят** — `fetch()` внутри сервиса получит
HTML-страницу. Сервисам, у которых есть свой JSON-API под префиксом, нужно
либо проверять `content-type`, либо отдавать ошибки кодом `200` с телом.
Ветка «JSON вместо страницы для API-клиентов» заготовлена в `errors.inc`
и включается вместе с `map $http_accept $gw_wants_json` в `nginx.conf`.

### Чей раздел ответил

Имя на странице берётся из `$gw_service_key` — переменную выставляет
сгенерированный конфиг сервиса (шаблон в `auth-service/routes/nginx_config.go`).
Строка стоит **до** `rewrite ... break`: с флагом `break` фаза rewrite
обрывается, и `set` после него не выполнится.

Если до сервиса не дошло (незарегистрированный префикс отдаёт `404` на уровне
шлюза), имя берётся из первого сегмента пути — `map $request_uri
$gw_path_service` в `nginx.conf`. Источник именно `$request_uri`: `error_page`
делает внутренний редирект, и `$uri` к моменту вычисления карты указывал бы
уже на страницу ошибки. Собственные разделы шлюза в карте дают пустую строку,
и страница пишет «шлюз портала».

### Прочее по ошибкам

- `limit_req_status 429` — превышение лимита частоты отдаёт `429`, а не
  дефолтный `503`: причина другая, и текст страницы должен ей соответствовать.
- Фон с частицами — тот же `particles.js`, что на остальных страницах портала,
  из `/_shared/js/particles.min.js`. Файл отдаёт сам nginx, поэтому анимация
  жива и когда лежит auth-service. Скрипта нет или включён
  `prefers-reduced-motion` — страница просто рисуется без фона. Из-за внешнего
  файла в CSP страниц ошибок стоит `script-src 'self' 'unsafe-inline'`.
- Ссылка на поддержку берётся из `$gw_support_telegram` (см. `SUPPORT_TELEGRAM`
  выше) и обёрнута в `<!--# if -->`: переменная пуста — блока нет.
- Тема страницы читается из cookie `gh_theme` через SSI — той же, что и на
  остальных страницах портала; без cookie работает системная тема.
- SSI включён **только** на `/_errors/` и `/errors-preview/`. Глобально его
  включать нельзя: nginx начнёт разбирать HTML всех проксируемых страниц.
- `add_header` в `location /_errors/` отменяет наследование серверных
  заголовков безопасности, поэтому они перечислены в блоке заново, с более
  строгим CSP.

## Прочее

- `client_max_body_size 100M` — импорт Excel и вложения документов.
- `gzip` на текстовые типы, уровень 6.
- Общие `proxy_set_header Host / X-Real-IP / X-Forwarded-For /
  X-Forwarded-Proto` объявлены на уровне server-блока и наследуются всеми
  `location`.

## Перезагрузка конфига

Руками:

```bash
docker exec gateway-nginx-1 nginx -t && docker kill -s HUP gateway-nginx-1
```

Автоматически: auth-service шлёт `SIGHUP` через docker-socket-proxy при
регистрации сервиса. Логи проверки — `docker logs gateway-nginx-1`.
