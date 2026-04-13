EventEmmiter - в nodejs это класс реализующий паттерн EventEmmiter.
В чем суть паттерна EventEmmiter - дать возможность с любого места в нашем приложении, сообщать о каком либо событии и соответственно реагировать на это событие.

# Методы
```js
// on регистрирует новый слушатель для события
// сигнатура
emmiter.on(evetName, handler)
// once регистрирует новый слушатель, но выполняет не более 1 раза
//сигнатура
emmiter.once(eventName, handler)

//emmit вызывает событие
// сигнатура
emmiter.emit(eventName)
```

# Пример собственнго решения
```ts
class SelfEmmiter {
	private events = new Map<string, Function[]>()
	on(name: string, func: Function) {
		const events = this.events.get(name) ?? []
		events.push(func)
		this.events.set(name, events)
	}
	emit(name: string, ...args: any) {
		const exists = this.events.get(name);
		if (exists?.length) {
			exists.forEach((d) => {d(...args)})
		}
	}
	once(name: string, func: Function) {
		this.on(name, (data: any)=>{func(data); this.events.delete(name)})
	}
}
```