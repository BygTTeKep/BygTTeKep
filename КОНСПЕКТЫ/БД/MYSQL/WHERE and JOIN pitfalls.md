# WHERE
## WHERE-условия, которые «ломают» индексы
1) Функция над колонкой
```sql
SELECT *
FROM users
WHERE YEAR(created_at) = 2024;
```
**Проблема:**  
Индекс по `created_at` не используется → полный скан таблицы.
2) Приведение типов
```sql
WHERE user_id = '123'
```
если user_id INT
Проблема:
MySQL приводит колонку, а не значение → индекс может не использоваться.
3) LIKE с ведущим `%`
```sql
WHERE email LIKE '%@gmail.com'
```
Проблема:
Индекс бесполезен — поиск начинается не с начала строки.
4) `OR` вместо `UNION`
```sql
WHERE status = 'active'
   OR status = 'pending'
```
**Проблема:**  
Часто приводит к full scan.
вот это иногда быстрее
```sql
SELECT ... WHERE status = 'active'
UNION ALL
SELECT ... WHERE status = 'pending';
```
СОМНИТЕЛЬНО

---
# JOIN
1) `JOIN` без индекса
```sql
SELECT *
FROM orders o
JOIN users u ON o.user_id = u.id;
```
**Проблема:**  
Если `orders.user_id` или `users.id` без индекса → nested loop с огромной стоимостью.

2) LEFT JOIN + условие в WHERE (убивает LEFT JOIN)
```sql
SELECT *
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
WHERE o.status = 'paid';
```
**Проблема:**  
`LEFT JOIN` превращается в `INNER JOIN`.
Правильнее 
```sql
LEFT JOIN orders o
  ON o.user_id = u.id
 AND o.status = 'paid';
```
3) JOIN больших таблиц без фильтрации
```sql
FROM big_table_1 t1
JOIN big_table_2 t2 ON ...
WHERE t1.created_at > NOW() - INTERVAL 1 DAY;
```

**Проблема:**  
JOIN происходит **до** фильтрации (логически), если оптимизатор не сможет переупорядочить.
Лучше:
- фильтровать как можно раньше,
- использовать подзапросы или CTE.