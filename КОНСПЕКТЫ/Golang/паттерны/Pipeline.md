**Идея:** цепочка стадий, где каждая стадия принимает канал, обрабатывает значения и отдаёт новый канал. Стадии работают **одновременно**: пока вторая обрабатывает значение, первая уже готовит следующее.
```go
func generate(ctx context.Context, nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for _, n := range nums {
            select {
            case out <- n:
            case <-ctx.Done():
                return
            }
        }
    }()
    return out
}

func square(ctx context.Context, in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for n := range in {
            select {
            case out <- n * n:
            case <-ctx.Done():
                return
            }
        }
    }()
    return out
}

// использование
ctx, cancel := context.WithCancel(context.Background())
defer cancel() // отмена останавливает всю цепочку

for v := range square(ctx, square(ctx, generate(ctx, 1, 2, 3))) {
    fmt.Println(v)
}
```

**Правила стадии:**

1. Принимает `<-chan T`, возвращает `<-chan U` (направленные каналы).
2. **Закрывает свой выходной канал** (`defer close(out)`), когда входной закрыт или отменён контекст.
3. Каждая отправка защищена `select` с `ctx.Done()`.

**Ключевые моменты:**

- Закрытие канала «протекает» по цепочке: закрылся источник, затем завершается каждая следующая стадия.
- Если потребитель вышел раньше, без `ctx` все стадии зависнут на отправке (утечка). Поэтому `defer cancel()`.
- Медленную стадию распараллеливают fan-out'ом (несколько горутин читают входной канал) и склеивают fan-in'ом.
- Буферы между стадиями сглаживают разницу в скорости, но не убирают узкое место.