# Что такое scope в NestJS
Scope - это область жизни(life cycle) экземляра провайдера(@Injectable класс)
Scope управляет тем, когда и как часто NestJS создает новый экземпляр этого класса

# виды scope
- DEFAULT(SINLETON) - общий экземляр на все приложение, создается один раз при запуске(по умолчани.)
- REQUEST - создает новый экземпляр на каждый запрос, изолирует данные пользователя
- TRANSIENT - создает новый экземляр при каждой инъекции(Каждый @Inject() или new), у него очент короткий жизненый цикл.
# Принцип работы DI контейнера
NestJS использует - Dependency injection контейнер он управляет создание объектов и их зависимостями.
```js
@Injectable()
export class MyService {
  constructor(private readonly repo: Repository<UserEntity>) {}
}

```
NestJS ищет зарегистрированый провайдер для `Repository<UserEntity>` и внедряет его

 >**Scope** управляет тем, к какому контейнеру относится провайдер:
> - Singleton → глобальный контейнер
 - Request → контейнер запроса
 >- Transient → временный контейнер (уничтожается сразу)

# ПРИМЕРЫ
## Singleton (Scope.DEFAULT)
```ts
@Injectable()
export class AppService {
  private count = 0;
  increment() {
    this.count++;
    return this.count;
  }
}
```

Все контроллеры и модули получат **один и тот же экземпляр**:
```txt
controller1 -> AppService (один экземпляр)
controller2 -> AppService (тот же экземпляр)

```
**Использование:**  
Большинство сервисов, мапперов, утилит, репозиториев.

## Request Scope
```ts
@Injectable({ scope: Scope.REQUEST })
export class UserContextService {
  constructor(@Inject(REQUEST) private readonly request: Request) {}

  getUser() {
    return this.request.user;
  }
}

```
Создаётся **новый экземпляр на каждый запрос**:
```txt
GET /users   → новый экземпляр UserContextService
GET /leads   → новый экземпляр UserContextService
```
**Использование:**  
Когда сервис хранит данные, уникальные для текущего запроса (например, `user`).
**Особенности:**
- Все зависимости, которые он инжектирует, **также должны быть request-scoped или transient**, иначе Nest не сможет их разрешить.
- Работает немного медленнее (создаются новые объекты).
## Transient Scope
```ts
@Injectable({ scope: Scope.TRANSIENT })
export class LoggerService {
  constructor() {
    console.log('Создан новый экземпляр LoggerService');
  }
}

```
Создаётся **каждый раз, когда кто-то его инжектирует**:
```txt
controller1 -> LoggerService (новый экземпляр)
controller2 -> LoggerService (новый экземпляр)
serviceA -> LoggerService (новый экземпляр)
```
**Использование:**  
Редко. Обычно для легковесных, полностью изолированных, stateless утилит.  
Например — генерация случайных данных, временные билдеры, короткоживущие фабрики.

# Почему `Scope.TRANSIENT` ломает `@InjectRepository()`****
### Проблема:

`@InjectRepository()` создаёт **singleton-прокси TypeORM репозитория**, привязанную к `DataSource`.
```ts
@Injectable({ scope: Scope.TRANSIENT })
export class GetLeadsMapper {
  constructor(
    @InjectRepository(ProductEntity)
    private readonly productRepository: Repository<ProductEntity>,
  ) {}
}

```
— Nest должен создать новый transient-инстанс **без доступа к контексту TypeORM** (singleton контейнеру), поэтому:
- DI-контейнер **не находит подходящий репозиторий**,
- Инъекция завершается с пустым объектом `{}`,
- Методы `find()`, `createQueryBuilder()` становятся `undefined`.

**Корень проблемы:** transient-сервисы не "видят" singleton-провайдеров TypeORM в момент разрешения зависимостей.

> **Если `scope` у зависимостей "более короткий", чем у родителя — возможны ошибки DI.**

> Если родительский провайдер scope.default то все дочернии провайдеры будут созданы один раз как родительский провайдер, вне зависимости от их scope, так работает nestjs DI контейнер.