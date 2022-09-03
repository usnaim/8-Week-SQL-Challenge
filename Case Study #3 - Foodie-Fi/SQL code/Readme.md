## Case Study Questions

### A. Customer Journey
>Based off the 8 sample customers provided in the sample from the subscriptions table, write a brief description about each customer’s onboarding journey.

>Try to keep it as short as possible - you may also want to run some sort of join to make your explanations a bit easier!
````
SELECT customer_id, t1.plan_id, price, start_date
FROM foodie_fi.plans as t1
JOIN foodie_fi.subscriptions as t2
ON t1.plan_id= t2.plan_id
where customer_id  in (1,2,11,13,15,16,18,19);
````


| customer_id | plan_id   | price     | start_date  |
| ----------- | --------  | --------  | --------    |
|          1  |         0 |      0.00 | 2020-08-01  |
|          1  |         1 |      9.90 | 2020-08-08  |
|          2  |         0 |      0.00 | 2020-09-20  |
|          2  |         3 |    199.00 | 2020-09-27  |
|         11  |         0 |      0.00 | 2020-11-19  |
|         11  |         4 |      null | 2020-11-26  |
|         13  |         0 |      0.00 | 2020-12-15  |
|         13  |         1 |      9.90 | 2020-12-22  |
|         13  |         2 |     19.90 | 2021-03-29  |
|         15  |         0 |      0.00 | 2020-03-17  |
|         15  |         2 |     19.90 | 2020-03-24  |
|         15  |         4 |      null | 2020-04-29  |
|         16  |         0 |      0.00 | 2020-05-31  |
|         16  |         1 |      9.90 | 2020-06-07  |
|         16  |         3 |    199.00 | 2020-10-21  |
|         18  |         0 |      0.00 | 2020-07-06  |
|         18  |         2 |     19.90 | 2020-07-13  |
|         19  |         0 |      0.00 | 2020-06-22  |
|         19  |         2 |     19.90 | 2020-06-29  |
|         19  |         3 |    199.00 | 2020-08-29  | 

>From the table above, we can see how the Customers upgraded their susscriptions after the free trial differently.  Customesr with id 1,13 and 16 choose the Basic plan,  Customer with id 2 upgraded to the annual subscription. While customer with id 4 cancel the service after the free trial, customer with id 15 upgraded his subscription first to a pro monthly plan and then cancelled his susbscription. Meanwhile Customers upgraded their susbscription further to the annual plan, C. with id 16 passing by tha basic plan and C.  with id 19 passing by the monthly plan, customers with id 18 and 13 upgraded to the pro monthly plan.

### B. Data Analysis Questions

**1-How many customers has Foodie-Fi ever had?**
````
SELECT count (DISTINCT customer_id)
FROM foodie_fi.subscriptions;
````
**2-What is the monthly distribution of trial plan start_date values for our dataset - use the start of the month as the group by value?**
````

SELECT
  DATE_TRUNC('month', start_date)::DATE AS start_of_month,
  COUNT(*) AS trial_customers
FROM foodie_fi.subscriptions
WHERE plan_id = 0
GROUP BY start_of_month
ORDER BY start_of_month;
````
**3-What plan start_date values occur after the year 2020 for our dataset? Show the breakdown by count of events for each plan_name?**
````
SELECT
   plan_name,
 t1.plan_id, count(*)
FROM foodie_fi.plans as t1
JOIN foodie_fi.subscriptions as t2
ON t1.plan_id=t2.plan_id
WHERE start_date >'2020-12-31'
GROUP BY t1.plan_id,plan_name;
````
**4-What is the customer count and percentage of customers who have churned rounded to 1 decimal place?**
````
SELECT sum( case when plan_id=4 then 1 end) as churned_customer_count,
       Round(
     100*  sum( case when plan_id=4 then 1 end) /
       count( distinct customer_id)::Numeric,1 
           ) as percentage 
FROM foodie_fi.subscriptions;
````
**5-How many customers have churned straight after their initial free trial - what percentage is this rounded to 1 decimal place?**
````
WITH ranked_plans AS (
 SELECT
    customer_id,
    plan_id,
    ROW_NUMBER() OVER (
      PARTITION BY customer_id
      ORDER BY start_date 
    ) AS plan_rank
  FROM foodie_fi.subscriptions
)
SELECT
  SUM(CASE WHEN plan_id = 4 THEN 1 ELSE 0 END) AS churn_customers,
  ROUND(
    100 * SUM(CASE WHEN plan_id = 4 THEN 1 ELSE 0 END) /
    COUNT(*)
  ) AS percentage
FROM ranked_plans
WHERE plan_rank = 2;
````
**6-What is the number and percentage of customer plans after their initial free trial?**
````
WITH ranked_plans AS (
 SELECT
    customer_id,
    plan_id,
    ROW_NUMBER() OVER (
      PARTITION BY customer_id
      ORDER BY start_date 
    ) AS plan_rank
  FROM foodie_fi.subscriptions
)

