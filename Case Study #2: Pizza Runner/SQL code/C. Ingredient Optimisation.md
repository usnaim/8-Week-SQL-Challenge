# C. Ingredient Optimisation

**1-What are the standard ingredients for each pizza?**
````
WITH cte_split_pizza_names AS (
SELECT
  pizza_id,
  REGEXP_SPLIT_TO_TABLE(toppings, '[,\s]+')::INTEGER AS topping_id
FROM pizza_runner.pizza_recipes
)
SELECT
  pizza_id,
  STRING_AGG(t2.topping_name::TEXT, ',') AS standard_ingredients
FROM cte_split_pizza_names AS t1
INNER JOIN pizza_runner.pizza_toppings AS t2
  ON t1.topping_id = t2.topping_id
GROUP BY pizza_id
ORDER BY pizza_id;
````
**OR**
````
WITH cte_extras AS (
SELECT
  REGEXP_SPLIT_TO_TABLE(extras, '[,\s]+')::INTEGER AS topping_id
FROM pizza_runner.customer_orders
WHERE  extras not IN ('null', '')
)
SELECT
  topping_name,
  COUNT(*) AS extras_count
FROM cte_extras
INNER JOIN pizza_runner.pizza_toppings
  ON cte_extras.topping_id = pizza_toppings.topping_id
GROUP BY topping_name
ORDER BY extras_count DESC;
````
**2-What was the most commonly added extra?**
````
WITH step_a As(
SELECT
  pizza_id,
  REGEXP_SPLIT_TO_TABLE(extras, '[,\s]+')::INTEGER AS topping_id
FROM new_customer_orders
WHERE extras != '')
SELECT topping_name, count(topping_name) as most_added
FROM step_a as t1
JOIN pizza_runner.pizza_toppings as t2
ON t1.topping_id=t2.topping_id
GROUP BY topping_name
ORDER BY 2 DESC;
````
**3-What was the most common exclusion?**
````
WITH cte_extras AS (
SELECT
  REGEXP_SPLIT_TO_TABLE(exclusions, '[,\s]+')::INTEGER AS topping_id
FROM new_customer_orders 
WHERE  exclusions!=''
)
SELECT
  topping_name,
  COUNT(*) AS exclusions_count
FROM cte_extras as t1
INNER JOIN pizza_runner.pizza_toppings as t2
  ON t1.topping_id = t2.topping_id
GROUP BY topping_name
ORDER BY exclusions_count DESC;
````
**OR**
````
WITH cte_exclusions AS (
SELECT
  REGEXP_SPLIT_TO_TABLE(exclusions, '[,\s]+')::INTEGER AS topping_id
FROM pizza_runner.customer_orders
WHERE exclusions IS NOT NULL AND exclusions NOT IN ('null', '')
)
SELECT
  topping_name,
  COUNT(*) AS exclusions_count
FROM cte_exclusions
INNER JOIN pizza_runner.pizza_toppings
  ON cte_exclusions.topping_id = pizza_toppings.topping_id
GROUP BY topping_name
ORDER BY exclusions_count DESC;
````
**4-Generate an order item for each record in the customers_orders table in the format of one of the following:**
* Meat Lovers
* Meat Lovers - Exclude Beef
* Meat Lovers - Extra Bacon
* Meat Lovers - Exclude Cheese, Bacon - Extra Mushroom, Peppers

