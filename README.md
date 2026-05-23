# Лабораторная работа 04. Проектирование и работа с MongoDB

## Дисциплина

Архитектура информационных систем / Программная инженерия.

## Вариант

Вариант 10 - планирование задач.

Приложение содержит основные сущности:

- пользователь;
- цель;
- задача.

PostgreSQL из лабораторной работы 3 остается основным хранилищем для
`users`, `goals` и `tasks`. MongoDB добавлена для документных данных:

- история активности задач;
- комментарии к задачам;
- журнал уведомлений.

## Технологии

- C++20;
- Yandex Userver;
- PostgreSQL 16;
- MongoDB 7;
- Docker Compose;
- REST API.

Redis и RabbitMQ в лабораторной работе 4 не используются.

## Структура проекта

```text
.
├── CMakeLists.txt
├── Dockerfile
├── docker-compose.yaml
├── configs/
│   └── static_config.yaml
├── db/
│   ├── schema.sql
│   ├── data.sql
│   └── queries.sql
├── mongo/
│   ├── validation.js
│   ├── data.js
│   └── queries.js
├── src/
│   └── main.cpp
├── tests/
│   └── curl_examples.md
├── schema_design.md
├── optimization.md
└── openapi.yaml
```

## Хранилища данных

PostgreSQL хранит нормализованные основные сущности:

- `users` - пользователи и роли;
- `goals` - цели;
- `tasks` - задачи цели, исполнитель, автор, статус и срок.

MongoDB использует базу `task_planning_mongo` и коллекции:

- `task_activity` - события истории задачи;
- `task_comments` - комментарии к задаче;
- `notification_log` - журнал уведомлений.

Подробное описание документной модели, выбора коллекций и решения
embedded vs references находится в [schema_design.md](schema_design.md).

## Запуск

Собрать и запустить API, PostgreSQL и MongoDB:

```bash
docker compose up --build
```

API доступен по адресу:

```text
http://localhost:8080
```

Если порт `8080` уже занят другим контейнером, можно выбрать другой host-порт:

```bash
API_PORT=18080 docker compose up --build
```

В PowerShell:

```powershell
$env:API_PORT = "18080"
docker compose up --build
```

Тогда API будет доступен по адресу `http://localhost:18080`.

MongoDB не публикует порт на host и доступна внутри compose-сети как сервис
`mongo`. Скрипты из папки `mongo/` монтируются в контейнер по пути `/scripts`.

## MongoDB scripts

Создать коллекции, validators и индексы:

```bash
docker compose exec mongo mongosh /scripts/validation.js
```

Загрузить тестовые документы, по 10 документов в каждую коллекцию:

```bash
docker compose exec mongo mongosh /scripts/data.js
```

Выполнить CRUD-запросы и aggregation pipeline:

```bash
docker compose exec mongo mongosh /scripts/queries.js
```

`queries.js` содержит create/read/update/delete операции с операторами `$eq`,
`$ne`, `$gt`, `$lt`, `$in`, `$and`, `$or`, `$addToSet`, `$pull`, а также
aggregation pipeline с группировкой `task_activity` по `taskId` и `type`.

## API endpoints

Базовые endpoints PostgreSQL из лабораторной 3 сохранены:

| Метод | URL | Назначение | Auth |
|---|---|---|---|
| GET | `/ping` | Проверка сервиса | Нет |
| POST | `/api/users` | Создание пользователя | Нет |
| POST | `/api/auth/login` | Получение bearer token | Нет |
| GET | `/api/users/by-login?login=alexey` | Поиск пользователя по логину | Да |
| GET | `/api/users/search?mask=iv` | Поиск пользователей по имени/фамилии | Да |
| POST | `/api/goals` | Создание цели | Да |
| GET | `/api/goals` | Получение целей | Да |
| POST | `/api/goals/{goalId}/tasks` | Создание задачи в цели | Да |
| GET | `/api/goals/{goalId}/tasks` | Получение задач цели | Да |
| PATCH | `/api/goals/{goalId}/tasks/{taskId}/status` | Изменение статуса задачи | Да |

Новые endpoints лабораторной 4:

| Метод | URL | Назначение | Хранилище |
|---|---|---|---|
| POST | `/api/goals/{goalId}/tasks/{taskId}/comments` | Добавить комментарий к задаче | MongoDB |
| GET | `/api/goals/{goalId}/tasks/{taskId}/comments` | Получить комментарии задачи | MongoDB |
| GET | `/api/goals/{goalId}/tasks/{taskId}/activity` | Получить историю активности задачи | MongoDB |

При успешном `PATCH /api/goals/{goalId}/tasks/{taskId}/status` API обновляет
статус задачи в PostgreSQL и добавляет событие `status_changed` в MongoDB:

```json
{
  "taskId": 1,
  "goalId": 1,
  "type": "status_changed",
  "actor": { "userId": 1 },
  "payload": { "newStatus": "in_progress" },
  "createdAt": "2026-05-24T10:00:00Z"
}
```

## Проверка API

Проверить сервис:

```bash
curl http://localhost:8080/ping
```

Получить учебный bearer token для seed-пользователя:

```bash
TOKEN=$(curl -s -X POST http://localhost:8080/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"login":"alexey","password":"pass123"}' | sed -E 's/.*"token":"([^"]+)".*/\1/')
```

Получить задачи цели из PostgreSQL:

```bash
curl -i http://localhost:8080/api/goals/1/tasks \
  -H "Authorization: Bearer $TOKEN"
```

Добавить комментарий к задаче в MongoDB:

```bash
curl -i -X POST http://localhost:8080/api/goals/1/tasks/1/comments \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"text":"Комментарий из API лабораторной 4","tags":["api","mongo"]}'
```

Получить комментарии задачи:

```bash
curl -i http://localhost:8080/api/goals/1/tasks/1/comments \
  -H "Authorization: Bearer $TOKEN"
```

Изменить статус задачи и записать событие в `task_activity`:

```bash
curl -i -X PATCH http://localhost:8080/api/goals/1/tasks/1/status \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"status":"in_progress"}'
```

Получить историю активности задачи:

```bash
curl -i http://localhost:8080/api/goals/1/tasks/1/activity \
  -H "Authorization: Bearer $TOKEN"
```

## PostgreSQL checks

Открыть `psql` внутри контейнера:

```bash
docker compose exec postgres psql -U task_planning -d task_planning
```

Проверить таблицы:

```bash
docker compose exec -T postgres psql -U task_planning -d task_planning \
  -c "\dt"
```

Проверить количество записей:

```bash
docker compose exec -T postgres psql -U task_planning -d task_planning \
  -c "SELECT 'users' AS table_name, COUNT(*) FROM users UNION ALL SELECT 'goals', COUNT(*) FROM goals UNION ALL SELECT 'tasks', COUNT(*) FROM tasks;"
```
