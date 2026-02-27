Преобразует простой объект в экземляр класса где будут работать декораторы class-transformer
```ts
const instance = plainToInstance(UpdateLeadWithStatusRequestDto, requestDto);
await validateOrReject(instance);
```

Создаёт `new Class()`, копирует поля, применяет декораторы