это структура данных нужна для того чтобы избегать гонки при таких операциях как i++
когда несколько потоков пишут данные, основная идея заключается в том чтобы в каждой операции записи данных проверялось предыдущие значение тоесть в односвязаном списке это выглядит так

есть два потока A и B
поток A пишет 1
поток B пишет 2
поток B обогнал A и получили ситуацию когда значение переменой i не 2 а 1, в случае с этой структурой данных такое невозможно за счет того что мы сравниваем старое значение с head и в случае если оно отличается ретраим с новыым сначением head

Реализация 
```ts
class NodeLFS<T> {
  constructor(
    public value: T,
    public next: NodeLFS<T> | null = null,
  ) {}
}

class LockFreeStack<T> {
	private head: NodeLFS<T> | null = null
	push(value: T): void{
		const node = new NodeLFS(value);

		do {
			node.next = this.head
		} while(!this.cas(node.next, node))
	}
	pop(): T | null{
		let oldhead: NodeLFS<T> | null
		do{
			oldhead = this.head
			if(!oldhead) return null
		} while(!this.cas(oldhead, oldhead?.next))
		return oldhead.value
	}
	cas(expected: NodeLFS<T> | null, update: NodeLFS<T> | null) {
		console.log(this.head)
		if (this.head?.value === expected?.value) {
			this.head = update
			return true
		}
		console.log('race condition. retry')
		return false
	}
	read() {
		let node = this.head
		while(node !== null){
			console.log(node?.value)
			node = node?.next
		}
	}
}

```
стоит обратить внимание на опрядок в CAS(compare and swap(важно понимать что cas это операция процессора и должна быть встроенна в яп)) в push и pop они отличаются
актуальное значение всегда будет в head

тоесть при в ставке мы посути в head пишем тукущее значение а next будет head

а при pop получем разрываем связь с "old" head

РАБОТАЕТ ПО ПРИНЦИПУ LIFO