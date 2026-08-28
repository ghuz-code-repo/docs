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
| `dynamic/services.conf` + `dynamic/service-*.conf` | том `nginx_dynamic_config` | **автогенерация** auth-service, руками не править |
| `certs/` | `/etc/nginx/certs` (ro) | `gh.uz.chain.pem`, `key.pem` |
| `html/` | образ | `404.html`, `favicon.ico`, `favicon.svg`, `_shared/` |

`conf/legacy.inc/` — пустой каталог, оставшийся от ручных конфигов сервисов;
нигде не подключается, маршруты давно генерируются динамически.

## Entrypoint

`docker-entrypoint.sh` перед стартом nginx создаёт
`/etc/nginx/conf.d/dynamic/services.conf`, если файла нет, — иначе `include`
упал бы и шлюз не поднялся бы без единого зарегистрированного сервиса.
`nginx -t` выполняется только для лога: недоступный upstream на этом этапе
нормален.

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
| `/404.html` | `html/404.html` |

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
| `404` | `html/404.html` |

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
