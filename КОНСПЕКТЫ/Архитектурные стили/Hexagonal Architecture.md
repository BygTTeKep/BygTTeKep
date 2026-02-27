
Ключевая идея
```
Domain/Application <-> Port <-> Adapter <-> Infrastructure
```

# Типы портов
## Input Ports (Driving)
- user cases
- Commands
- Queries
## Output Ports(Driven)
- Repositories
- Message Bus
- External services
# Адаптеры
## Driving Adapters
- HTTP controllers
- CLI
- Message consumers
## Driven adapters
- TypeORM repositories
- Kafka producers
- Rest clients