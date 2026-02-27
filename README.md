# Personal Assistant — учебный проект для практики Docker

Простое FastAPI-приложение, которое используется как основа для лабораторных работ по Docker.

## Структура проекта

```
simple_example/
├── app/
│   ├── __init__.py
│   ├── main.py          # FastAPI + маршруты + Redis-кэширование
│   ├── database.py      # Подключение к PostgreSQL
│   ├── models.py        # Модель Note (SQLAlchemy)
│   └── crud.py          # CRUD операции
├── config/
│   └── app.conf         # Конфиг для bind mount
├── docker-compose.yml   # Оркестрация всех сервисов
├── Dockerfile           # Сборка образа приложения
├── .env                 # Переменные окружения (персональные!)
├── requirements.txt     # Python-зависимости
└── README.md
```

## Быстрый старт (Docker Compose)

### 1. Настроить `.env`

Отредактируйте файл `.env` — укажите **своё имя** и **свой порт**:

```bash
APP_OWNER=ВашеИмя
APP_SECRET=ваш_секретный_ключ
APP_PORT=8000          # у каждого студента свой порт!
```

### 2. Запустить все сервисы

```bash
docker compose up -d --build
```

Эта команда:
- Соберёт образ приложения из `Dockerfile`
- Запустит PostgreSQL, Redis и FastAPI
- Настроит сеть между контейнерами
- Создаст volumes для данных

### 3. Проверить работу

```bash
# Статус контейнеров
docker compose ps

# Логи приложения
docker compose logs app

# Проверить API
curl http://localhost:8000/
curl http://localhost:8000/health
curl http://localhost:8000/weather/Moscow
curl http://localhost:8000/weather/Moscow   # второй раз — из кэша!
curl http://localhost:8000/cache/stats
```

### 4. Остановить

```bash
docker compose down           # остановить контейнеры
docker compose down -v        # остановить + удалить volumes
```

---

## Архитектура сервисов

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   FastAPI    │────▶│  PostgreSQL  │     │    Redis     │
│   (app)      │     │    (db)      │     │   (redis)    │
│  :8000       │     │  :5432       │     │  :6379       │
│              │────▶│              │     │              │
│              │────────────────────────▶│              │
└─────────────┘     └─────────────┘     └─────────────┘
       ▲                   │                    │
       │            postgres-data          redis-data
       │              (volume)              (volume)
   APP_PORT
  (хост-порт)
```

### Сервисы в `docker-compose.yml`

| Сервис | Образ | Назначение |
|--------|-------|------------|
| `app` | Собирается из `Dockerfile` | FastAPI-приложение |
| `db` | `postgres:16-alpine` | База данных для заметок |
| `redis` | `redis:7-alpine` | Кэш погодных запросов |

### Ключевые возможности Compose

| Возможность | Где используется |
|-------------|-----------------|
| `depends_on` + `condition` | `app` ждёт готовности `db` и `redis` (healthcheck) |
| `networks` | Общая сеть `assistant-net` для связи сервисов |
| `volumes` | `postgres-data` и `redis-data` для персистентности |
| `environment` | Переменные окружения из `.env` файла |
| `healthcheck` | Проверка готовности PostgreSQL и Redis |
| `restart` | Автоперезапуск при падении (`unless-stopped`) |

---

## Redis-кэширование погоды

Эндпоинт `/weather/{city}` кэширует результаты в Redis, чтобы не спамить внешний API Open-Meteo.

### Как это работает

1. Запрос `/weather/Moscow` — приложение проверяет Redis
2. **Cache miss** — данные запрашиваются у Open-Meteo, сохраняются в Redis с TTL
3. Повторный запрос `/weather/Moscow` — **cache hit**, данные из Redis (мгновенно)
4. Через `CACHE_TTL` секунд кэш истекает — снова запрос к API

### Ключ кэша

Формат: `weather:{APP_OWNER}:{city}` — данные в кэше привязаны к владельцу, что обеспечивает уникальность.

### Поля ответа

В ответе `/weather/{city}` появились новые поля:

| Поле | Описание |
|------|----------|
| `source` | `"api"` — данные из Open-Meteo, `"cache"` — из Redis |
| `hostname` | ID контейнера (полезно при масштабировании) |
| `server_time` | Время сервера в UTC |
| `owner` | Имя владельца из `APP_OWNER` |

### Эндпоинт `/cache/stats`

Показывает статистику кэша: сколько городов закэшировано, TTL каждого ключа.

---

## Эндпоинты

| Метод | URL | Описание |
|-------|-----|----------|
| GET | `/` | Главная страница — приветствие и список эндпоинтов |
| GET | `/health` | Healthcheck — статус приложения и Redis |
| GET | `/docs` | Swagger UI — интерактивная документация |
| GET | `/me` | Информация о владельце |
| GET | `/secret/{key}` | Проверка секретного ключа |
| GET | `/weather/{city}` | Погода с Redis-кэшированием |
| GET | `/cache/stats` | Статистика кэша текущего владельца |
| POST | `/notes` | Создать заметку (тело: `{"text": "..."}`) |
| GET | `/notes` | Получить все заметки |
| GET | `/notes/{id}` | Получить заметку по ID |
| DELETE | `/notes/{id}` | Удалить заметку по ID |

---

## Переменные окружения

| Переменная | Описание | Значение по умолчанию |
|------------|----------|-----------------------|
| `APP_OWNER` | Имя владельца приложения | `anonymous` |
| `APP_SECRET` | Секретный ключ для `/secret/{key}` | `default_secret` |
| `APP_PORT` | Порт приложения на хосте | `8000` |
| `POSTGRES_USER` | Пользователь PostgreSQL | `postgres` |
| `POSTGRES_PASSWORD` | Пароль PostgreSQL | `postgres` |
| `POSTGRES_DB` | Имя базы данных | `assistant_db` |
| `REDIS_URL` | URL подключения к Redis | `redis://redis:6379/0` |
| `CACHE_TTL` | Время жизни кэша в секундах | `300` |

