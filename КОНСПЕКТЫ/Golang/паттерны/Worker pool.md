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