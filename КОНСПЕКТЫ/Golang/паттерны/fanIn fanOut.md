 fanout - Одна горутина отправляет задачи нескольким горутинам. Это позволяет распараллеливать вычисления, что полезно при работе с I/O операциями, загрузкой данных или обработкой запросов.
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