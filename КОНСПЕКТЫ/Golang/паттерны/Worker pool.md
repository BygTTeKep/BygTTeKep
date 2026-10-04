Это патерн позволяющий ограничить количество горутин которые будут выполняться параллельно, главное отличие от семафоров заключается в том что воркер пул фиксирует N-количество горутин, а семафор говорит о том что одновременно может выполняться от 0 до N горутин

простой пример
```go
package main

import (
	"fmt"
	"time"
)

type Job struct {
	id int
}

func worker(id int, jobs <-chan Job) {
	for job := range jobs {
		fmt.Printf("Worker %d started job %d\n", id, job.id)
		time.Sleep(time.Second) // имитация работы
		fmt.Printf("Worker %d finished job %d\n", id, job.id)
	}
}

func main() {
	const numWorkers = 3
	const numJobs = 10

	jobs := make(chan Job, numJobs)

	// запускаем воркеров
	for w := 1; w <= numWorkers; w++ {
		go worker(w, jobs)
	}

	// отправляем задачи в канал
	for j := 1; j <= numJobs; j++ {
		jobs <- Job{id: j}
	}
	close(jobs)

	// ждем, чтобы воркеры завершили
	time.Sleep(5 * time.Second)
}
```

**Ключевые моменты:**
- `jobCh` закрывает продюсер, `resCh` закрывает отдельная горутина после `wg.Wait()`.
- Воркеры завершаются сами, когда `jobCh` закрыт и пуст (`range`).
- Результаты приходят в **произвольном порядке**. Нужен порядок: передавайте индекс вместе с задачей и пишите в `results[i]`.
- Размер пула подбирают под тип нагрузки: для CPU-bound около `GOMAXPROCS`, для I/O-bound больше.
- Буфер у `jobCh` сглаживает всплески, но не заменяет решение проблемы обратного давления.
**Короткая альтернатива:** `errgroup` с `SetLimit(n)` (см. прошлый ответ) даёт тот же эффект без ручных каналов, плюс ошибки и отмену.

```go
func workerPool(ctx context.Context, jobs []int, n int) []int {
    jobCh := make(chan int)
    resCh := make(chan int)

    var wg sync.WaitGroup
    for range n {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for j := range jobCh {
                select {
                case resCh <- process(j):
                case <-ctx.Done():
                    return
                }
            }
        }()
    }

    // продюсер
    go func() {
        defer close(jobCh)
        for _, j := range jobs {
            select {
            case jobCh <- j:
            case <-ctx.Done():
                return
            }
        }
    }()

    // закрываем результаты, когда все воркеры закончили
    go func() {
        wg.Wait()
        close(resCh)
    }()

    var out []int
    for r := range resCh {
        out = append(out, r)
    }
    return out
}
```