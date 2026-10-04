ModuleRef - это мост к DI контейнеру NestJS, которыйй позволяет лениво загружать модули. Повзоляет разрешать циклические ависимости
Это сервис в NestJS который дает доступ к внутреннему контейнеру провайдеров модуля во время выполнения. Позволяет получать экземплеры провайдеров программно а не только через constructor injection

```ts
@Injectable()
export class ServiceA implements OnModuleInit {
  private serviceB: ServiceB;

  constructor(private moduleRef: ModuleRef) {}

  onModuleInit() {
    this.serviceB = this.moduleRef.get(ServiceB, { strict: false });
  }

  doSomething() {
    this.serviceB.doWork();
  }
}
```

| Метод                                  | Назначение                                                               |
| -------------------------------------- | ------------------------------------------------------------------------ |
| `get(token, options?)`                 | Синхронно получить существующий (singleton) провайдер                    |
| `resolve(token, contextId?, options?)` | Асинхронно получить request/transient-scoped провайдер (новый экземпляр) |
| `create(type)`                         | Создать инстанс класса без регистрации в модуле                          |
| `introspect(token)`                    | Узнать scope провайдера                                                  |