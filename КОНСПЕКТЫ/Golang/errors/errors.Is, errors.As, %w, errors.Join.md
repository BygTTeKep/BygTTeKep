### Оборачивание (`%w`)

```go
err := fmt.Errorf("get user: %w", ErrNotFound)
```

Новая ошибка хранит исходную и имеет метод `Unwrap() error`. Так образуется цепочка. С `%v` текст тот же, но цепочка теряется.

### errors.Is

Проверяет, есть ли в цепочке ошибка, **равная** указанной (через олператор сравнения или через метод `Is(target error) bool`, если он определён):

```go
if errors.Is(err, ErrNotFound) {
    // 404
}

errors.Is(err, os.ErrNotExist) // работает и для ошибок стандартной библиотеки
errors.Is(err, context.DeadlineExceeded)
```

Сравнение `err == ErrNotFound` не увидит обёрнутую ошибку, поэтому используйте `errors.Is`.

### errors.As

Ищет в цепочке ошибку **определённого типа** и записывает её в переданную переменную. Нужен, когда важны поля ошибки:

```go
var ve *ValidationError
if errors.As(err, &ve) {
    fmt.Println(ve.Field, ve.Msg)
}
```

Второй аргумент должен быть **указателем** на тип, реализующий `error` (или на интерфейс). Если передать не указатель, будет паника (`go vet` это ловит).

### errors.Join (Go 1.20)

Объединяет несколько ошибок в одну. Полезно при параллельных задачах или валидации, когда нужно вернуть все проблемы сразу:

```go
err := errors.Join(err1, err2, err3) // nil-ошибки отбрасываются, если все nil, вернётся nil
errors.Is(err, err1) // true
```

Текст такой ошибки это тексты составляющих через перевод строки. Аналог: несколько `%w` в одном `fmt.Errorf` (тоже с Go 1.20).

### errors.Unwrap

Снимает один слой: `errors.Unwrap(err)`. В коде чаще используют `Is` и `As`, они идут по всей цепочке.

### Свой Is / Unwrap

```go
func (e *ValidationError) Is(target error) bool {
    t, ok := target.(*ValidationError)
    return ok && t.Field == e.Field
}
```

Если тип оборачивает другую ошибку, добавляют `Unwrap() error` (или `Unwrap() []error` для нескольких).
