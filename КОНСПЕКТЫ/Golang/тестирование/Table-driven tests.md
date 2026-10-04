Идиоматичный стиль Go: набор случаев в слайсе структур и один цикл проверки. Новый случай добавляется одной строкой.

```go
func TestDivide(t *testing.T) {
    tests := []struct {
        name    string
        a, b    int
        want    int
        wantErr bool
    }{
        {name: "обычное деление", a: 10, b: 2, want: 5},
        {name: "деление на ноль", a: 1, b: 0, wantErr: true},
        {name: "отрицательные", a: -10, b: 2, want: -5},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got, err := Divide(tt.a, tt.b)
            if (err != nil) != tt.wantErr {
                t.Fatalf("err = %v, wantErr %v", err, tt.wantErr)
            }
            if got != tt.want {
                t.Errorf("got %d, want %d", got, tt.want)
            }
        })
    }
}
```
Плюсы: минимум дублирования, видно все сценарии сразу, легко добавлять граничные случаи. Для сравнения структур и слайсов используют `reflect.DeepEqual` или лучше `github.com/google/go-cmp/cmp` (`cmp.Diff` показывает, что именно различается).

### Subtests: `t.Run`

`t.Run(name, func(t *testing.T))` запускает подтест со своим именем.

- Можно запускать выборочно: `go test -run 'TestDivide/деление_на_ноль'` (пробелы заменяются на `_`).
- Падение подтеста отображается отдельно, остальные продолжаются.
- Внутри можно делать общую подготовку (setup) и `t.Cleanup`.

**Параллельные подтесты:**

```go
for _, tt := range tests {
    t.Run(tt.name, func(t *testing.T) {
        t.Parallel()
        // ...
    })
}
```

До Go 1.22 здесь требовалась копия `tt := tt` (замыкание над переменной цикла), с 1.22 не нужна.
### Прочее
- **Golden files**: ожидаемый вывод хранится в `testdata/` (имя каталога игнорируется сборкой).
- **`TestMain(m *testing.M)`**: общая подготовка на весь пакет (поднять БД, `os.Exit(m.Run())`).
- **Example-тесты** (`func ExampleSum()` с комментарием `// Output:`) одновременно документация и проверка.
- **Fuzzing** (Go 1.18): `FuzzXxx(f *testing.F)`, запуск `go test -fuzz=FuzzXxx`.
- **Интеграционные тесты**: отделяют build-тегом (`//go:build integration`) или `testing.Short()`; БД и сервисы поднимают через `testcontainers-go`.
- **`httptest`**: `httptest.NewRecorder()` для проверки хендлеров, `httptest.NewServer` для фейкового сервера.