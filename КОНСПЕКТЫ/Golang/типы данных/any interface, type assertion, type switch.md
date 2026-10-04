`interface{}` это **пустой интерфейс**: интерфейс без методов, которому удовлетворяет любой тип. `any` (с Go 1.18) просто алиас: `type any = interface{}`. Разницы нет, в новом коде пишут `any`.

```go
var x any
x = 42
x = "hello"
x = []int{1, 2}
```
Внутри интерфейс хранит пару **(тип, значение)**. Статически вы видите только `any`, поэтому напрямую с содержимым работать нельзя, нужно его «достать».

Применение: `fmt.Println(a ...any)`, `json.Unmarshal` в `map[string]any` для неизвестной структуры, `context.WithValue`, контейнеры до дженериков.

Минусы: нет проверки типов на этапе компиляции, ошибки уходят в рантайм, возможны лишние аллокации. Если нужна универсальность с типобезопасностью, используйте **дженерики**.

### Type assertion

Позволяет достать конкретное значение из интерфейса: `x.(T)`.
```go
var x any = "hello"

s := x.(string)      // ок
n := x.(int)         // panic: interface conversion: interface {} is string, not int

n, ok := x.(int)     // безопасная форма (comma ok)
if !ok {
    // внутри был не int, n == 0
}
```

- Одно значение: при несовпадении **паника**.
- Форма `v, ok`: паники нет, при неудаче `v` получает zero value, `ok == false`.
- Можно проверять и на интерфейс: `if s, ok := x.(fmt.Stringer); ok { ... }`. Так проверяют, поддерживает ли значение дополнительное поведение (например, `http.Flusher`, `io.WriterTo`).
- Если сам интерфейс `nil`, assertion не сработает (в форме с `ok` вернёт `false`).

### Type switch
Удобная проверка сразу нескольких типов:
```go
func describe(x any) string {
    switch v := x.(type) {
    case nil:
        return "nil"
    case int:
        return fmt.Sprintf("int %d", v)        // v имеет тип int
    case string, []byte:
        return fmt.Sprintf("string-like %v", v) // при нескольких типах v остаётся any
    case error:
        return "error: " + v.Error()           // можно и интерфейсы
    case fmt.Stringer:
        return v.String()
    default:
        return fmt.Sprintf("other %T", v)
    }
}
```

**Особенности:**

- Запись `x.(type)` допустима **только** внутри `switch`.
- В каждой ветке `v` имеет тип этой ветки (кроме случая с перечислением нескольких типов и `default`, там `v` остаётся исходным типом интерфейса).
- Ветки проверяются **сверху вниз**, срабатывает первая подходящая. Поэтому более общий интерфейс лучше ставить после конкретных типов.
- `case nil` ловит именно nil-интерфейс.
- `fallthrough` в type switch запрещён.

Пример: работа с JSON неизвестной структуры
```go
var data map[string]any
json.Unmarshal([]byte(`{"a":1,"b":"x","c":[1,2]}`), &data)

for k, v := range data {
    switch t := v.(type) {
    case float64:          // числа в JSON становятся float64
        fmt.Println(k, "number", t)
    case string:
        fmt.Println(k, "string", t)
    case []any:
        fmt.Println(k, "array", len(t))
    case map[string]any:
        fmt.Println(k, "object")
    }
}
```

### Ловушки

- **Assertion без `ok`** на значении, в котором может быть другой тип, приводит к панике.
- **Ловушка с nil**: интерфейс с типизированным nil-указателем внутри не равен `nil`, и `case nil` его не поймает.
- **Assertion к неинтерфейсному типу** требует точного совпадения типа: `x.(int)` не сработает, если внутри `int64` или именованный тип `type MyInt int`.
- **Злоупотребление `any`** превращает Go в динамически типизированный язык и ухудшает читаемость. Лучше маленькие интерфейсы с методами или дженерики.