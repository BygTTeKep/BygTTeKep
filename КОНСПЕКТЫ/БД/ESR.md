ESR Rule: Equality, Sort, Range

Правило составления составного индекса

E - EQUALITY(равенство) — колонки по которым происходит равенство должны стоять первыми

тоесть это `where id = …` связанно это с тем что бд быстро находит такие строки

S - SORT(СОРТИРОВКА) — колонки по которым выполняется сортировка должны стоять сразу после колонок для равенства

RANGE(ДИАПАЗОН) — колонки по которым происходит фильтрация с использованием диапазона `where id BETWEEN 1 AND 2` ИЛИ `WHERE id > 2` должны стоять самыми последними

Например

```JavaScript
SELECT * FROM Sales
WHERE Region = 'North' 
AND Date BETWEEN '2023-01-01' AND '2023-01-31'
ORDER BY Amount;


CREATE INDEX idx_sales_region_date_amount 
ON Sales (Region, Date, Amount);
```