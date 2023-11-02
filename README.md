# Домашнее задание к занятию «Индексы» - Стрельцов Владимир

## Задание 1
Напишите запрос к учебной базе данных, который вернёт процентное отношение общего размера всех индексов к общему размеру всех таблиц.

```
SELECT  SUM(DATA_LENGTH) as 'Length Tables', SUM(INDEX_LENGTH) AS 'Length Indexes',
CONCAT(  ROUND ((SUM(INDEX_LENGTH)*100 / SUM(DATA_LENGTH))), ' %')  AS '% Indexes'
FROM INFORMATION_SCHEMA.TABLES
WHERE  TABLE_SCHEMA = 'sakila';
```
![img](/img/2023-11-02_11-54-56.png)

## Задание 2
Выполните explain analyze следующего запроса:
```
select distinct concat(c.last_name, ' ', c.first_name), sum(p.amount) over (partition by c.customer_id, f.title)
from payment p, rental r, customer c, inventory i, film f
where date(p.payment_date) = '2005-07-30' and p.payment_date = r.rental_date and r.customer_id = c.customer_id and i.inventory_id = r.inventory_id
```
### перечислите узкие места;

**Узкое место в бессмысленной обработке таблиц rental, film, inventory. Все требуемые данные находятся в payment и customer.**

### оптимизируйте запрос: внесите корректировки по использованию операторов, при необходимости добавьте индексы.
```
select distinct concat(c.last_name, ' ', c.first_name), sum(p.amount) over (partition by c.customer_id)
from payment p, customer c
where date(p.payment_date) = '2005-07-30' and p.customer_id = c.customer_id;
```

### До оптимизации
```
-> Limit: 200 row(s)  (cost=0..0 rows=0)
    -> Table scan on <temporary>  (cost=2.5..2.5 rows=0)
        -> Temporary table with deduplication  (cost=0..0 rows=0)
            -> Window aggregate with buffering: sum(payment.amount) OVER (PARTITION BY c.customer_id,f.title ) 
                -> Sort: c.customer_id, f.title
                    -> Stream results  (cost=23.3e+6 rows=17.1e+6)
                        -> Nested loop inner join  (cost=23.3e+6 rows=17.1e+6)
                            -> Nested loop inner join  (cost=21.6e+6 rows=17.1e+6)
                                -> Nested loop inner join  (cost=19.9e+6 rows=17.1e+6)
                                    -> Inner hash join (no condition)  (cost=1.65e+6 rows=16.5e+6)
                                        -> Filter: (cast(p.payment_date as date) = '2005-07-30')  (cost=1.94 rows=16500)
                                            -> Table scan on p  (cost=1.94 rows=16500)
                                        -> Hash
                                            -> Index scan on f using idx_title  (cost=112 rows=1000)
                                    -> Covering index lookup on r using rental_date (rental_date=p.payment_date)  (cost=1 rows=1.04)
                                -> Single-row index lookup on c using PRIMARY (customer_id=r.customer_id)  (cost=0.001 rows=1)
                            -> Single-row covering index lookup on i using PRIMARY (inventory_id=r.inventory_id)  (cost=0.001 rows=1)
```
### После
```
-> Limit: 200 row(s)  (cost=0..0 rows=0)
    -> Table scan on <temporary>  (cost=2.5..2.5 rows=0)
        -> Temporary table with deduplication  (cost=0..0 rows=0)
            -> Window aggregate with buffering: sum(payment.amount) OVER (PARTITION BY c.customer_id ) 
                -> Sort: c.customer_id
                    -> Stream results  (cost=7449 rows=16500)
                        -> Nested loop inner join  (cost=7449 rows=16500)
                            -> Filter: (cast(p.payment_date as date) = '2005-07-30')  (cost=1674 rows=16500)
                                -> Table scan on p  (cost=1674 rows=16500)
                            -> Single-row index lookup on c using PRIMARY (customer_id=p.customer_id)  (cost=0.25 rows=1)
```
