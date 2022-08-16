# D. Pricing and Ratings

**1-If a Meat Lovers pizza costs $12 and Vegetarian costs $10 and there were no charges for changes - how much money has Pizza Runner made so far 
if there are no delivery fees?**

**my solution**
 ````
 With pizza_count as(
SELECT t1.pizza_id, case when pizza_id=1 then 12*count(*) else '0'end as meatlovers_pizza_count,
case when pizza_id=2 then 10*count(*)else '0' end as vegetarian_pizza_count  
FROM new_customer_orders as t1
JOIN new_runner_orders as t2
ON t1.order_id=t2.order_id
WHERE pickup_time!=''
GROUP BY 1
)
SELECT sum( meatlovers_pizza_count + vegetarian_pizza_count) || '$' as money_made
FROM pizza_count;
````
**Dannys solution**
````
SELECT
  SUM(
    CASE
      WHEN pizza_id = 2 THEN 12
      WHEN pizza_id = 1 THEN 10
      END
  ) AS revenue
FROM pizza_runner.customer_orders
join pizza_runner.runner_orders
on customer_orders.order_id=runner_orders.order_id;
````
**2-What if there was an additional $1 charge for any pizza extras?
Add cheese is $1 extra**
````
With cte_extras  as(
select pizza_id,
REGEXP_SPLIT_TO_TABLE(extras, '[,\s]+')::integer as extras_id
from new_customer_orders
where extras not in ('','null')
),
extras_id as(
select pizza_id,count(*) as extras_count
from cte_extras
group by 1
),
 pizza_count as(
SELECT t1.pizza_id, case when t1.pizza_id=1 then 12*count(*) else '0'end as meatlovers_pizza_count,
                    case when t1.pizza_id=2  then 10*count(*) else '0' end as vegetarian_pizza_count  
FROM new_customer_orders as t1 
join new_runner_orders as t2
ON t1.order_id=t2.order_id
WHERE pickup_time!=''
GROUP BY t1.pizza_id
)
SELECT sum( meatlovers_pizza_count + vegetarian_pizza_count+extras_count) || '$' as money_made
FROM pizza_count as t1
join extras_id as t2
on t1.pizza_id=t2.pizza_id;
````
**3-The Pizza Runner team now wants to add an additional ratings system that allows customers to rate their runner, how would you design an additional 
--table for this new dataset - generate a schema for this new table and insert your own data for ratings for each successful customer order between 1 to 5.**
````
SELECT SETSEED(1);
DROP TABLE If EXISTS pizza_runner.rating;
CREATE TABLE  pizza_runner.rating
(
        "order_id" INTEGER,
  "rating" INTEGER
);

INSERT INTO pizza_runner.rating
SELECT
  order_id,
  FLOOR(1 + 5 * RANDOM()) AS rating
FROM pizza_runner.runner_orders
WHERE pickup_time IS NOT NULL;
select * from pizza_runner.rating;
````
**4-Using your newly generated table - can you join all of the information together to form a table which has 
the following information for successful deliveries?**
````
With cte_speed as (
SELECT order_id,
round(avg(distance::numeric/(duration::numeric/60)),1) as Average_speed
FROM new_runner_orders
WHERE distance!= '' and distance!= '' 
group by order_id
)
select 
customer_id,t1.order_id,runner_id,t2.pizza_id,rating,order_time,pickup_time::TIMESTAMP,
date_part('minutes', pickup_time::TIMESTAMP - order_time)as timebetweenorderandpickup, duration as Delivery_duration,
sum(pizza_id)over() as total_numberof_pizza, Average_speed
FROM  new_runner_orders as t1 
join new_customer_orders as t2 
on t1.order_id=t2.order_id join pizza_runner.rating as t3 
on t1.order_id=t3.order_id 
join cte_speed
on cte_speed.order_id=t1.order_id
where pickup_time!= '' and distance!= '' and distance!= '';
````
**5-If a Meat Lovers pizza was $12 and Vegetarian $10 fixed prices with no cost for extras and each runner is paid $0.30 per kilometre traveled 
 how much money does Pizza Runner have left over after these deliveries?**
````
WITH cte_runner_fees as(
SELECT sum(case when distance !='' then 0.30 else '0' end) as runner_fees,
      sum(case when pizza_id=1 then 12 when pizza_id=2 then 10 end) as pizza_revenue
from new_runner_orders as t1
join new_customer_orders as t2
on t1.order_id=t2.order_id
where pickup_time !='' and distance !=''
)
select (pizza_revenue - runner_fees) || '$' as money_left_over
from cte_runner_fees;
````
# E. Bonus Questions
**If Danny wants to expand his range of pizzas - how would this impact the existing data design? Write an INSERT statement to demonstrate what would happen 
if a new Supreme pizza with all the toppings was added to the Pizza Runner menu?**
````
DROP  TABLE IF EXISTS temp_pizza_names;
CREATE TEMP TABLE temp_pizza_names AS
SELECT * FROM pizza_runner.pizza_names;

INSERT INTO temp_pizza_names(pizza_id,pizza_name)
VALUES (3, 'Supreme');

DROP  TABLE IF EXISTS temp_pizza_recipes;
CREATE TEMP TABLE temp_pizza_recipes AS
SELECT * FROM pizza_runner.pizza_recipes;

INSERT INTO temp_pizza_recipes(pizza_id,toppings)
SELECT
  3,
  STRING_AGG(topping_id::text, ',')
FROM pizza_runner.pizza_toppings;
````

