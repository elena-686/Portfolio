


**Задание 1.** 
 *Выведите название самолетов, которые имеют менее 50 посадочных мест.*

```sql
select model as "Название самолета"
from seats s 
join aircrafts a on s.aircraft_code = a.aircraft_code
group by s.aircraft_code, model 
having count(s.aircraft_code) < 50
```

/*Задание 2 Выведите процентное изменение ежемесячной суммы бронирования билетов, округленной до сотых. */

--ДОРАБОТКА
select date_trunc ('month', book_date::date) as "Месяц", 
       sum (total_amount) as "Сумма бронирования" ,  
       coalesce (round (sum (total_amount) / (lag (sum (total_amount)) 
                 over ( order by date_trunc ('month', book_date::date) ))*100-100,2), 0.00) as "Изменение,%"
from bookings b 
group by date_trunc ('month', book_date::date)
 



/*Задание 3  Выведите названия самолетов не имеющих бизнес - класс. Решение должно быть через функцию array_agg.*/

select model
from (select aircraft_code
      from seats s 
      group by aircraft_code
      having 'Business' != all(array_agg(fare_conditions)))c
join aircrafts a using (aircraft_code)


/*Задание 4 Вывести накопительный итог количества мест в самолетах по каждому аэропорту на каждый день, 
 * учитывая только те самолеты, которые летали пустыми и только те дни, где из одного аэропорта таких самолетов 
 * вылетало более одного.
 В результате должны быть код аэропорта, дата, количество пустых мест в самолете и накопительный итог.*/

--ДОРАБОТКА 
with cte as( 
            select airport_code, date_trunc('day',scheduled_departure), count(bp.ticket_no), a2.aircraft_code
            from flights f 
            left join airports a on a.airport_code = f.departure_airport 
            left join boarding_passes bp using (flight_id)
            left join aircrafts a2 on a2.aircraft_code = f.aircraft_code
            group by  scheduled_departure, airport_code, a2.aircraft_code
            having count(bp.ticket_no) = 0
            order by date_trunc('day',scheduled_departure)      
           ),
      cte1 as (
            select airport_code,cte.aircraft_code, date_trunc(cte)
            from cte
            group by airport_code ,date_trunc(cte),count(cte),cte.aircraft_code
            having count(date_trunc(cte)) >= 2
            order by date_trunc(cte)
            )
select cte1.airport_code,  date_trunc(cte1), count(seat_no), 
       sum (count(seat_no)) over (partition by cte1.airport_code order by date_trunc(cte1))
from cte1
join seats s on s.aircraft_code = cte1.aircraft_code
group by cte1.airport_code,  date_trunc(cte1),cte1.aircraft_code
order by 1



/*Задание 5 Найдите процентное соотношение перелетов по маршрутам от общего количества перелетов.
 Выведите в результат названия аэропортов и процентное отношение.
 Решение должно быть через оконную функцию.*/

--ДОРАБОТКА
select concat(a.airport_name , '-', a2.airport_name) as "Маршрут", 
       (count(f.flight_id)/sum(count(flight_id)) over())*100 as "Соотношение перелетов по маршруту от общего количества перелетов" 
from flights f 
join airports a on a.airport_code = f.departure_airport 
join airports a2 on a2.airport_code = f.arrival_airport 
group by concat(a.airport_name , '-', a2.airport_name)




/*Задание 6 Выведите количество пассажиров по каждому коду сотового оператора, если учесть,
 *  что код оператора - это три символа после +7*/

select substring((contact_data->>'phone'),3,3) as "код оператора", count(1) as "количество пассажиров"--::numeric --, character_length(contact_data->>'phone') --, pg_typeof ((contact_data->>'phone')::numeric) 
from tickets t 
group  by 1


/*Задание 7  Классифицируйте финансовые обороты (сумма стоимости перелетов) по маршрутам:
 До 50 млн - low
 От 50 млн включительно до 150 млн - middle
 От 150 млн включительно - high
 Выведите в результат количество маршрутов в каждом полученном классе.*/


select "case", count (t.flight_no)
     from (select sum(tf.amount), f.flight_no ,
               case
   	           when sum(amount)<50000000 then 'low'
     	           when sum(amount)>=50000000 and sum(amount) <150000000 then 'middle'
    	           else 'high'
               end  
            from flights f
            join ticket_flights tf using(flight_id)
            group by f.flight_no) t
group by  "case"
order by 2

/*Задание 8 Вычислите медиану стоимости перелетов, медиану размера бронирования и отношение медианы бронирования 
 * к медиане стоимости перелетов, округленной до сотых*/

select percentile_cont(0.5) within group (order by amount) as "Медиана стоимости перелетов",
      b."Медиана размера бронирования" , 
      round(b."Медиана размера бронирования"::numeric/percentile_cont(0.5) within group (order by amount)::numeric,2)
      as "Отношение"
from ticket_flights tf,( 
              select percentile_cont(0.5) within group (order by total_amount) as "Медиана размера бронирования"
              from bookings) b 
group by 2


/*Задание 9 Найдите значение минимальной стоимости полета 1 км для пассажиров. 
 * То есть нужно найти расстояние между аэропортами и с учетом стоимости перелетов получить искомый результат*/

--ДОРАБОТКА
select  f.departure_airport , 
        f.arrival_airport,sum(amount),
        earth_distance(ll_to_earth(a.longitude,a.latitude), ll_to_earth(a2.longitude,a2.latitude))/1000 AS distance,
        min((sum(amount)/(earth_distance(ll_to_earth(a.longitude,a.latitude), ll_to_earth(a2.longitude,a2.latitude))/1000))) over() 
from flights f 
join airports a on f.departure_airport = a.airport_code 
join airports a2 on f.arrival_airport = a2.airport_code 
join ticket_flights tf on tf.flight_id = f.flight_id 
group by f.flight_id, f.departure_airport,  a.longitude, a.latitude, f.arrival_airport, 
a2.longitude, a2.latitude,amount

