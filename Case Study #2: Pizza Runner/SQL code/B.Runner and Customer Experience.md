# B. Runner and Customer Experience
# Case Study questions

**1-How many runners signed up for each 1 week period? (i.e. week starts 2021-01-01)?**
````
SELECT (DATE_TRUNC('WEEK', registration_date) ::DATE) + 4 as registration_week , count(runner_id) as runners_count
FROM pizza_runner.runners
GROUP BY 1
ORDER BY 1;
````
**2-What was the average time in minutes it took for each runner to arrive at the Pizza Runner HQ to pickup the order?**
````
WITH cte_pickup_minutes AS (
  SELECT DISTINCT
    t1.order_id,t1.pickup_time,t2.order_time,
    DATE_PART('minutes', AGE(t1.pickup_time::TIMESTAMP ,t2.order_time))::INTEGER AS pickup_minutes
  FROM new_runner_orders AS t1
  INNER JOIN new_customer_orders AS t2
    ON t1.order_id = t2.order_id
  WHERE t1.pickup_time != ''
)
SELECT
  ROUND(AVG(pickup_minutes), 3) AS avg_pickup_minutes
FROM cte_pickup_minutes;
````
**3a-Is there any relationship between the number of pizzas and how long the order takes to prepare?**
````
WITH count_pizza AS (
  SELECT DISTINCT
 t2.order_id,count(pizza_id) as pizza_count
 FROM new_customer_orders AS t2
 GROUP BY order_id) 
SELECT DISTINCT
    t1.order_id, pizza_count,
    DATE_PART('minutes', AGE(t1.pickup_time::TIMESTAMP ,t2.order_time))::INTEGER AS pickup_minutes
    FROM count_pizza as t3
     JOIN new_runner_orders AS t1
     ON t3.order_id = t1.order_id
  INNER JOIN new_customer_orders AS t2
    ON t1.order_id = t2.order_id
  WHERE t1.pickup_time != ''
  ORDER BY pickup_minutes;
  ````
  **3b-Is there any relationship between the number of pizzas and how long the order takes to prepare?**
  ````
SELECT DISTINCT
   count(pizza_id) as pizza_count,
    DATE_PART('minutes', AGE(t1.pickup_time::TIMESTAMP ,t2.order_time))::INTEGER AS pickup_minutes
    FROM  new_runner_orders AS t1
       INNER JOIN new_customer_orders AS t2
    ON t1.order_id = t2.order_id
  WHERE t1.pickup_time != ''
   GROUP BY t2.order_id,pickup_minutes
  ORDER BY pickup_minutes;
````
**4-What was the average distance travelled for each customer?**
````
  SELECT customer_id,ROUND(avg(distance::numeric),2)
  FROM new_runner_orders AS t1
  INNER JOIN new_customer_orders AS t2
    ON t1.order_id = t2.order_id
  WHERE distance!=''
  GROUP BY customer_id
  ORDER BY customer_id;
````
**5-What was the difference between the longest and shortest delivery times for all orders?**
````
WITH delivery_time As(
 SELECT 
    duration::NUMERIC AS delivery_minutes
  FROM new_runner_orders AS t1
  INNER JOIN new_customer_orders AS t2
    ON t1.order_id = t2.order_id
    WHERE t1.pickup_time != ''
 )
 SELECT ( max(delivery_minutes) - min(delivery_minutes)) as time_difference
 FROM delivery_time;
````
**6-What was the average speed for each runner for each delivery and do you notice any trend for these values?**
 ````
 WITH info AS(
  SELECT 
 (distance::numeric) as distance,(duration::numeric) as duration,
  runner_id, order_id
 FROM new_runner_orders 
 WHERE pickup_time != '')
 SELECT runner_id, order_id, distance, duration, ROUND(distance / (duration / 60),1) AS avg_speed
 FROM info
 GROUP BY runner_id,order_id,distance,duration
 ORDER BY runner_id;
 ````
**7-What is the successful delivery percentage for each runner?**
````
SELECT runner_id, 
 ROUND(
 100 * SUM(CASE WHEN pickup_time!='' THEN 1 ELSE 0 END)/
 count(*)) AS success_delivery_perc
 FROM new_runner_orders
 GROUP BY runner_id
 ORDER BY runner_id;
 ````
