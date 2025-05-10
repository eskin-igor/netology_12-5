# netology_12-5

# Домашнее задание к занятию «Индексы»

## Задание 1

Напишите запрос к учебной базе данных, который вернёт процентное отношение общего размера всех индексов к общему размеру всех таблиц.

## Решение 1 
```
SELECT ROUND((SUM(index_length) / (SUM(data_length) + SUM(index_length))) * 100, 2) AS 'Общий размер всех индексов к общему размеру всех таблиц, %', SUM(index_length) AS 'Общий размер всех индексов, бит', SUM(data_length)+SUM(index_length) AS 'Общий размер всех таблиц, бит'
FROM information_schema.tables
WHERE information_schema.tables.table_schema = 'sakila';
```
![](https://github.com/eskin-igor/netology_12-5/blob/main/12-5/12-05-1-1.JPG)

## Задание 2

Выполните explain analyze следующего запроса:
```
select distinct concat(c.last_name, ' ', c.first_name), sum(p.amount) over (partition by c.customer_id, f.title)
from payment p, rental r, customer c, inventory i, film f
where date(p.payment_date) = '2005-07-30' and p.payment_date = r.rental_date and r.customer_id = c.customer_id and i.inventory_id = r.inventory_id
```
* перечислите узкие места;
* оптимизируйте запрос: внесите корректировки по использованию операторов, при необходимости добавьте индексы.

## Решение 2

Результат запроса без анализа.

![](https://github.com/eskin-igor/netology_12-5/blob/main/12-5/12-05-2-1.JPG)

Результат запроса с  анализом.
```
EXPLAIN ANALYZE
select distinct concat(c.last_name, ' ', c.first_name), sum(p.amount) over (partition by c.customer_id, f.title)
from payment p, rental r, customer c, inventory i, film f
where date(p.payment_date) = '2005-07-30' and p.payment_date = r.rental_date and r.customer_id = c.customer_id and i.inventory_id = r.inventory_id
```
![](https://github.com/eskin-igor/netology_12-5/blob/main/12-5/12-05-2-2.JPG)

По выводу команды обнаружены следующие узкие места:   
1. Дублирующие данные (Temporary table with deduplication). Была создана временная таблица с удалёнными дублирующими данными. На что было потрачено очень много времени (actual time=30376..30376).
2. Отсутствие группировки. Использованы оконные функции (Window aggregate with buffering) вместо группировки. Затрачено времени (actual time=13760..29069). Если использовать группировку, то мы получим уменьшение количества строк и соответственно уменьшение времени обработки.
3. Сортировка (Sort) по двум полям c.customer_id, f.title. Затрачено времени (actual time=13760..14133).
```
-> Table scan on <temporary>  (cost=2.5..2.5 rows=0) (actual time=30376..30376 rows=391 loops=1)
    -> Temporary table with deduplication  (cost=0..0 rows=0) (actual time=30376..30376 rows=391 loops=1)
        -> Window aggregate with buffering: sum(payment.amount) OVER (PARTITION BY c.customer_id,f.title )   (actual time=13760..29069 rows=642000 loops=1)
            -> Sort: c.customer_id, f.title  (actual time=13760..14133 rows=642000 loops=1)
                -> Stream results  (cost=23.3e+6 rows=17.1e+6) (actual time=26.1..11257 rows=642000 loops=1)
                    -> Nested loop inner join  (cost=23.3e+6 rows=17.1e+6) (actual time=26.1..9910 rows=642000 loops=1)
                        -> Nested loop inner join  (cost=21.6e+6 rows=17.1e+6) (actual time=23.4..9300 rows=642000 loops=1)
                            -> Nested loop inner join  (cost=19.9e+6 rows=17.1e+6) (actual time=20.8..8288 rows=642000 loops=1)
                                -> Inner hash join (no condition)  (cost=1.65e+6 rows=16.5e+6) (actual time=17.7..434 rows=634000 loops=1)
                                    -> Filter: (cast(p.payment_date as date) = '2005-07-30')  (cost=1.94 rows=16500) (actual time=9..106 rows=634 loops=1)
                                        -> Table scan on p  (cost=1.94 rows=16500) (actual time=8.97..95.8 rows=16044 loops=1)
                                    -> Hash
                                        -> Covering index scan on f using idx_title  (cost=112 rows=1000) (actual time=6.79..8.42 rows=1000 loops=1)
                                -> Covering index lookup on r using rental_date (rental_date=p.payment_date)  (cost=1 rows=1.04) (actual time=0.00873..0.0119 rows=1.01 loops=634000)
                            -> Single-row index lookup on c using PRIMARY (customer_id=r.customer_id)  (cost=0.001 rows=1) (actual time=983e-6..0.00107 rows=1 loops=642000)
                        -> Single-row covering index lookup on i using PRIMARY (inventory_id=r.inventory_id)  (cost=0.001 rows=1) (actual time=566e-6..629e-6 rows=1 loops=642000)
```
Проведена оптимизация:  
* удалена сортировка,
* удалена таблица film,
* удалена таблица inventory,
* добавлена группировка.

Результат оптимизированного запроса.
```
select distinct concat(c.last_name, ' ', c.first_name), sum(p.amount)
from payment p, rental r, customer c
where date(p.payment_date) = '2005-07-30' and p.payment_date = r.rental_date and r.customer_id = c.customer_id
GROUP BY concat(c.last_name, ' ', c.first_name);
```
![](https://github.com/eskin-igor/netology_12-5/blob/main/12-5/12-05-2-3.JPG)

Реультат запроса с анализом.
```
EXPLAIN ANALYZE
select distinct concat(c.last_name, ' ', c.first_name), sum(p.amount)
from payment p, rental r, customer c
where date(p.payment_date) = '2005-07-30' and p.payment_date = r.rental_date and r.customer_id = c.customer_id
GROUP BY concat(c.last_name, ' ', c.first_name);
```
![](https://github.com/eskin-igor/netology_12-5/blob/main/12-5/12-05-2-4.JPG)
```
-> Table scan on <temporary>  (actual time=30..30 rows=391 loops=1)
    -> Aggregate using temporary table  (actual time=30..30 rows=391 loops=1)
        -> Nested loop inner join  (cost=25368 rows=17130) (actual time=0.797..26.5 rows=642 loops=1)
            -> Nested loop inner join  (cost=19373 rows=17130) (actual time=0.772..21 rows=642 loops=1)
                -> Filter: (cast(p.payment_date as date) = '2005-07-30')  (cost=1674 rows=16500) (actual time=0.71..13.5 rows=634 loops=1)
                    -> Table scan on p  (cost=1674 rows=16500) (actual time=0.68..9.98 rows=16044 loops=1)
                -> Covering index lookup on r using rental_date (rental_date=p.payment_date)  (cost=0.969 rows=1.04) (actual time=0.00793..0.0114 rows=1.01 loops=634)
            -> Single-row index lookup on c using PRIMARY (customer_id=r.customer_id)  (cost=0.25 rows=1) (actual time=0.00781..0.00807 rows=1 loops=642)
```

