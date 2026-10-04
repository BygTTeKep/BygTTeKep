errgroup можно сравнить с Promise.all в ноде, он также выполняет все горутины одновременно и если хоть одна из горутин падает с ошибкой отменяет все остальные горутины
сигнатура
```go
type Group struct {
	cancel func(error)
	wg sync.WaitGroup
	sem chan token
	errOnce sync.Once
	err error
}
```
метод для объявления самой группы
```go
func WithContext(ctx context.Context) (*Group, context.Context) {
	ctx, cancel := context.WithCancelCause(ctx)
	return &Group{cancel: cancel}, ctx
}
```
метод для запуска горутины в группе
```go
func (g *Group) Go(f func() error) {
	if g.sem != nil {
		g.sem <- token{}
	}
	g.wg.Add(1)
	go func() {
		defer g.done()
		if err := f(); err != nil {
			g.errOnce.Do(func() {
				g.err = err
				if g.cancel != nil {
					g.cancel(g.err)
				}
			})
		}
	}()
}
```
Важно использовать контекст который возращается при объявлении группы иначе, если допустим вы ходите в базу, запросы будут выполняться но результат их выполнения никому уже не нужен будет, это есть утекшие горутины

sem нужен для реализаци паттерна semaphore который позволяет ограничить одновременное выполннение горутин
чтобы ограничить кол-во параллельно выполняющихся горутин
```go
type token struct{}
func (g *Group) SetLimit(n int) {
	if n < 0 {
		g.sem = nil
		return
	}

	if active := len(g.sem); active != 0 {
		panic(fmt.Errorf("errgroup: modify limit while %v goroutines in the group are still active", active))
	}
	g.sem = make(chan token, n)
}
```

**`errgroup`** это пакет `golang.org/x/sync/errgroup`. Он расширяет идею `sync.WaitGroup`: запускает группу горутин, **ждёт их завершения** и **возвращает первую возникшую ошибку**. Вместе с контекстом умеет **отменять остальных** при сбое.

```bash
go get golang.org/x/sync/errgroup
```

Это не стандартная библиотека, а официальный подпроект `x/`.

### Базовое использование

```go
var g errgroup.Group

for _, url := range urls {
    g.Go(func() error {
        return fetch(url)
    })
}

if err := g.Wait(); err != nil {
    return err // первая ненулевая ошибка
}
```

- `g.Go(f)` запускает `f` в новой горутине. `f` имеет сигнатуру `func() error`.
- `g.Wait()` блокируется, пока все горутины не завершатся, и возвращает **первую** ненулевую ошибку (остальные теряются).
- Не нужны `Add` и `Done`: всё делает группа.

### `errgroup.WithContext`: отмена при ошибке

```go
g, ctx := errgroup.WithContext(parentCtx)

for _, id := range ids {
    g.Go(func() error {
        return process(ctx, id) // ctx отменится при первой ошибке
    })
}

if err := g.Wait(); err != nil {
    return err
}
```

Возвращаемый `ctx` **отменяется**, когда:

- любая функция вернула ошибку (первая),
- или `Wait` вернулся.

Остальные горутины должны **сами проверять `ctx`** (`ctx.Done()`, `QueryContext`, `NewRequestWithContext`). Группа их принудительно не останавливает.

Пример воркера, реагирующего на отмену:

```go
g.Go(func() error {
    for {
        select {
        case <-ctx.Done():
            return ctx.Err()
        case job, ok := <-jobs:
            if !ok {
                return nil
            }
            if err := handle(ctx, job); err != nil {
                return err
            }
        }
    }
})
```

Важно: после `Wait` этот `ctx` уже отменён, использовать его для дальнейшей работы нельзя, берите родительский.

### `SetLimit`: ограничение параллелизма

```go
g := new(errgroup.Group)
g.SetLimit(10) // не более 10 горутин одновременно

for _, item := range items { // хоть 100 000
    g.Go(func() error {
        return handle(item)
    })
}
err := g.Wait()
```

- Когда лимит достигнут, **`g.Go` блокируется**, пока не освободится слот. Это встроенный worker pool без ручных каналов и семафоров.
- `SetLimit` вызывают **до** первого `Go`. `-1` означает без лимита.
- `g.TryGo(f)` не блокируется: возвращает `false`, если слот занят, и функция не запускается.

### Паттерн: pipeline и сбор результатов

Результаты пишут в заранее выделенный слайс по индексу (без гонки, у каждой горутины своя ячейка) или в канал:

```go
results := make([]Result, len(ids))
g, ctx := errgroup.WithContext(ctx)

for i, id := range ids {
    g.Go(func() error {
        r, err := fetch(ctx, id)
        if err != nil {
            return err
        }
        results[i] = r // разные индексы, гонки нет
        return nil
    })
}
if err := g.Wait(); err != nil {
    return nil, err
}
return results, nil
```

### Сравнение с `WaitGroup`

| |`sync.WaitGroup`|`errgroup.Group`|
|---|---|---|
|Ожидание завершения|да|да|
|Возврат ошибки|нет|да (первая)|
|Отмена остальных при ошибке|нет|да (с `WithContext`)|
|Ограничение параллелизма|нет|да (`SetLimit`)|
|`Add` / `Done` вручную|да|не нужны|
|Где живёт|стандартная библиотека|`golang.org/x/sync`|
### Ловушки

- **Возвращается только первая ошибка.** Остальные теряются. Нужны все: собирайте сами (мьютекс/слайс) и объединяйте через `errors.Join`.
- **Отмена кооперативная.** Если горутина не смотрит на `ctx`, она продолжит работу, а `Wait` будет ждать её завершения.
- **Паника в горутине** группы роняет процесс. `errgroup` её не перехватывает (нужен свой `recover`, который превращает панику в `error`).
- **Захват переменной цикла** до Go 1.22: `i` и `v` в замыкании нужно копировать.
- **`g.Go` блокируется при `SetLimit`**, поэтому цикл запуска может подвиснуть, если слоты не освобождаются.
- **Нельзя переиспользовать `ctx` из `WithContext` после `Wait`**: он уже отменён.
- **Не вызывать `Go` после `Wait`** на той же группе, если нужна повторная работа, создайте новую группу.
- Если ни одна горутина не завершилась ошибкой, `Wait` возвращает `nil`, а `ctx` всё равно отменяется после `Wait`.

### Коротко для собеседования

- **`errgroup`** (`golang.org/x/sync/errgroup`) это `WaitGroup` с ошибками: `g.Go(func() error)` запускает горутину, `g.Wait()` ждёт всех и возвращает первую ошибку.
- **`errgroup.WithContext`** даёт `ctx`, который отменяется при первой ошибке, чтобы остальные могли остановиться (если проверяют `ctx`).
- **`SetLimit(n)`** ограничивает число одновременных горутин, `Go` при этом блокируется, `TryGo` нет.
- Минусы: только первая ошибка, отмена не принудительная, паники не перехватываются.