**How we will do it**
>We will need to use both of our adjusted records in questions 2 and 3 to make this happen - the first step is to clean up 
the customer_orders table so we won’t need to deal with so many WHERE filters conditions as we’re solving this problem!
>
>Then we will need to deal with the special case where records are not generated for the REGEXP_SPLIT_TO_TABLE function 
using a UNION before we continue with some string manipulation to get us to the final output!
````
WITH cte_cleaned_customer_orders AS (
  SELECT
    order_id,
    customer_id,
    pizza_id,
    CASE
      WHEN exclusions IN ('', 'null') THEN NULL
      ELSE exclusions end
    AS exclusions,
    CASE
      WHEN extras IN ('', 'null') THEN NULL
      ELSE extras end
    AS extras,
    order_time,
    ROW_NUMBER() OVER () AS original_row_number
  FROM pizza_runner.customer_orders
),
-- when using the regexp_split_to_table function only records where there are
-- non-null records remain so we will need to union them back in!
cte_extras_exclusions AS (
    SELECT
      order_id,
      customer_id,
      pizza_id,
      REGEXP_SPLIT_TO_TABLE(exclusions, '[,\s]+')::INTEGER AS exclusions_topping_id,
     REGEXP_SPLIT_TO_TABLE(extras, '[,\s]+')::INTEGER AS extras_topping_id,
      order_time,
      original_row_number
    FROM cte_cleaned_customer_orders
  -- here we add back in the null extra/exclusion rows
  -- does it make any difference if we use UNION or UNION ALL?
  UNION
    SELECT
      order_id,
      customer_id,
      pizza_id,
      NULL AS exclusions_topping_id,
      NULL AS extras_topping_id,
      order_time,
      original_row_number
    FROM cte_cleaned_customer_orders
    WHERE exclusions IS NULL AND extras IS NULL
),
cte_complete_dataset AS (
  SELECT
    base.order_id,
    base.customer_id,
    base.pizza_id,
    names.pizza_name,
    base.order_time,
    base.original_row_number,
    STRING_AGG(exclusions.topping_name, ', ') AS exclusions,
    STRING_AGG(extras.topping_name, ', ') AS extras
  FROM cte_extras_exclusions AS base
  INNER JOIN pizza_runner.pizza_names AS names
    ON base.pizza_id = names.pizza_id
  LEFT JOIN pizza_runner.pizza_toppings AS exclusions
    ON base.exclusions_topping_id = exclusions.topping_id
  LEFT JOIN pizza_runner.pizza_toppings AS extras
    ON base.extras_topping_id = extras.topping_id
  GROUP BY
    base.order_id,
    base.customer_id,
    base.pizza_id,
    names.pizza_name,
    base.order_time,
    base.original_row_number
),
cte_parsed_string_outputs AS (
SELECT
  order_id,
  customer_id,
  pizza_id,
  order_time,
  original_row_number,
  pizza_name,
  CASE WHEN exclusions IS NULL THEN '' ELSE ' - Exclude ' || exclusions end AS exclusions,
  CASE WHEN extras IS NULL THEN '' ELSE ' - Extra ' || extras end AS extras
FROM cte_complete_dataset
),
final_output AS (
  SELECT
    order_id,
    customer_id,
    pizza_id,
    order_time,
    original_row_number,
    (pizza_name || exclusions || extras) AS order_item
  FROM cte_parsed_string_outputs
)
SELECT
  order_id,
  customer_id,
  pizza_id,
  order_time,
  order_item, 1
