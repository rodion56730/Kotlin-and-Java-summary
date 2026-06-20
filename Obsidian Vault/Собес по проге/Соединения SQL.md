
Select -  выводит выборки 
Нужны для соединения информации из двух и более таблиц 
Ограничения выборки
CROSS JOIN - никак не ограничивает каждый с каждым
```sql
select * from product CROSS JOIN category;
--вернет все из двух таблиц
```
INNER JOIN - удовлетворяет некоторому условию 
```sql
select * from product inner join category on product.category_id = category.category_id;
-- вернет таблицу продуктов и соотнесет ее с таблицей категорий , выведутся продукты и категории к которым они относятся
```
LEFT OUTER JOIN и RIGHT OUTER JOIN - удовлетворяет левому или правому внешнему соединению 
```sql
select * from category left outer join product on product.category_id = category.category_id; 
--результат - кортежи из внутреннего соединения источников и не вошедшие во внутренне соединение кортежи левого или правого источника
```
FULL OUTER JOIN - полное внешнее соединение - то же самое только включает и левые и правые

NATURAL JOIN - Естественное соединение ()

union - соединяет выборки и возвращает только уникальные выборки
all - возвращает c повторами
```sql
select * from product where price > 900;
union
select * from product where price < 100;
--вернет только результаты без повторов
```
отличия с join в том , что union соединяет без смыссленно , а join ищет вхождения данных из одной таблицы в другую
## GROUP BY - группировка данных

**GROUP BY** - используется для группировки строк с одинаковыми значениями в указанных столбцах, чтобы выполнить агрегирующие функции (COUNT, SUM, AVG, MAX, MIN) для каждой группы.

```sql

SELECT column1, COUNT(column2) 
FROM table 
GROUP BY column1;
```

### Основные принципы:

**1. Группировка по одному столбцу:**

```sql

SELECT category_id, COUNT(*) as product_count
FROM product
GROUP BY category_id;


-- результат: количество товаров в каждой категории
```


**2. Группировка по нескольким столбцам:**

```sql

SELECT category_id, supplier_id, AVG(price) as avg_price
FROM product
GROUP BY category_id, supplier_id;

-- результат: средняя цена для каждой комбинации категория-поставщик
```
### Важные правила:

**Правило агрегации:** Все неагрегированные столбцы в SELECT должны быть указаны в GROUP BY.

```sql

-- Правильно:
SELECT first_name, last_name, COUNT(*) 
FROM clients 
GROUP BY first_name, last_name;

-- Ошибка (first_name не в GROUP BY):
SELECT first_name, last_name, COUNT(*) 
FROM clients 
GROUP BY last_name;
```

### Примеры использования:

**Подсчет заказов по клиентам:**

```sql

SELECT client_id, COUNT(*) as order_count
FROM sale
GROUP BY client_id;
```
**Сумма продаж по статусам:**

```sql

SELECT status_id, SUM(sale_sum) as total_sales
FROM sale
GROUP BY status_id;
```
**Группировка с фильтрацией (HAVING):**

```sql

SELECT category_id, COUNT(*) as product_count
FROM product
GROUP BY category_id
HAVING COUNT(*) > 5;  -- только категории с более чем 5 товарами
```

### Ключевые моменты:

- GROUP BY создает группы строк с одинаковыми значениями
    
- Агрегирующие функции работают внутри каждой группы
    
- HAVING фильтрует результаты после группировки (в отличие от WHERE, который фильтрует до группировки)
