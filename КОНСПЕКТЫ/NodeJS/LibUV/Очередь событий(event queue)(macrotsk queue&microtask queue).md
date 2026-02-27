В него попадают события после выполнения определенных операций
События обрабатываются на определенной итерации event loop

Порядок выполнения кода
1) синхронный код
2) Микротаски(then, catch, finaly, Promise, async/await)
3) Макротаски(setTimeout, setInterval, I/O callbacks, setImmediate, requestAnimationFrame)
```js
async function foo() {
    console.log('A'); //2
    await Promise.resolve(
        await (async () => {
            console.log('resolve'); //3
            await Promise.resolve();
            console.log('resolve2'); //5
        })()
    );
    console.log('B'); //6
}
console.log('start'); //1
foo();
console.log('end'); //4
```
вывод
```txt
start
A
resolve
end
resolve2
B
```
**код после await никогда не выполняется в этом же стеке**, даже если промис уже resolved.
также существует microtask queue
## Микротаски (Microtask Queue)
- Есть **отдельная очередь для микротасок**, куда попадают:
    - `Promise.then / catch / finally`
    - `async/await` (код после `await`)
    - `queueMicrotask`
- **Микротаски выполняются сразу после текущего стека**, до макротасок (Event Queue).

при этом важно понимать что очередь там не одна, а на каждую фазу, микротаски обрабатываются  между фазами