FROM final_output
ORDER BY original_row_number;
````
**5-Generate an alphabetically ordered comma separated ingredient list for each pizza order from the customer_orders table
and add a 2x in front of any relevant ingredients. For example: "Meat Lovers: 2xBacon, Beef, ... , Salami"**
````
WITH cte_cleaned_customer_orders AS (
  SELECT
    order_id,
    customer_id,
    pizza_id,
    CASE
      WHEN exclusions IN ('', 'null') THEN NULL
      ELSE exclusions end
    AS exclusions,
    CASE
      WHEN extras IN ('', 'null') THEN NULL
      ELSE extras end
    AS extras,
    order_time,
    ROW_NUMBER() OVER () AS original_row_number
  FROM pizza_runner.customer_orders
),
-- when using the regexp_split_to_table function only records where there are
-- non-null records remain so we will need to union them back in!
cte_extras_exclusions AS (
    SELECT
      order_id,
      customer_id,
     pizza_recipes.pizza_id,
       REGEXP_SPLIT_TO_TABLE(toppings, '[,\s]+')::INTEGER AS toppings_topping_id,
      REGEXP_SPLIT_TO_TABLE(exclusions, '[,\s]+')::INTEGER AS exclusions_topping_id,
     REGEXP_SPLIT_TO_TABLE(extras, '[,\s]+')::INTEGER AS extras_topping_id,
      order_time,
      original_row_number
    FROM cte_cleaned_customer_orders as cte
    join pizza_runner.pizza_recipes as pizza_recipes
    on pizza_recipes.pizza_id= cte.pizza_id
  -- here we add back in the null extra/exclusion rows
  -- does it make any difference if we use UNION or UNION ALL?
  UNION
    SELECT
      order_id,
      customer_id,
      pizza_id,
      null AS toppings_topping_id,
      NULL AS exclusions_topping_id,
      NULL AS extras_topping_id,
      order_time,
      original_row_number
    FROM cte_cleaned_customer_orders
    WHERE exclusions IS NULL AND extras IS NULL
),
cte_complete_dataset AS (
  SELECT
    base.order_id,
    base.customer_id,
    base.pizza_id,
    names.pizza_name,
    base.order_time,
    base.original_row_number,
    STRING_AGG(case 
    when extras is not null and toppings.topping_name=extras.topping_name then '2x' || toppings.topping_name
     when exclusions is not null and toppings.topping_name=exclusions.topping_name then '' 
    else toppings.topping_name end , ', ') AS toppings,
    STRING_AGG(exclusions.topping_name, ', ') AS exclusions,
    STRING_AGG(extras.topping_name, ', ') AS extras
  FROM cte_extras_exclusions AS base
     INNER JOIN pizza_runner.pizza_toppings AS toppings
    ON base.toppings_topping_id = toppings.topping_id
  INNER JOIN pizza_runner.pizza_names AS names
    ON base.pizza_id = names.pizza_id
  LEFT JOIN pizza_runner.pizza_toppings AS exclusions
    ON base.exclusions_topping_id = exclusions.topping_id
  LEFT JOIN pizza_runner.pizza_toppings AS extras
    ON base.extras_topping_id = extras.topping_id
  GROUP BY
    base.order_id,
    base.customer_id,
    base.pizza_id,
    names.pizza_name,
    base.order_time,
    base.original_row_number
),
cte_parsed_string_outputs AS (
SELECT
  order_id,
  customer_id,
  pizza_id,
  order_time,
  original_row_number,
  pizza_name,
  toppings,
  CASE WHEN exclusions IS NULL THEN '' ELSE '  ' || exclusions end AS exclusions,
  CASE WHEN extras IS NULL THEN '' ELSE '  ' || extras end AS extras
FROM cte_complete_dataset
),
final_output AS (
  SELECT
    order_id,
    customer_id,
    pizza_id,
    order_time,
    original_row_number,
            (pizza_name || ':' || toppings ) AS customer_ingredient
  FROM cte_parsed_string_outputs
)
SELECT
  order_id,
  customer_id,
  pizza_id,
  order_time,original_row_number,
  customer_ingredient
