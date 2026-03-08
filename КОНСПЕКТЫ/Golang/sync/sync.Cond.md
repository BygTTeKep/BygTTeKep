syncCond - Это condition variable координирующая горутины. Она позволяет одним горутинам ждать изменения какогото состояния, а другим сигнализировать что состояние поменялось
Сигнатура
```go
type Cond struct {
	noCopy noCopy

	L Locker
	notify notifyList
	checker copyChecker
}

type copyChecker uintptr
```
где 
- Locker это обычно Mutex или RWMutex
- notifyList это список ожидающих горутин
- copyChecker это указатель на самого себя, чтобы можно было определить копирование

Cond работает напрямую с runtime sheduler'ом
при вызове Wait() горутина паркуется

При вызове signal горутина снова продолжает свою работу

при вызове broadcast все горутины становятся runnable, но не все сразу выполняются т.к. берется лок одной из горутины

Пример использования
```go
var (
	queues []int
	cond = cond.NewCond(&sync.Mutex)
)
func producer() {
	for i := range 5 {
		cond.L.Lock()
		queues = append(queues, i)
		cond.Signal()
		cond.L.Unlock()
	}
}

func consumer() {
	for {
		cond.L.Lock()
		for len(queues) == 0 {
			cond.Wait()
		}
		item := queues[0]
		queues = queues[1:]
		fmt.Println("consumed: ", item)
		cond.L.Unlock()
	}
}
```
В большинстве случаев будет достаточно использовать каналы для координации горутин

Lock нужен для избежания Lost wakeup 
пример
```
1) консюмер проверил len(uqeues) == 0
2) продюсер закинул данные
3) продюсер вызвал Signal
4) консюмер вызвал Wait()
```
получается ситуация что консюмер уснул когда в очереди были добавленны данные 