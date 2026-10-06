**Reflector** - это встроенный сервис, который читает **метаданные**, прикреплённые к классам и методам через декораторы. Он нужен, чтобы guard, interceptor или pipe могли узнать, какие "пометки" вы поставили на контроллере или хендлере.

### Зачем это нужно

Guard (или interceptor) выполняется до хендлера и ничего не знает о нём. Но часто нужно, чтобы поведение зависело от того, что вы пометили декоратором: "этот роут публичный", "сюда допускаются только админы". Декоратор записывает метаданные, а Reflector их читает.

### Пример: роли

**1. Декоратор, который пишет метаданные:**

```ts
import { SetMetadata } from '@nestjs/common';

export const ROLES_KEY = 'roles';
export const Roles = (...roles: string[]) => SetMetadata(ROLES_KEY, roles);
```

**2. Использование на роуте:**

```ts
@Controller('users')
export class UsersController {
  @Roles('admin')
  @Get()
  findAll() { /* ... */ }
}
```

**3. Guard, который читает метаданные через Reflector:**

```ts
import { CanActivate, ExecutionContext, Injectable } from '@nestjs/common';
import { Reflector } from '@nestjs/core';

@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const roles = this.reflector.getAllAndOverride<string[]>(ROLES_KEY, [
      context.getHandler(), // метод
      context.getClass(),   // контроллер
    ]);

    if (!roles) return true; // метка не стоит, пускаем

    const { user } = context.switchToHttp().getRequest();
    return roles.includes(user.role);
  }
}
```

### Методы Reflector

|Метод|Что делает|
|---|---|
|`get(key, target)`|Читает метаданные с одной цели (метод или класс)|
|`getAll(key, targets)`|Возвращает массив значений со всех целей|
|`getAllAndOverride(key, targets)`|Берёт первое найденное значение, **метод перекрывает класс**|
|`getAllAndMerge(key, targets)`|Объединяет значения с метода и класса (массивы склеиваются)|

Чаще всего используют `getAllAndOverride`: можно поставить `@Roles('user')` на весь контроллер, а на конкретном методе переопределить `@Roles('admin')`.

### Типичный кейс: `@Public()`

Глобальный auth guard на всё приложение, а некоторые роуты должны быть открытыми:

```ts
export const IS_PUBLIC_KEY = 'isPublic';
export const Public = () => SetMetadata(IS_PUBLIC_KEY, true);

@Injectable()
export class AuthGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext) {
    const isPublic = this.reflector.getAllAndOverride<boolean>(IS_PUBLIC_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);
    if (isPublic) return true;

    // ...проверка токена
  }
}
```

```ts
@Public()
@Post('login')
login() { /* ... */ }
```

### Типизированный декоратор (альтернатива `SetMetadata`)

```ts
export const Roles = Reflector.createDecorator<string[]>();

// на роуте
@Roles(['admin'])

// в guard
const roles = this.reflector.get(Roles, context.getHandler());
```

Так ключ-строка не нужна, и типы выводятся автоматически.

### Коротко

- `SetMetadata` / кастомный декоратор **записывает** данные на класс или метод.
- `Reflector` **читает** их в guard, interceptor и т.д.
- Это основной механизм для декларативных ролей, прав, публичных роутов, настроек кеша и т.п.