FROM final_output
ORDER BY original_row_number;
--Lösung von Danny
--5-Generate an alphabetically ordered comma separated ingredient list for each pizza order from the customer_orders table
--and add a 2x in front of any relevant ingredients. For example: "Meat Lovers: 2xBacon, Beef, ... , Salami"
WITH cte_cleaned_customer_orders AS (
  SELECT
    order_id,
    customer_id,
    pizza_id,
    CASE
      WHEN exclusions IN ('', 'null') THEN NULL
      ELSE exclusions
    END AS exclusions,
    CASE
      WHEN extras IN ('', 'null') THEN NULL
      ELSE extras
    END AS extras,
    order_time,
    ROW_NUMBER() OVER () AS original_row_number
  FROM pizza_runner.customer_orders
),
-- split the toppings using our previous solution
cte_regular_toppings AS (
SELECT
  pizza_id,
  REGEXP_SPLIT_TO_TABLE(toppings, '[,\s]+')::INTEGER AS topping_id
FROM pizza_runner.pizza_recipes
),
-- now we can should left join our regular toppings with all pizzas orders
cte_base_toppings AS (
  SELECT
    cte_cleaned_customer_orders.order_id,
    cte_cleaned_customer_orders.customer_id,
    cte_cleaned_customer_orders.pizza_id,
    cte_cleaned_customer_orders.order_time,
    cte_cleaned_customer_orders.original_row_number,
    cte_regular_toppings.topping_id
  FROM cte_cleaned_customer_orders
  LEFT JOIN cte_regular_toppings
    ON cte_cleaned_customer_orders.pizza_id = cte_regular_toppings.pizza_id
),
-- now we can generate CTEs for exclusions and extras by the original row number
cte_exclusions AS (
  SELECT
    order_id,
    customer_id,
    pizza_id,
    order_time,
    original_row_number,
    REGEXP_SPLIT_TO_TABLE(exclusions, '[,\s]+')::INTEGER AS topping_id
  FROM cte_cleaned_customer_orders
  WHERE exclusions IS NOT NULL
),
cte_extras AS (
  SELECT
    order_id,
    customer_id,
    pizza_id,
    order_time,
    original_row_number,
    REGEXP_SPLIT_TO_TABLE(extras, '[,\s]+')::INTEGER AS topping_id
  FROM cte_cleaned_customer_orders
  WHERE extras IS NOT NULL
),
-- now we can perform an except and a union all on the respective CTEs
cte_combined_orders AS (
  SELECT * FROM cte_base_toppings
  EXCEPT
  SELECT * FROM cte_exclusions
  UNION ALL
  SELECT * FROM cte_extras
),
-- aggregate the count of topping ID and join onto pizza toppings
cte_joined_toppings AS (
  SELECT
    t1.order_id,
    t1.customer_id,
    t1.pizza_id,
    t1.order_time,
    t1.original_row_number,
    t1.topping_id,
    t2.pizza_name,
    t3.topping_name,
    COUNT(t1.topping_id) AS topping_count
  FROM cte_combined_orders AS t1
  INNER JOIN pizza_runner.pizza_names AS t2
    ON t1.pizza_id = t2.pizza_id
  INNER JOIN pizza_runner.pizza_toppings AS t3
    ON t1.topping_id = t3.topping_id
  GROUP BY
    t1.order_id,
    t1.customer_id,
    t1.pizza_id,
    t1.order_time,
    t1.original_row_number,
    t1.topping_id,
    t2.pizza_name,
    t3.topping_name
)
SELECT
  order_id,
  customer_id,
  pizza_id,
  order_time,
  original_row_number,
  -- this logic is quite intense!
  pizza_name || ': ' || STRING_AGG(
    CASE
      WHEN topping_count > 1 THEN topping_count || 'x ' || topping_name
      ELSE topping_name
      END,
    ', '
  ) AS ingredients_list
FROM cte_joined_toppings
GROUP BY
  order_id,
  customer_id,
  pizza_id,
  order_time,
  original_row_number,
  pizza_name;
--6-What is the total quantity of each ingredient used in all delivered pizzas sorted by most frequent first?
--Data cleaning from table customer_orders
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
**6-What is the total quantity of each ingredient used in all delivered pizzas sorted by most frequent first?**
````
With cte_cleaned as(
select order_id,
customer_id,
pizza_id,
order_time,
 case when exclusions  in ('','null') then null else exclusions end,
 case when extras  in ('','null') then null else extras  end
from new_customer_orders
),
cte_toppings as (
select pizza_id,
REGEXP_SPLIT_TO_TABLE(toppings,'[,\s]+')::INTEGER AS topping_id
from pizza_runner.pizza_recipes
),
cte_toppings_customers as (
select cte_cleaned.order_id,
cte_cleaned.customer_id,
cte_cleaned.pizza_id,
cte_cleaned.order_time,
cte_toppings.topping_id
from cte_cleaned 
join cte_toppings
on cte_toppings.pizza_id=cte_cleaned.pizza_id
),
cte_exclusions as(
select order_id,
customer_id,
pizza_id,
order_time,
REGEXP_SPLIT_TO_TABLE(exclusions, '[,\s]+')::integer  as exclusions
from cte_cleaned
where exclusions is not null
),
cte_extras as(
select order_id,customer_id,pizza_id, order_time,
REGEXP_SPLIT_TO_TABLE(extras, '[,\s]+')::integer as extras
from cte_cleaned
where extras is not null

),
cte_complete as(
select * from cte_toppings_customers
except  
select * from cte_exclusions
union all
select * from cte_extras
)
select 
pizza_toppings.topping_name, count(*) as topping_count
from cte_complete
join pizza_runner.pizza_toppings
on pizza_toppings.topping_id=cte_complete.topping_id
group by topping_name
order by topping_count DESC;
````
