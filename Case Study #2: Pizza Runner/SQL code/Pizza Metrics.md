# A. Pizza Metrics

## Data cleaning from table customer_orders:
````
DROP TABLE IF EXISTS new_customer_orders;
CREATE TEMP TABLE new_customer_orders as(
SELECT order_id,
customer_id,
pizza_id,
 case 
 when exclusions is null or exclusions like 'null' then ''
else exclusions end as exclusions,
 case 
 when extras is null or extras like 'null' then '' else extras end as extras,
order_time
FROM pizza_runner.customer_orders
);
SELECT *
FROM new_customer_orders;
--Data cleaning from runner_orders
DROP TABLE IF EXISTS new_runner_orders;
CREATE TEMP TABLE new_runner_orders as(
SELECT order_id,
runner_id,
case 
  when pickup_time is null or pickup_time like 'null' then '' else pickup_time end as pickup_time,
case 
  when distance is null or distance like 'null' then ''  
    when distance  like '%km' then trim( 'km' from distance) else distance end as distance,
case 
  when duration is null or duration like 'null' then ''
  when duration like '%mins' then trim('mins' from duration) 
  when duration like '%minutes' then trim('minutes' from duration)
  when duration like '%minute' then trim('minute' from duration)
  else duration end as duration,
case 
  when cancellation is null or cancellation like 'null' then '' else cancellation end as cancellation
FROM pizza_runner.runner_orders
);
SELECT *
FROM new_runner_orders;
````
**Case Study questions**

**1-How many pizzas were ordered?**
````
SELECT count(order_id)
FROM pizza_runner.new_customer_orders;
````
**2-How many unique customer orders were made?**
````
SELECT  COUNT( DISTINCT order_id)
FROM pizza_runner.new_customer_orders;
````
**3-How many successful orders were delivered by each runner?**
````
SELECT  runner_id, count( order_id) as successful_orders
FROM new_runner_orders
WHERE distance  not like ''
GROUP BY 1
ORDER BY 2 DESC;
````
**4-How many of each type of pizza was delivered?**
````
SELECT  pizza_names.pizza_name,count(new_runner_orders.order_id)as count_delivered_pizza
FROM new_runner_orders
JOIN new_customer_orders
ON new_runner_orders.order_id= new_customer_orders.order_id
JOIN pizza_runner.pizza_names
ON new_customer_orders.pizza_id= pizza_names.pizza_id
WHERE distance  not like ''
GROUP BY 1
ORDER BY 1;
````
**5-How many Vegetarian and Meatlovers were ordered by each customer?**
````
SELECT
  customer_id,
  SUM(CASE WHEN pizza_id = 1 THEN 1 ELSE 0 END) AS meatlovers,
  SUM(CASE WHEN pizza_id = 2 THEN 1 ELSE 0 END) AS vegetarian
FROM new_customer_orders
GROUP BY 1
ORDER BY 1;
````
**6-What was the maximum number of pizzas delivered in a single order?**
````
WITH max_pizza as(
SELECT order_id,count(pizza_id) as pizza_count
FROM new_customer_orders
GROUP BY 1)
SELECT max(pizza_count)
FROM max_pizza;
````
**7-For each customer, how many delivered pizzas had at least 1 change and how many had no changes?**
````
SELECT new_customer_orders.customer_id, 
sum(case when new_customer_orders.exclusions != '' or new_customer_orders.extras !='' then 1 else 0 end) as has_changes,
sum(case when new_customer_orders.exclusions like '' and new_customer_orders.extras like'' then 1 else 0 end ) as no_changes
FROM new_customer_orders
JOIN new_runner_orders
ON new_runner_orders.order_id= new_customer_orders.order_id
WHERE new_runner_orders.cancellation not like 'Restaurant Cancellation' and new_runner_orders.cancellation not like 'Customer Cancellation' 
GROUP BY 1
ORDER BY 1;
````
**8-How many pizzas were delivered that had both exclusions and extras?**
````
SELECT  
sum(case when new_customer_orders.exclusions != '' and new_customer_orders.extras !='' then 1 else 0 end) as has_both_changes
FROM new_customer_orders
JOIN new_runner_orders
ON new_runner_orders.order_id= new_customer_orders.order_id
WHERE new_runner_orders.cancellation not like 'Restaurant Cancellation' and new_runner_orders.cancellation not like 'Customer Cancellation';
````
**9-What was the total volume of pizzas ordered for each hour of the day?**
````
SELECT
  EXTRACT('hour'FROM order_time) AS order_hour,
  count(pizza_id) AS total_pizza
FROM new_customer_orders
GROUP BY 1
ORDER BY 1;
````
**OR**
````
SELECT
  DATE_PART('hour', order_time) AS hour_of_day,
  COUNT(*) AS pizza_count
FROM pizza_runner.customer_orders
GROUP BY hour_of_day
ORDER BY hour_of_day;
````
**10-What was the volume of orders for each day of the week?**
````
SELECT
  TO_CHAR(order_time, 'Day') AS day_of_week,
  COUNT(order_id) AS pizza_count
FROM pizza_runner.customer_orders
GROUP BY day_of_week, DATE_PART('dow', order_time)
ORDER BY DATE_PART('dow', order_time);
````
