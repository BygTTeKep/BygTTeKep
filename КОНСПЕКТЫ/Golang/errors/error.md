error - это встроенный интерфейс с одним методов 
```go
type error interface {
	Error() string
}
```

Ошибки в Go это **обычные значения**. Функция возвращает `error` последним результатом, вызывающий код проверяет его явно. `nil` означает «ошибки нет».

### Способы создать ошибку

**1. `errors.New`** для простого текста:

```go
var ErrEmpty = errors.New("empty name")
```

**2. `fmt.Errorf`** для текста с форматированием и обёртыванием (`%w`):

```go
return fmt.Errorf("user %d: %w", id, ErrNotFound)
```

**3. Свой тип**, если нужны дополнительные данные:

```go
type ValidationError struct {
    Field string
    Msg   string
}

func (e *ValidationError) Error() string {
    return e.Field + ": " + e.Msg
}

func validate(name string) error {
    if name == "" {
        return &ValidationError{Field: "name", Msg: "required"}
    }
    return nil
}
```

Если у типа есть метод `Unwrap() error`, он участвует в цепочке ошибок (см. ниже).
### Ловушки

- **Nil-указатель в интерфейсе.** Если функция возвращает `error`, а внутри лежит типизированный nil (`var e *ValidationError; return e`), то `err != nil` будет `true`. Возвращайте именно `nil`.
- **Игнорирование ошибки** (`_ = f()`) допустимо только осознанно.
- **Не логировать и возвращать одновременно.** Либо обработайте на месте, либо верните наверх с контекстом, иначе одна ошибка попадёт в лог несколько раз.
- Текст ошибки по конвенции пишут **со строчной буквы и без точки в конце**, потому что он часто склеивается в цепочку: `open config: read file: permission denied`.