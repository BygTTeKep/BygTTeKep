Все методы есть в двух вариантах: обычный и с контекстом (`QueryContext`, `ExecContext` и т.д.). **Используйте варианты с `Context`**, чтобы запрос отменялся вместе с контекстом (клиент отключился, таймаут).

### `Exec`: запрос без строк результата

`INSERT`, `UPDATE`, `DELETE`, DDL.

```go
res, err := db.ExecContext(ctx,
    "UPDATE users SET name = $1 WHERE id = $2", name, id)
if err != nil {
    return err
}
n, _ := res.RowsAffected() // сколько строк затронуто
id, _ := res.LastInsertId() // поддерживают не все драйверы (MySQL да, pgx нет)
```

Для PostgreSQL получить id вставки: `INSERT ... RETURNING id` через `QueryRow`.

### `QueryRow`: одна строка

Возвращает `*Row`. Ошибка откладывается до `Scan`. Соединение освобождается после `Scan`.

```go
var u User
err := db.QueryRowContext(ctx,
    "SELECT id, name FROM users WHERE id = $1", id).
    Scan(&u.ID, &u.Name)

switch {
case errors.Is(err, sql.ErrNoRows):
    return nil, ErrNotFound // строки нет: это НЕ сбой
case err != nil:
    return nil, err
}
```

`sql.ErrNoRows` возвращается только у `QueryRow`. Если строк несколько, берётся первая, остальные отбрасываются.

### `Query`: много строк

Возвращает `*Rows`, по которым нужно итерироваться.

```go
rows, err := db.QueryContext(ctx, "SELECT id, name FROM users WHERE age > $1", 18)
if err != nil {
    return nil, err
}
defer rows.Close() // обязательно

var users []User
for rows.Next() {
    var u User
    if err := rows.Scan(&u.ID, &u.Name); err != nil {
        return nil, err
    }
    users = append(users, u)
}
if err := rows.Err(); err != nil { // ошибка, прервавшая итерацию
    return nil, err
}
```

**Обязательно:**

- `defer rows.Close()`: освобождает соединение (после полного прохода `Next` закрывается автоматически, но при раннем выходе нет).
- `rows.Err()` после цикла: `Next` возвращает `false` и при ошибке сети, и при конце данных.
- `Scan` требует **указатели** и порядок/количество, совпадающие с колонками.

**NULL:** в обычный `string` или `int` NULL не сканируется (ошибка). Варианты: `sql.NullString`, `sql.NullInt64`, `sql.Null[T]` (Go 1.22) или указатель (`*string`).