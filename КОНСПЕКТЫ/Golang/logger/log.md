### Пакет `log`

Простой логгер из стандартной библиотеки:

```go
log.Println("started")
log.Printf("user %d created", id)
log.SetFlags(log.LstdFlags | log.Lshortfile) // дата, время, файл:строка
log.Fatal("boom")   // печатает и вызывает os.Exit(1), defer не выполнятся
log.Panic("boom")   // печатает и вызывает panic
```

Ограничения: **нет уровней** (debug/info/error), вывод только текстом, нет полей. Подходит для простых утилит, для сервисов недостаточно.