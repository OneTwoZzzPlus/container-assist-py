# 🎓 Personal Assistant — учебный проект для практики Docker

Простое FastAPI-приложение, которое используется как основа для лабораторной работы по Docker.

## Лабораторная 3. Персистентность и изоляция

| Параметр                    | Ваше значение   |
| --------------------------- | --------------- |
| Номер ИСУ                   | 467335          |
| Персональный порт assistant | 8035            |
| Персональный порт postgres  | 5465            |
| Персональный префикс        | sakulin         |
| Секретный ключ              | secret35sakulin |
| Имя базы данных             | sakulin35       |

### 1. Запуск контейнера с именованным volume

```bash
# Создание volume
docker volume create sakulin-pgdata
docker volume ls | grep sakulin-pgdata
docker volume inspect sakulin-pgdata

# Запуск контейнера с volume
docker run -d \
  --name sakulin-postgres \
  -e POSTGRES_USER=user_35 \
  -e POSTGRES_PASSWORD=pgpass_35 \
  -e POSTGRES_DB=assistant_35 \
  -v sakulin-pgdata:/var/lib/postgresql/data \
  -p 5465:5432 \
  postgres:15
docker ps --filter "name=sakulin-postgres" --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"

# Сборка assistant
docker build -t sakulin-assistant:v3 .
docker images | grep sakulin-assistant:v3

# Запуск контейнера assistant
docker run -d \
  --name sakulin-assistant \
  --add-host=host.docker.internal:host-gateway \
  -p 8035:8035 \
  -e APP_OWNER=sakulin \
  -e APP_SECRET=secret35sakulin \
  -e DATABASE_URL="postgresql://user_35:pgpass_35@host.docker.internal:5465/assistant_35" \
  sakulin-assistant:v3
docker ps --filter "name=sakulin-assistant" --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"

# Главная страница
curl http://localhost:8035/; echo
# Проверка подключения к БД
curl http://localhost:8035/health; echo
# Создание заметки
curl -X POST http://localhost:8035/notes \
  -H "Content-Type: application/json" \
  -d '{"text": "ЛР3: данные в БД"}'; echo
# Получение заметки
curl http://localhost:8035/notes; echo
```

### 2. Тест персистентности

```bash
# Удаление контейнера assistant
docker stop sakulin-assistant
docker rm sakulin-assistant
docker ps --filter "name=sakulin" --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"

# Запуск контейнера assistant
docker run -d \
  --name sakulin-assistant \
  --add-host=host.docker.internal:host-gateway \
  -p 8035:8035 \
  -e APP_OWNER=sakulin \
  -e APP_SECRET=secret35sakulin \
  -e DATABASE_URL="postgresql://user_35:pgpass_35@host.docker.internal:5465/assistant_35" \
  sakulin-assistant:v3
docker ps --filter "name=sakulin-assistant" --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"

# Получите заметки — они должны быть на месте!
curl http://localhost:8035/notes

# Артефакт работы
echo "========================================" && \
echo "Тест персистентности: $(date '+%H:%M:%S')" && \
echo "$(whoami)@$(hostname)" && \
echo "========================================" && \
curl -s http://localhost:8035/notes | python3 -m json.tool --no-ensure-ascii
```

### 3. Bind mount для конфигурации

```bash
# Просмотр конфигурации
cat config/app.conf

# Перезапуск assistant c примонтированным конфигом
docker stop sakulin-assistant
docker run -d \
  --name sakulin-assistant-config \
  --add-host=host.docker.internal:host-gateway \
  -p 8035:8035 \
  -e APP_OWNER=sakulin \
  -e APP_SECRET=secret35sakulin \
  -e DATABASE_URL="postgresql://user_35:pgpass_35@host.docker.internal:5465/assistant_35" \
  -v $(pwd)/config:/app/config:ro \
  sakulin-assistant:v3
docker ps --filter "name=sakulin-assistant" --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"

# Вход в контейнер
docker exec -it sakulin-assistant-config sh
# Внутри контейнера:
cat /app/config/app.conf
# Выход
exit
```

### 4. Пользовательские сети для изоляции

