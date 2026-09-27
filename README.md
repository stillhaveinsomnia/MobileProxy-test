# NOT FOR PRODUCTION

## MobileProxy-test

**Shitcode for home testing.** Домашний экспериментальный проект для управления мобильными HTTP-прокси, проверки прокси-трафика и тестирования смены WAN IP через модемы и роутеры.

Проект тестировался дома с модемами **HUAWEI** и **ZTE**. Проверялись подключение к устройствам, сценарии reset и то, проходит ли трафик через прокси. Также проверялась работа локально и через VPS-сервер. Это описание домашнего тестирования, а не заявление о production-надежности или универсальной совместимости со всеми моделями модемов.

## Возможности

- React-панель со входом, списком прокси, фильтрами, состояниями и страницами статуса и биллинга.
- Express API с JWT-аутентификацией и ролями admin/operator/user.
- Список прокси и назначение доступа пользователям хранятся в PostgreSQL.
- Reset одной или нескольких прокси ставится в очередь; состояние задания и журнал сохраняются в PostgreSQL.
- Router Manager содержит драйверы OpenWrt, Huawei, MikroTik и ZTE. Управление роутерами выполняется по SSH; отдельные драйверы используют HTTP API модема с SSH-командами в качестве альтернативного пути.
- Reset Executor проверяет WAN IP до и после операции и записывает результат.
- Периодическая проверка прокси отправляет HTTP-запрос через прокси на `api.ipify.org`.
- Статусы и события передаются через Socket.IO; Redis используется для очередей, счетчиков и Pub/Sub.
- 3proxy предоставляет HTTP proxy listeners на портах `3001-3050`; конфигурация включает аутентификацию и ограничения целевых портов.
- Обработчик логов 3proxy и таблица usage ledger предназначены для учета длительности и стоимости использования.

## Как проходит reset

1. Авторизованный клиент отправляет `POST /api/reset` с портом или массивом портов.
2. API проверяет наличие прокси, права пользователя и применимые ограничения, создает запись в `reset_logs` и задание `reset_jobs` в состоянии `pending`.
3. Router Service опрашивает ожидающие задания, выбирает зарегистрированный онлайн-агент и переводит задание в очередь выполнения.
4. Reset Executor берет задание, устанавливает блокировку на ресурс роутера, вызывает драйвер выбранного типа и получает WAN IP после операции.
5. Результат сохраняется в БД и передается клиентам через Redis Pub/Sub и WebSocket.

Состояния задания: `pending`, `routed`, `queued`, `running`, `success`, `failed`, `dead_letter`.

`backend/src/agents/pocAgent.js` — PoC-исполнитель: он имитирует успешный reset и отправляет тестовый IP. Для реальной смены IP нужен настроенный Reset Executor и достижимый роутер с подходящим драйвером.

## Архитектура и порты

| Компонент | Назначение | Порт |
|---|---|---:|
| Frontend (React/Vite, собранный через Nginx) | Web-панель | `80` в основном Compose; `5173` в app Compose |
| Backend (Node.js/Express) | REST API, проверки состояния, Socket.IO | `4000` |
| PostgreSQL | Пользователи, прокси, задания, журналы и биллинг | `5432` |
| Redis | Очереди, временные счетчики и события | `6379` |
| 3proxy | HTTP proxy listeners | `3001-3050` |

Основные каталоги: `backend/` — API и фоновые процессы; `frontend/` — панель; `proxy/` — конфигурация 3proxy; `infra/` — локальные и VPS-скрипты; `backend/tests/` — smoke и audit проверки.

## Требования

- Docker Engine и Docker Compose plugin.
- Node.js 20+ для локального запуска backend и тестовых скриптов.
- Модем/роутер, доступный с хоста backend по сети, и настроенный доступ к его управляющему интерфейсу.
- Для реального reset — подходящий тип устройства, действующие учетные данные и доступ по SSH; для проверки прокси — рабочий uplink и корректная маршрутизация через нужный модем.

## Запуск через Docker Compose

Из корня репозитория:

```powershell
Copy-Item .env.example .env
```

Отредактируйте `.env`: задайте уникальные пароли PostgreSQL и Redis, длинные случайные JWT-секреты, URL панели и параметры прокси. Не публикуйте `.env` и не используйте демонстрационные значения на доступном извне сервере.

Затем соберите и запустите основной Compose-проект:

```powershell
docker compose up --build -d
docker compose ps
docker compose logs -f backend
```

