**CTE (Common Table Expression)** — это временный результирующий набор, определённый внутри SQL-запроса с помощью конструкции `WITH`.  
Он существует только в рамках одного запроса и позволяет писать более чистый, читаемый и структурированный SQL.

```sql
WITH cte_name AS (
    SELECT ...
)
SELECT *
FROM cte_name;

```

в typeorm
```ts
const users = await dataSource
  .createQueryBuilder()
  .addCommonTableExpression(
    `
      SELECT id, name
      FROM user
      WHERE isActive = true
    `,
    "active_users"
  )
  .from("active_users", "u")
  .select("u.*")
  .getRawMany();

```
# ❗ В MySQL CTE = _Derived Table_ (почти всегда)

В отличие от PostgreSQL/SQL Server, **MySQL не оптимизирует CTE более эффективно, чем подзапросы**.

**До MySQL 8.0.14 CTE всегда материализовались → были медленнее.**  
Сейчас MySQL может inline'ить CTE, но:

- если CTE используется **один раз** (как у тебя)
- если он не рекурсивный
- и если его можно inline’ить

→ **MySQL просто заменяет CTE на Derived Table**.

**EXPLAIN показывает это: `<derived2>`**  
Это значит: MySQL _взял твой CTE и превратил его в derived table (подзапрос)_.

