```
modules/leads/
├── application/
│   ├── dto/
│   ├── use-cases/
│   ├── services/
│   ├── mappers/
│   └── events/
│
├── domain/
│   ├── entities/
│   ├── repositories/
│   ├── services/
│   └── events/
│
├── infrastructure/
│   ├── repositories/
│   ├── mappers/
│   ├── persistence/
│   └── services/
│
└── presentation/
    ├── controllers/
    └── dto/
```
# Слой Domain
## Цель 
Хранить чистую бизнес логику без привязки к фреймворку,Бд или технологиям
Этот слой можно тестировать и использовать даже без NestJS
## Что содержит 
| Компонент                 | Назначение                                             | Пример                    |
| ------------------------- | ------------------------------------------------------ | ------------------------- |
| **Entities**              | Сущности с поведением и инвариантами                   | `Lead`, `User`, `Product` |
| **Value Objects**         | Объекты без идентичности (e.g. `Email`, `PhoneNumber`) |                           |
| **Domain Services**       | Правила, не привязанные к конкретной сущности          | `LeadAssignmentService`   |
| **Domain Events**         | События предметной области                             | `LeadAssignedEvent`       |
| **Repository Interfaces** | Контракты для хранения сущностей                       | `ILeadRepository`         |
Пример сущности
```ts
// domain/entities/lead.entity.ts
export class Lead {
  constructor(
    public readonly id: number,
    public name: string,
    public email: string,
    public phone: string,
    public status: LeadStatus,
  ) {}

  markAsConverted() {
    if (this.status === LeadStatus.CONVERTED) {
      throw new Error('Лид уже конвертирован');
    }
    this.status = LeadStatus.CONVERTED;
  }
}
```
# Слой INFRASTRUCTURE
Слой где домены, контракты реализуются технически(TypeORM, Redis, HTTP, MQ, Email, API и т.п.)
## Цель
Обеспечить работу доменов с внешним миром через адаптеры.
Инфраструктура никогда не содержит бизнес правила - только техническую реализацию
Что содежит

| Компонент                     | Назначение                                 | Пример                               |
| ----------------------------- | ------------------------------------------ | ------------------------------------ |
| **Repositories (реализации)** | Реализация `ILeadRepository` через TypeORM | `LeadTypeOrmRepository`              |
| **Persistence Entities**      | ORM-описания таблиц                        | `LeadOrmEntity`                      |
| **Mappers**                   | ORM ↔ Domain                               | `LeadOrmMapper`                      |
| **External Services**         | Интеграции (SMTP, Redis, Kafka и т.п.)     | `EmailSenderService`, `CacheService` |
Пример репозитория
```ts
// infrastructure/repositories/lead-typeorm.repository.ts
@Injectable()
export class LeadTypeOrmRepository implements ILeadRepository {
  constructor(
    @InjectRepository(LeadOrmEntity)
    private readonly repo: Repository<LeadOrmEntity>,
    private readonly mapper: LeadOrmMapper,
  ) {}

  async findById(id: number): Promise<Lead | null> {
    const entity = await this.repo.findOne({ where: { id } });
    return entity ? this.mapper.toDomain(entity) : null;
  }

  async save(lead: Lead): Promise<void> {
    const ormEntity = this.mapper.toOrmEntity(lead);
    await this.repo.save(ormEntity);
  }
}
```

# APPLICATION
Отвечает за выполнение конкретных use case. Управляет доменными  объектами и вызывает инфраструктуру
## Цель 
Инкапсулировать внешние сценарии использования домена= "что делает пользователь или система"
Что сожержит 

| Компонент                | Назначение                                   | Пример                                |
| ------------------------ | -------------------------------------------- | ------------------------------------- |
| **Use Cases**            | Реализация бизнес-сценариев                  | `AssignLeadUseCase`, `GetLeadUseCase` |
| **DTOs**                 | Входные и выходные данные                    | `AssignLeadDto`, `LeadResponseDto`    |
| **Application Services** | Служебная логика между несколькими use cases | `LeadAggregatorService`               |
| **Mappers**              | Domain ↔ DTO                                 | `LeadResponseMapper`                  |
| **Event Handlers**       | Реакции на domain events                     | `OnLeadAssignedHandler`               |
Пример Use case
```ts
@Injectable()
export class GetLeadUseCase {
  constructor(
    private readonly leadRepo: ILeadRepository,
    private readonly mapper: LeadResponseMapper,
  ) {}

  async execute(id: number): Promise<LeadResponseDto> {
    const lead = await this.leadRepo.findById(id);
    if (!lead) throw new NotFoundException('Lead not found');
    return this.mapper.toResponse(lead);
  }
}

```

Application orchestration:  
— достаёт данные через доменный контракт  
— управляет потоком  
— ничего не знает о БД, ORM и фреймворке.

# PRESENTATION
## Цель
Получить запрос → передать в use case → вернуть ответ.  
Минимум логики — максимум маршрутизации.
Что содержит 

| Компонент                         | Назначение                           | Пример                 |
| --------------------------------- | ------------------------------------ | ---------------------- |
| **Controllers / Resolvers**       | Входные точки API                    | `LeadsController`      |
| **DTOs / Validators**             | Валидация входных данных             | `CreateLeadRequestDto` |
| **Interceptors / Guards / Pipes** | Авторизация, преобразования, логгинг |                        |
Пример контроллера
```ts
@Controller('leads')
export class LeadsController {
  constructor(private readonly getLeadUseCase: GetLeadUseCase) {}

  @Get(':id')
  async getLead(@Param('id') id: number) {
    return this.getLeadUseCase.execute(id);
  }
}

```
# ПОТОК ДАННЫХ 

```
[HTTP Controller] → [UseCase] → [Domain Entity] → [Repository Interface]
                                               ↕
                                      [Repository Implementation]
                                               ↕
                                         [Database]

```
или 
```
Presentation → Application → Domain ←→ Infrastructure
```
**Главное правило зависимостей (Dependency Rule):**

> В DDD зависимости всегда направлены **вглубь**, к домену.
> 
> `Presentation → Application → Domain`
> 
> А `Infrastructure` — технический слой, который **подключается снизу** и **реализует интерфейсы домена**.