SELECT
  SUM(CASE WHEN plan_id = 4 THEN 1 ELSE 0 END) AS churn_customers,
  ROUND(
    100 * SUM(CASE WHEN plan_id = 4 THEN 1 ELSE 0 END)/SUM(count(*)) over())
    AS churn_customers_percentage,
    
  SUM(CASE WHEN plan_id = 1 THEN 1 ELSE 0 END) AS basic_plan,
  ROUND(
    100 * SUM(CASE WHEN plan_id = 1 THEN 1 ELSE 0 END) /SUM(count(*)) over())
   AS basic_plan_percentage,
  
  SUM(CASE WHEN plan_id = 4 THEN 2 ELSE 0 END) AS monthly_plan,
  ROUND(
    100 * SUM(CASE WHEN plan_id = 2 THEN 1 ELSE 0 END) /
    SUM(count(*)) over()) AS monthly_plan_percentage,
    
  SUM(CASE WHEN plan_id = 3 THEN 1 ELSE 0 END) AS pro_annual,
  ROUND(
    100 * SUM(CASE WHEN plan_id = 3 THEN 1 ELSE 0 END) /
   SUM(count(*)) over())
   AS pro_plan_percentage
FROM ranked_plans
WHERE plan_rank = 2;
````
**7-What is the customer count and percentage breakdown of all 5 plan_name values at 2020-12-31? **
````
With prior_date as( 
SELECT plan_id,count(distinct customer_id) as customer_count
FROM foodie_fi.subscriptions 
WHERE start_date<='2020-12-31'
GROUP BY plan_id
),
actual_customers as(
SELECT plan_id, customer_count
FROM prior_date
)
SELECT plans.plan_id,plans.plan_name, customer_count, ROUND(100 * customer_count/ sum(customer_count) over(),1) as percentage
FROM  actual_customers
JOIN foodie_fi.plans
ON  actual_customers.plan_id = foodie_fi.plans.plan_id
ORDER BY 1;
````
**8-How many customers have upgraded to an annual plan in 2020?**
````
With year_part as(
SELECT customer_id, plan_id,date_part('year',start_date) as year_2020
FROM foodie_fi.subscriptions
WHERE plan_id=3
)
SELECT count(distinct customer_id)
FROM year_part 
WHERE year_2020= 2020;
````
**--9-How many days on average does it take for a customer to an annual plan from the day they join Foodie-Fi?**
````
With basic_plan as(
SELECT customer_id, start_date as trial_date
FROM foodie_fi.subscriptions
WHERE plan_id = 0
),
pro_annual as (
SELECT customer_id, start_date as annual_date
FROM foodie_fi.subscriptions
WHERE plan_id = 3
)

SELECT ROUND(avg(date_part('day',(annual_date::timestamp - trial_date::timestamp)))) as avg_days
FROM basic_plan
JOIN pro_annual
ON   basic_plan.customer_id=pro_annual.customer_id;
````
**--10-Can you further breakdown this average value into 30 day periods (i.e. 0-30 days, 31-60 days etc)?**
````
With basic_plan as(
SELECT customer_id, start_date as trial_date
FROM foodie_fi.subscriptions
WHERE plan_id = 0
),
pro_annual as (
SELECT customer_id, start_date as annual_date
FROM foodie_fi.subscriptions
WHERE plan_id = 3
)

SELECT           ( case when (date_part('day',(annual_date::timestamp - trial_date::timestamp)))<= 30 then  '0-30 days'
                        when (date_part('day',(annual_date::timestamp - trial_date::timestamp))) <= 60 then '31-60 days'
                        when (date_part('day',(annual_date::timestamp - trial_date::timestamp))) <= 90 then  '61-90 days'
                        when (date_part('day',(annual_date::timestamp - trial_date::timestamp))) <= 120 then '91-120 days'
                        when (date_part('day',(annual_date::timestamp - trial_date::timestamp))) <= 150 then '121-150 days'
                        when (date_part('day',(annual_date::timestamp - trial_date::timestamp))) <= 180 then '151-180 days'
                        when (date_part('day',(annual_date::timestamp - trial_date::timestamp))) <= 210 then '181-210 days'
                        when (date_part('day',(annual_date::timestamp - trial_date::timestamp))) <= 240 then '211-240 days'
                        when (date_part('day',(annual_date::timestamp - trial_date::timestamp))) <= 270 then '241-270 days'
                        when (date_part('day',(annual_date::timestamp - trial_date::timestamp))) <= 300 then '271-300 days'
                        when (date_part('day',(annual_date::timestamp - trial_date::timestamp))) <= 330 then '301-330 days'
                        when (date_part('day',(annual_date::timestamp - trial_date::timestamp))) <= 360 then '331-360 days'
                        when (date_part('day',(annual_date::timestamp - trial_date::timestamp))) <= 390 then '361-390 days' end)as avg_days,
 count(*) as customers
FROM basic_plan
JOIN pro_annual
ON   basic_plan.customer_id=pro_annual.customer_id 
GROUP BY avg_days
ORDER BY avg_days DESC;
````
**--11-How many customers downgraded from a pro monthly to a basic monthly plan in 2020?**
````
WITH abc as(
SELECT customer_id
FROM foodie_fi.subscriptions
WHERE start_date< '2020-12-31' and plan_id=2
GROUP BY 1
),
cba as (
SELECT customer_id
FROM foodie_fi.subscriptions
WHERE start_date between '2020-01-01' and '2020-12-31' and plan_id=1
)
SELECT count(abc.customer_id)
FROM abc 
JOIN cba
ON abc.customer_id=cba.customer_id;
````
### C. Challenge Payment Question
