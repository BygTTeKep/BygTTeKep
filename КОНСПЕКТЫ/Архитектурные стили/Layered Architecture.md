Layered Architecture - это организация кода по слоям ответственности с направленным потоком зависимостей
Классическая схема
```
UI -> Application -> Domain -> Infrastructure
```
Назначение
Решает задачу:
- разделение ответственности
- снижение связанности
- упрощение поддержки
# Типовые слои
## Presentation(UI)
- Controllers
- HTTP/GraphQL
- DTO
Нет бизнес логики
---
## Appication Layer
- Use cases
- Orchestration
- Transaction boundaries
Нет бизнес правил
Есть координация

---
## Domain Layer
- Aggregates
- Entities
- VO
- Domain Service
- Busines Invariants
Сердце системы
Не знает про БД, HTTP, фреймфворки

---

## Infrastructure Layer
- ORM
- DB
- Messaging
- External APIs
Не содержит бизнес логики

# Ограничения LA
- слои знают друг о друге
- Частые нарушения направление зависимостей
- 