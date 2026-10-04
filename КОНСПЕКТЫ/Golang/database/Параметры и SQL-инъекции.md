Всегда передавайте значения **параметрами** (`$1` в PostgreSQL, `?` в MySQL/SQLite), а не склеивайте строку:

```go
// плохо: SQL-инъекция
db.Query("SELECT * FROM users WHERE name = '" + name + "'")

// хорошо
db.QueryContext(ctx, "SELECT * FROM users WHERE name = $1", name)
```

Параметризация не работает для **имён таблиц и колонок**. Их нужно валидировать по белому списку.