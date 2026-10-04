В SQL любое поле может быть `NULL`, а `Scan` в обычный `string`/`int` вернёт ошибку (`converting NULL to string is unsupported`).

**Вариант 1: типы `sql.Null*`**

```go
var name sql.NullString
err := row.Scan(&name)

if name.Valid {
    fmt.Println(name.String)
} else {
    // в БД был NULL
}
```

Есть: `NullString`, `NullInt64`, `NullInt32`, `NullInt16`, `NullFloat64`, `NullBool`, `NullByte`, `NullTime`, а с Go 1.22 универсальный `sql.Null[T]`:

```go
var age sql.Null[int]
row.Scan(&age)
```

**Вариант 2: указатели**

```go
var name *string
row.Scan(&name)
if name == nil { /* NULL */ }
```

Проще и читается лучше, особенно в моделях для JSON.

**Вариант 3: `COALESCE` в SQL**

```sql
SELECT COALESCE(name, '') FROM users
```

Подходит, когда разница между `NULL` и пустым значением не важна.

### Особенности

- **JSON:** `sql.NullString` сериализуется как объект `{"String":"x","Valid":true}`, что обычно не нужно. Поэтому в API-моделях используют указатели или пишут свой `MarshalJSON`.
- **Время:** `time.Time` для nullable колонок заменяют на `sql.NullTime` или `*time.Time`.
- **`omitempty` и nil-указатель** хорошо сочетаются: `NULL` не попадёт в JSON.
- Свой тип с `Scan(src any) error` (`sql.Scanner`) и `Value() (driver.Value, error)` (`driver.Valuer`) позволяет хранить, например, JSON-поле или enum.