```bash
# Создание пользовательской сети
docker network create sakulin-backend
docker network ls --filter name=sakulin

# Остановка старых контейнеров
docker stop sakulin-assistant-config
docker rm sakulin-assistant-config
docker stop sakulin-postgres sakulin-assistant
docker rm sakulin-postgres sakulin-assistant
# Запуск БД в пользовательской сети
docker run -d \
  --name sakulin-postgres \
  --network sakulin-backend \
  -p 5465:5432 \
  -e POSTGRES_USER=user_35 \
  -e POSTGRES_PASSWORD=pgpass_35 \
  -e POSTGRES_DB=assistant_35 \
  -v sakulin-pgdata:/var/lib/postgresql/data \
  postgres:15
# Запуск ассистента в той же сети
docker run -d \
  --name sakulin-assistant \
  --network sakulin-backend \
  -p 8035:8035 \
  -e APP_OWNER=sakulin \
  -e APP_SECRET=secret35sakulin \
  -e DATABASE_URL="postgresql://user_35:pgpass_35@sakulin-postgres:5432/assistant_35" \
  -v $(pwd)/config:/app/config:ro \
  sakulin-assistant:v3
# Проверка работы контейнеров
docker ps --filter "name=sakulin" --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"

# Проверка работоспособности
curl http://localhost:8035/health; echo
# Создание новой заметки
curl -X POST http://localhost:8035/notes \
  -H "Content-Type: application/json" \
  -d '{"text": "ЛР3: сеть работает!"}'; echo
# Получение всех заметок
curl http://localhost:8035/notes; echo

# Запуск временного контейнера вне сети
docker run -it --rm alpine:3.18 ping -c 2 sakulin-postgres
# Запуск временного контейнера ВНУТРИ сети
docker run -it --rm --network sakulin-backend alpine:3.18 ping -c 2 sakulin-postgres
```

### 5. Финальный артефакт

```bash
echo "========================================" && \
echo "ЛР3 завершена: $(date '+%d.%m.%Y %H:%M:%S')" && \
echo "Студент: $(whoami)@$(hostname)" && \
echo "========================================" && \
echo "" && \
echo "Контейнеры:" && \
docker ps --filter "name=sakulin-" --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}" && \
echo "" && \
echo "Volumes:" && \
docker volume ls --filter "name=sakulin-" && \
echo "" && \
echo "Сети:" && \
docker network ls --filter "name=sakulin-"
```

### 6. Очистка

```bash
docker rm -f sakulin-assistant
docker rm -f sakulin-assistant-config
docker rmi sakulin-assistant:v3
```

## 📁 Структура проекта

```tree
simple_example/
├── app/
│   ├── __init__.py
│   ├── main.py          # FastAPI + маршруты
│   ├── database.py      # Подключение к PostgreSQL
│   ├── models.py        # Модель Note (SQLAlchemy)
│   └── crud.py          # CRUD операции
├── config/
│   └── app.conf         # Конфиг для bind mount (челлендж)
├── requirements.txt     # Python-зависимости
└── README.md
```

## 💻 Локальный запуск (без Docker)

### 1. Установить зависимости

```bash
cd simple_example
pip install -r requirements.txt
```

### 2. Задать переменные окружения

**macOS / Linux:**

```bash
export APP_OWNER=ВашеИмя
export APP_SECRET=ваш_секретный_ключ
export DATABASE_URL=ваш урл
```

**Windows (PowerShell):**

```powershell
$env:APP_OWNER="ВашеИмя"
$env:APP_SECRET="ваш_секретный_ключ"
$env:DATABASE_URL="postgresql://postgres"
```

**Windows (CMD):**

```cmd
set APP_OWNER=ВашеИмя
set APP_SECRET=ваш_секретный_ключ
set DATABASE_URL=postgresql://postgres:postgres@localhost:5432/assistant_db
```

> Если не задать переменные, приложение запустится со значениями по умолчанию.
> Для работы заметок необходима запущенная PostgreSQL (см. ниже).

### 3. Запустить сервер

```bash
uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload
```

Флаг `--reload` автоматически перезапускает сервер при изменении кода.

### 4. Открыть в браузере

- Приложение: <http://localhost:8000>
- Swagger UI: <http://localhost:8000/docs>

## 📡 Эндпоинты

