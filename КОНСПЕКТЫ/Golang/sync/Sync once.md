Это примитив синхроназации который гарантирует что функция будет вызвана только 1 раз
пример
```go
var once sync.Once

func initConfig(){
	fmt.Println("init config")
}

func main(){
	for i:=0; i<5; i++ {
		go func(){
			once.Do(initCOnfig)
		}
	}
	select{}
}
```
В примере выше "initConfig" выведится только 1 раз