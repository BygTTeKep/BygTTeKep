Единственная команда в redis stream которая добавляет данные в поток
```
XADD mystream * field1 value1 field2 value2

```
- `mystream` — имя потока.
- `*` — Redis сам сгенерирует уникальный ID.
- Далее идут пары `field value`, которые составляют данные сообщения.