| Метод  | URL               | Описание                                                               |
| ------ | ----------------- | ---------------------------------------------------------------------- |
| GET    | `/`               | Главная страница — приветствие и список эндпоинтов                     |
| GET    | `/health`         | Healthcheck — статус приложения                                        |
| GET    | `/docs`           | Swagger UI — интерактивная документация (встроена в FastAPI)           |
| GET    | `/me`             | Информация о владельце (из переменных окружения)                       |
| GET    | `/secret/{key}`   | Проверка секретного ключа — убедитесь, что `APP_SECRET` пробрасывается |
| GET    | `/weather/{city}` | Текущая погода в городе через Open-Meteo API                           |
| POST   | `/notes`          | Создать заметку в БД (тело: `{"text": "..."}`)                         |
| GET    | `/notes`          | Получить все заметки текущего владельца                                |
| GET    | `/notes/{id}`     | Получить заметку по ID                                                 |
| DELETE | `/notes/{id}`     | Удалить заметку по ID                                                  |

### Примеры запросов

```bash
# Главная
curl http://localhost:8000/

# Погода в Москве
curl http://localhost:8000/weather/Moscow

# Проверка секретного ключа
curl http://localhost:8000/secret/ваш_секретный_ключ

# Создать заметку (сохраняется в PostgreSQL)
curl -X POST http://localhost:8000/notes \
  -H "Content-Type: application/json" \
  -d '{"text": "Моя первая заметка"}'

# Получить заметки текущего владельца
curl http://localhost:8000/notes

# Получить заметку по ID
curl http://localhost:8000/notes/1

# Удалить заметку
curl -X DELETE http://localhost:8000/notes/1
```

## 🔑 Переменные окружения

| Переменная     | Описание                                     | Значение по умолчанию                                        |
| -------------- | -------------------------------------------- | ------------------------------------------------------------ |
| `APP_OWNER`    | Имя владельца приложения                     | `anonymous`                                                  |
| `APP_SECRET`   | Секретный ключ для эндпоинта `/secret/{key}` | `default_secret`                                             |
| `DATABASE_URL` | Строка подключения к PostgreSQL              | `postgresql://postgres:postgres@localhost:5432/assistant_db` |

## 🗄 PostgreSQL (для эндпоинтов /notes)

Заметки хранятся в PostgreSQL. Для локальной разработки проще всего запустить БД через Docker:

```bash
docker run -d \
  --name assistant-db \
  -e POSTGRES_USER=postgres \
  -e POSTGRES_PASSWORD=postgres \
  -e POSTGRES_DB=assistant_db \
  -p 5432:5432 \
  postgres:16-alpine
```

Таблица `notes` создаётся автоматически при старте приложения (`Base.metadata.create_all`).

### Файл config/app.conf

В папке `config/` лежит файл `app.conf` — пример конфигурации для **bind mount** в лабораторной работе 4. Студент может монтировать этот файл в контейнер и использовать значения из него.

## 🌤 API погоды — важно для пользователей из РФ

Эндпоинт `/weather/{city}` обращается к открытому API **[Open-Meteo](https://open-meteo.com/)** (бесплатный, без ключа):

1. **Геокодинг** — `geocoding-api.open-meteo.com` — получает координаты города
2. **Прогноз** — `api.open-meteo.com` — получает текущую погоду по координатам

### ⚠️ Блокировки РКН

Домены `open-meteo.com` могут быть недоступны из российских сетей из-за блокировок РКН.

Если эндпоинт `/weather/{city}` возвращает ошибку таймаута — **необходимо использовать средства обхода блокировок:**

- **VPN** — включите VPN на машине, где запущен Docker

## ✅ Как проверить, что всё работает

1. `curl http://localhost:8000/` — должен вернуть приветствие с вашим именем
2. `curl http://localhost:8000/me` — должен показать `secret_prefix` (первые 6 символов вашего ключа)
3. `curl http://localhost:8000/secret/ваш_ключ` — должен вернуть `LAB2_SUCCESS_ВАШЕИМЯ`
4. `curl http://localhost:8000/weather/London` — должен вернуть температуру, влажность, ветер
5. `docker inspect --format='{{.State.Health.Status}}' personal-assistant` — должен вернуть `healthy`
