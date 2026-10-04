**Миграция** это версионируемое изменение схемы БД (создать таблицу, добавить колонку, индекс). Хранятся в репозитории как файлы, применяются по порядку и фиксируются в служебной таблице, чтобы схема на всех окружениях была одинаковой и воспроизводимой.

Принципы:

- **Каждая миграция применяется один раз**, список применённых хранится в таблице (`schema_migrations` у golang-migrate, `goose_db_version` у goose).
- Миграция имеет направление **up** (применить) и **down** (откатить).
- **Применённые миграции не редактируют**: нужна новая миграция.
- Схема меняется миграциями, а не вручную и не `AutoMigrate`.
### golang-migrate

Отдельные файлы для up и down:

```
migrations/
  000001_create_users.up.sql
  000001_create_users.down.sql
  000002_add_email.up.sql
  000002_add_email.down.sql
```

```sql
-- 000001_create_users.up.sql
CREATE TABLE users (
    id   BIGSERIAL PRIMARY KEY,
    name TEXT NOT NULL
);

-- 000001_create_users.down.sql
DROP TABLE users;
```

CLI:

```bash
migrate create -ext sql -dir migrations -seq create_users
migrate -path migrations -database "$DSN" up
migrate -path migrations -database "$DSN" down 1
migrate -path migrations -database "$DSN" version
migrate -path migrations -database "$DSN" force 3   # сбросить "грязное" состояние
```

Можно запускать из кода (`migrate.New(...)`, `m.Up()`), в том числе с `embed.FS`.

Особенность: если миграция упала посреди выполнения, версия помечается как **dirty**, и нужно вручную починить схему и вызвать `force`.

### goose (`pressly/goose`)

Up и down в **одном файле**, разделены аннотациями:

```sql
-- +goose Up
CREATE TABLE users (
    id   BIGSERIAL PRIMARY KEY,
    name TEXT NOT NULL
);

-- +goose Down
DROP TABLE users;
```

CLI:

```bash
goose -dir migrations postgres "$DSN" create add_email sql
goose -dir migrations postgres "$DSN" up
goose -dir migrations postgres "$DSN" down
goose -dir migrations postgres "$DSN" status
```

Дополнительно у goose:

- **Миграции на Go** (когда нужна логика, а не только SQL: перенос и преобразование данных).
- Нумерация по времени или последовательная.
- Блоки для функций и триггеров: `-- +goose StatementBegin` / `StatementEnd`.
- Режим `NO TRANSACTION` для операций, которые нельзя выполнять в транзакции (`CREATE INDEX CONCURRENTLY`).
- Использование как библиотеки с `embed.FS`.

### Сравнение

|                           | golang-migrate                    | goose                   |
| ------------------------- | --------------------------------- | ----------------------- |
| Формат                    | два файла (up/down)               | один файл с аннотациями |
| Миграции на Go            | нет (только SQL, через код можно) | да                      |
| Поддержка источников      | файлы, embed, S3, GitHub и др.    | файлы, embed            |
| Грязное состояние (dirty) | есть, лечится `force`             | нет, статус по файлам   |
| Популярность              | очень высокая                     | высокая                 |

Ещё вариант: **Atlas** (декларативные миграции: описываете желаемую схему, инструмент сам считает разницу).

### Практика

- Каждая миграция **маленькая и атомарная**; лучше выполнять в транзакции (в PostgreSQL DDL транзакционный).
- Миграции **обратно совместимы** с работающей версией кода при выкатке: сначала добавить колонку, выкатить код, потом удалять старое (стратегия expand/contract). Иначе во время деплоя старые инстансы ломаются.
- Тяжёлые операции (`CREATE INDEX`, перенос данных на больших таблицах) делать осторожно: блокировки могут остановить сервис (`CREATE INDEX CONCURRENTLY`, батчи).
- Запускать миграции отдельным шагом деплоя (init-контейнер, job), а не в каждом инстансе приложения, чтобы избежать гонок. Инструменты используют advisory lock, но отдельный шаг надёжнее.
- Down-миграции для данных часто невозможны (удалённые данные не вернуть), поэтому в проде чаще «откат вперёд» новой миграцией.
- Хранить в Git вместе с кодом, прогонять в CI на чистой БД.
## Что такое `$DSN`

**DSN** (Data Source Name) это строка подключения к базе данных: адрес, порт, логин, пароль, имя БД и параметры.