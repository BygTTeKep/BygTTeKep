### `sql.ErrNoRows`

Возвращается из `Scan` после **`QueryRow`**, когда запрос не вернул ни одной строки. Это не сбой БД, а штатная ситуация «ничего не найдено», и её нужно отличать от настоящих ошибок.

```go
func (r *Repo) GetUser(ctx context.Context, id int) (*User, error) {
    var u User
    err := r.db.QueryRowContext(ctx,
        "SELECT id, name FROM users WHERE id = $1", id).
        Scan(&u.ID, &u.Name)

    switch {
    case errors.Is(err, sql.ErrNoRows):
        return nil, ErrNotFound // своя доменная ошибка
    case err != nil:
        return nil, fmt.Errorf("get user %d: %w", id, err)
    }
    return &u, nil
}
```

**Правила:**

- Проверять через `errors.Is`, а не сравнение чтобы работало с обёрнутыми ошибками.
- Конвертировать в **свою** ошибку (`ErrNotFound`) на границе репозитория: верхние слои не должны знать про `database/sql`.
- У `Query` (много строк) `ErrNoRows` не возникает: пустой результат это просто ноль итераций `rows.Next()`.
- `Exec` тоже не возвращает `ErrNoRows`: «ничего не обновилось» проверяют через `res.RowsAffected()`.
- У `pgx` (родной API) своя ошибка `pgx.ErrNoRows`. При использовании `pgx` через `database/sql` приходит `sql.ErrNoRows`.