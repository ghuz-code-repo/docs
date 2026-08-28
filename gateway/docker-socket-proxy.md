# docker-socket-proxy

Единственная точка, где `/var/run/docker.sock` попадает внутрь контейнера
(не считая незапущенного [guard-watchdog](guard-watchdog.md)). Пропускает
узкий набор операций Docker API и режет всё остальное.

- **Образ:** `tecnativa/docker-socket-proxy:latest`
- **Контейнер:** `gateway-docker-proxy`, слушает `:2375`
- **Сеть:** только `service_network`
- **Том:** `/var/run/docker.sock` → `/var/run/docker.sock` **:ro**
- **Healthcheck:** `GET /version`

## Зачем

Доступ к докер-сокету равен рутовому доступу на хост: имея сокет, контейнер
может запустить привилегированный контейнер и смонтировать корень.
Двум сервисам шлюза докер всё-таки нужен:

| Кто | Что делает | Какие операции |
|---|---|---|
| auth-service | перезагружает nginx после генерации конфигов | `POST /containers/gateway-nginx-1/kill?signal=HUP` |
| dozzle | читает и стримит логи | `GET /containers/json`, `/logs`, `/events`, `/info` |

Прокси даёт им ровно это.

## Разрешённые и запрещённые операции

| Переменная | Значение | Кому нужно |
|---|---|---|
| `CONTAINERS` | `1` | список и инспект контейнеров (dozzle) |
| `INFO` | `1` | `GET /info` — dozzle v10+ дёргает на старте |
| `POST` | `1` | отправка сигнала (перезагрузка nginx) |
| `LOG` | `1` | стриминг логов (dozzle) |
| `EVENTS` | `1` | обновления в реальном времени (dozzle) |
| `EXEC` | **`0`** | отключено намеренно |
| `IMAGES`, `NETWORKS`, `VOLUMES`, `BUILD`, `COMMIT`, `SWARM`, `NODES`, `SERVICES`, `TASKS`, `SECRETS`, `CONFIGS` | `0` | не нужно никому |

### Почему `EXEC=0`

Перезагрузка nginx делается **сигналом**, а не `docker exec nginx -s reload`:

```go
POST http://docker-socket-proxy:2375/containers/gateway-nginx-1/kill?signal=HUP
```

`EXEC=1` дал бы возможность выполнить произвольную команду в любом
контейнере — по сути ту же компрометацию, от которой прокси и защищает.
`POST=1` без `EXEC` оставляет только сигналы, старт/стоп и подобное.

## Что остаётся возможным

`CONTAINERS=1` + `POST=1` — это всё ещё право остановить или убить любой
контейнер платформы. Полной изоляции такая конфигурация не даёт; она
исключает выполнение кода и работу с образами, сетями и томами.
Отсюда и вторая ступень защиты: обращение к прокси от контейнера, которого
нет в allowlist, считается индикатором компрометации
(см. [guard-watchdog.md](guard-watchdog.md)).

Разрешённые клиенты по умолчанию: `auth-service` и `dozzle`
(`WATCHDOG_PROXY_ALLOWED`).

## Проверка

```bash
# из контейнера в service_network
wget -qO- http://docker-socket-proxy:2375/version
wget -qO- http://docker-socket-proxy:2375/containers/json | head -c 200

# запрещённое вернёт 403
wget -qO- http://docker-socket-proxy:2375/images/json
```

Логи прокси (haproxy) содержат клиентский IP в начале строки — по ним
watchdog и определяет, кто ходил.

## Грабли

- **Имя контейнера nginx зашито в переменную.** auth-service берёт его из
  `NGINX_CONTAINER_NAME` (по умолчанию `gateway-nginx-1`); при смене имени
  контейнера или проекта compose перезагрузка молча перестанет работать —
  в логах будет `nginx reload returned status 404`.
- **Прокси должен подняться раньше auth-service** — в `depends_on` это учтено,
  но при ручном рестарте порядок легко нарушить.
