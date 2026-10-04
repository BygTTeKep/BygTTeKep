 **Fan-out** раздаёт работу из одного канала **нескольким** горутинам (несколько читателей одного канала).
 fanout - Одна горутина отправляет задачи нескольким горутинам. Это позволяет распараллеливать вычисления, что полезно при работе с I/O операциями, загрузкой данных или обработкой запросов.

**Fan-in** собирает значения из **нескольких** каналов в один.
fanIn - Это обратный процесс. Когда несколько параллельно работающих горутин отправляют свои результаты в один канал, из которого читает главная горутина

```go
func worker(id int, jobs <-chan int, results chan<- int, activeWorkers *int32, wg *sync.WaitGroup) {
	defer wg.Done()
	for job := range jobs {
		time.Sleep(time.Duration(rand.Intn(200)) * time.Millisecond)
		fmt.Printf("Worker %d обработал задачу %d\n", id, job)
		results <- job * 2
	}
	atomic.AddInt32(activeWorkers, -1)
}

func FanInFanOut() {
	rand.Seed(time.Now().UnixNano())
	const numJobs = 50
	jobs := make(chan int, numJobs)
	results := make(chan int, numJobs)
	var wg sync.WaitGroup
	var activeWorkers int32 = 0
	go func() {
		for {
			time.Sleep(500 * time.Millisecond)
			if len(jobs) > 5 && atomic.LoadInt32(&activeWorkers) < 20 {
				wg.Add(1)
				atomic.AddInt32(&activeWorkers, 1)
				go worker(
					int(atomic.LoadInt32(&activeWorkers)),
					jobs,
					results,
					&activeWorkers,
					&wg,
				)
			}
		}
	}()
	for j := range numJobs {
		jobs <- j
	}
	close(jobs)
	for r := range results {
		fmt.Println(r)
	}
	wg.Wait()
	close(results)
}
```

**Схема:**

```
              ┌─ worker 1 ─┐
source ─────► ├─ worker 2 ─┤ ─────► merged result
 (fan-out)    └─ worker 3 ─┘  (fan-in)
```

**Ключевые моменты:**

- На каждый входной канал своя горутина, `out` закрывается после `wg.Wait()`.
- Порядок значений между источниками не гарантирован.
- Для двух каналов можно обойтись одним `select` в цикле с отключением закрытых веток через `nil` (см. раздел про `select`).
- Без проверки `ctx.Done()` при отправке в `out` горутины утекут, если потребитель ушёл раньше.