---

## Масштабирование

Для демонстрации масштабирования уберите `container_name` у сервиса `app` в `docker-compose.yml` и запустите несколько реплик:

```bash
docker compose up -d --build --scale app=3
```

Каждый контейнер вернёт свой `hostname` в ответе API — это позволяет увидеть, какой именно экземпляр обработал запрос.

> При масштабировании маппинг фиксированного порта (`APP_PORT:8000`) работать не будет.
> Нужно убрать `ports` или использовать nginx как reverse-proxy перед репликами.

---

## Полезные команды

```bash
# Сборка и запуск
docker compose up -d --build

# Статус
docker compose ps

# Логи конкретного сервиса
docker compose logs -f app
docker compose logs -f redis

# Подключиться к Redis CLI
docker compose exec redis redis-cli
> KEYS weather:*
> GET weather:student:moscow
> TTL weather:student:moscow

# Подключиться к PostgreSQL
docker compose exec db psql -U postgres -d assistant_db
> SELECT * FROM notes;

# Пересобрать только приложение
docker compose up -d --build app

# Полная очистка
docker compose down -v --rmi local
```

---

## Защита работы

Для защиты лабораторной необходимо продемонстрировать:

1. **Персональный порт** — в `.env` указан уникальный `APP_PORT`
2. **Персональные данные в кэше** — ключи содержат ваше имя (`weather:ВашеИмя:город`)
3. **Скриншоты с hostname и временем** — в ответах API есть `hostname` и `server_time`
4. **Работающий кэш** — первый запрос возвращает `"source": "api"`, повторный — `"source": "cache"`
5. **`docker compose ps`** — все три сервиса healthy

### Примеры запросов для скриншотов

```bash
# 1. Главная (hostname + server_time)
curl http://localhost:8000/

# 2. Первый запрос погоды (source: api)
curl http://localhost:8000/weather/London

# 3. Повторный запрос (source: cache)
curl http://localhost:8000/weather/London

# 4. Статистика кэша
curl http://localhost:8000/cache/stats

# 5. Health check (redis: true)
curl http://localhost:8000/health
```

---

## API погоды — важно для пользователей из РФ

Эндпоинт `/weather/{city}` обращается к **[Open-Meteo](https://open-meteo.com/)** (бесплатный, без ключа).

Домены `open-meteo.com` могут быть недоступны из российских сетей. Если `/weather/{city}` возвращает таймаут — используйте VPN.