Панель будет доступна на `http://localhost`, API — на `http://localhost:4000`. Проверка доступности API:

```powershell
Invoke-RestMethod http://localhost:4000/api/health
```

Остановить контейнеры:

```powershell
docker compose down
```

Данные PostgreSQL и Redis находятся в именованных Docker volumes. `docker compose down -v` удаляет volumes и данные; выполняйте это только если действительно хотите сбросить локальное состояние.

В репозитории также есть отдельные `docker-compose.core.yml`, `docker-compose.app.yml` и `docker-compose.proxy.yml`. Для обычного первого запуска используйте основной `docker-compose.yml`: отдельные файлы app/proxy настроены на внешнюю Docker-сеть `datacenter_dc-network`.

## Конфигурация

Все серверные параметры задаются переменными окружения; шаблон находится в `.env.example`.

| Переменная | Назначение |
|---|---|
| `POSTGRES_HOST`, `POSTGRES_PORT`, `POSTGRES_DB`, `POSTGRES_USER`, `POSTGRES_PASSWORD` | Подключение backend к PostgreSQL |
| `REDIS_HOST`, `REDIS_PORT`, `REDIS_PASSWORD` | Подключение backend к Redis |
| `NODE_ENV`, `API_PORT` | Окружение и порт API |
| `JWT_SECRET`, `JWT_ACCESS_EXPIRY`, `JWT_REFRESH_EXPIRY` | Подпись и срок действия токенов |
| `RESET_COOLDOWN_SEC`, `RESET_MAX_PER_HOUR`, `RESET_MAX_PER_DAY_USER` | Ограничения reset |
| `MAX_CONNECTIONS_PER_PROXY`, `MAX_CONNECTIONS_PER_USER` | Настройки лимитов подключений |
| `ROUTER_SSH_PORT`, `ROUTER_SSH_TIMEOUT`, `ROUTER_SSH_RETRIES`, `ROUTER_DEFAULT_TYPE` | Параметры подключения к роутерам |
| `HEALTH_CHECK_INTERVAL_SEC`, `HEALTH_CHECK_BATCH_SIZE` | Интервал и размер пачки health checks |
| `PROXY_HOST`, `PROXY_PORT_START`, `PROXY_PORT_END`, `PROXY_AUTH_USER`, `PROXY_AUTH_PASS` | Адрес, порты и учетные данные прокси для backend-проверок |
| `LOG_LEVEL`, `LOG_RETENTION_DAYS` | Уровень и срок хранения логов |
| `VITE_API_URL`, `VITE_WS_URL` | URL API и WebSocket при отдельной сборке frontend |
| `ALLOWED_ORIGINS` | Разрешенные браузерные origins, через запятую |
| `AGENT_REGISTRATION_SECRET`, `AGENT_ID`, `AGENT_TOKEN`, `AGENT_NAME`, `AGENT_REGION` | Регистрация и идентификация агента |
| `EXECUTOR_MODE` | Режим PoC agent/reset executor; hybrid запрещен кодом в production |

Параметры конкретных устройств (адрес, тип, SSH-пользователь и ключ) хранятся в записи прокси. Примеры записей находятся в `backend/src/models/seed.sql`; перед использованием замените тестовые адреса и credentials на свои.

## Проверка прокси вручную

Сначала убедитесь, что нужный порт открыт и модем действительно является исходящим gateway этого proxy listener. Пример запроса с `curl`:

```bash
curl --proxy http://PROXY_USER:PROXY_PASSWORD@HOST:3001 https://api.ipify.org
```

Сравните результат с IP без прокси и повторите после reset. Сам факт доступности порта 3proxy не доказывает, что он выходит через отдельный модем: это зависит от сетевой маршрутизации хоста и конфигурации gateway. Конфигурация `proxy/3proxy.cfg` содержит примечание о настройке маршрутизации через соответствующий роутер.

## API

