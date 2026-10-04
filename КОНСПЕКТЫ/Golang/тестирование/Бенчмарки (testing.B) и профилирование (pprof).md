### Бенчмарки

Функция `BenchmarkXxx(b *testing.B)` в `_test.go`. Цикл выполняется `b.N` раз; фреймворк подбирает `N`, чтобы замер был стабильным.

```go
func BenchmarkConcat(b *testing.B) {
    for i := 0; i < b.N; i++ {
        _ = strings.Repeat("a", 100) + "b"
    }
}
```

С Go 1.24 можно писать `for b.Loop() { ... }`: он сам управляет таймером и не даёт компилятору выбросить «неиспользуемый» вызов.

```bash
go test -bench=. -benchmem ./...        # все бенчмарки + аллокации
go test -bench=Concat -benchtime=3s     # дольше
go test -bench=. -count=10              # несколько прогонов для статистики
```

Вывод:

```
BenchmarkConcat-8    5000000    250 ns/op    112 B/op    2 allocs/op
```

`-8` это `GOMAXPROCS`, `ns/op` время на операцию, `B/op` байт на операцию, `allocs/op` число аллокаций (при `-benchmem`).
**Управление таймером:**

```go
func BenchmarkX(b *testing.B) {
    data := prepare()      // дорогая подготовка
    b.ResetTimer()         // не считать её
    for i := 0; i < b.N; i++ {
        process(data)
    }
}
```

Также `b.StopTimer()` / `b.StartTimer()`, `b.ReportAllocs()`, `b.SetBytes(n)` (покажет MB/s).

**Подбенчмарки и параметры:**

```go
for _, size := range []int{10, 1000, 100000} {
    b.Run(fmt.Sprintf("size=%d", size), func(b *testing.B) {
        for i := 0; i < b.N; i++ { work(size) }
    })
}
```

**Параллельные:** `b.RunParallel(func(pb *testing.PB) { for pb.Next() { ... } })`.

**Ловушки:**

- Компилятор может **выкинуть** вычисление, результат которого не используется: сохраняйте результат в глобальную переменную (`sink = result`).
- Подготовка внутри цикла искажает замер.
- Один прогон шумит (нагрев CPU, фоновые процессы). Делайте `-count=10` и сравнивайте через **`benchstat`** (`golang.org/x/perf/cmd/benchstat`): он показывает статистическую значимость разницы до/после оптимизации.
- Микробенчмарк не равен реальной нагрузке: проверяйте оптимизации на реалистичных данных.