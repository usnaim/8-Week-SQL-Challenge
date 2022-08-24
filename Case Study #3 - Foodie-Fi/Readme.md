````
SELECT customer_id, t1.plan_id, price, start_date
FROM foodie_fi.plans as t1
JOIN foodie_fi.subscriptions as t2
ON t1.plan_id= t2.plan_id
where customer_id  in (1,2,11,13,15,16,18,19);
````
**Case Study Questions**

| customer_id | plan_id   | price     | start_date  |
| ----------- | --------  | --------  | --------    |
|          1  |         0 |      0.00 | 2020-08-01  |
|          1  |         1 |      9.90 | 2020-08-08  |
|          2  |         0 |      0.00 | 2020-09-20  |
|          2  |         3 |    199.00 | 2020-09-27  |
|         11  |         0 |      0.00 | 2020-11-19  |
|         11  |         4 |      null | 2020-11-26  |
|          1  |         0 |      0.00 | 2020-08-01  |
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

**A. Customer Journey**
>Based off the 8 sample customers provided in the sample from the subscriptions table, write a brief description about each customer’s onboarding journey.

>Try to keep it as short as possible - you may also want to run some sort of join to make your explanations a bit easier!