| Метод | Endpoint | Доступ | Назначение |
|---|---|---|---|
| `POST` | `/api/login` | Публичный | Вход, выдача access/refresh token |
| `POST` | `/api/refresh` | Публичный | Обновление пары токенов |
| `POST` | `/api/logout` | JWT | Выход и отзыв refresh token |
| `POST` | `/api/register` | Admin | Создание пользователя |
| `GET` | `/api/proxies` | JWT | Список прокси; фильтры `group_id`, `status`, `search` |
| `GET` | `/api/proxies/groups` | JWT | Группы прокси |
| `GET` | `/api/proxies/:id` | JWT и права на прокси | Детали и недавние reset-записи |
| `POST` | `/api/reset` | JWT | Reset одного порта или списка портов |
| `GET` | `/api/reset/queue` | JWT | Сводка очереди reset |
| `GET` | `/api/status` | JWT | Состояние системы и сервисов |
| `GET` | `/api/status/resets?period=24h` | JWT | Статистика reset за `1h`, `24h`, `7d` или `30d` |
| `GET` | `/api/health` | Публичный | Health check backend/БД для Docker |
| `POST` | `/api/agents/register` | Shared secret | Регистрация агента через `X-AGENT-SECRET` |
| `POST` | `/api/agents/:id/heartbeat` | Agent Bearer token | Heartbeat агента |
| `GET` | `/api/agents` | Operator/Admin | Список агентов |

Заголовок для защищенных пользовательских endpoints: `Authorization: Bearer <access-token>`.

Одиночный reset:

```bash
curl -X POST http://localhost:4000/api/reset \
  -H "Authorization: Bearer TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"port":3001}'
```

Массовый reset:

```bash
curl -X POST http://localhost:4000/api/reset \
  -H "Authorization: Bearer TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"ports":[3001,3002]}'
```

Bulk reset принимает не более 20 портов за запрос. Применяются cooldown и лимиты из конфигурации; конкретные ограничения зависят от тарифа и состояния Redis.

## Тесты и домашние испытания

В ходе домашних испытаний проект проверялся локально и через VPS-сервер. Использовались модемы **HUAWEI** и **ZTE**; проверялись соединение с устройствами, вызов сценария reset и прохождение HTTP-запросов через прокси с проверкой внешнего IP. Совместимость зависит от модели, прошивки, настроек оператора и сети; это не сертификация всего семейства устройств.

В репозитории есть:

- `backend/tests/smoke/smokeTest.js` — конкурентные запросы reset и наблюдение за состояниями заданий;
- `backend/tests/audit/crossEntityAudit.js` — проверка согласованности заданий, журналов и статусов прокси;
- `backend/tests/experiment/emit_experiment_header.ps1` — создание метаданных экспериментального запуска.

PowerShell-обертки для smoke и audit находятся в `backend/tests/smoke/` и `backend/tests/audit/`. Скрипты требуют запущенных и настроенных API, PostgreSQL, Redis и фоновых компонентов; smoke test создает тестового пользователя и задания, поэтому не запускайте его на production-данных.

## Известные ограничения

- Это home-testing PoC, не готовая production-платформа. Проверяйте команды драйверов и резервного reboot на конкретном устройстве: они могут временно прервать соединение или перезапустить модем.
- `pocAgent.js` имитирует выполнение reset и генерирует фиктивный IP; его события нельзя считать подтверждением смены реального IP.
- Тестовые данные и fallback/mock-ответы присутствуют в seed-данных и отдельных API-контроллерах.
- В текущем `backend/package.json` есть ошибка JSON в разделе `scripts` (нет запятой после команды `worker`); npm/Docker build может завершиться ошибкой до исправления.
- В текущем `backend/src/models/init.sql` таблица `reset_jobs` ссылается на `node_agents` до создания этой таблицы; проверьте порядок DDL перед чистой инициализацией PostgreSQL.
- Убедитесь, что 3proxy действительно маршрутизирует соединения через нужные мобильные uplink'и; наличие listener'ов на портах само по себе этого не настраивает.

## Безопасность перед доступом из интернета

- Замените все примерные пароли, JWT-ключи и agent shared secret; не коммитьте `.env`.
- Настройте TLS, firewall и список `ALLOWED_ORIGINS`.
- Не публикуйте порты PostgreSQL и Redis наружу без необходимости.
- Ограничьте SSH-доступ к роутерам, используйте отдельные credentials и проверьте допустимость выполнения команд драйверами.
- Удалите тестовые аккаунты/данные и отключите mock-режимы перед любым реальным использованием.
- Настройте резервные копии БД, ротацию логов и мониторинг.

## Структура каталогов

```text
backend/   Express API, модели, драйверы роутеров, workers и тесты
frontend/  React/Vite web-панель и Nginx-конфигурация
proxy/     Dockerfile, entrypoint и конфигурация 3proxy
infra/     Скрипты локального/VPS-развертывания и вспомогательные сервисы
docker-compose*.yml
           Основная и раздельные конфигурации Docker Compose
deploy.ps1 Windows-скрипт подготовки локального окружения
setup.sh   Bash-скрипт подготовки Ubuntu
```
