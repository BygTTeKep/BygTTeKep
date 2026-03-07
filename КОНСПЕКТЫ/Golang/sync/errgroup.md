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