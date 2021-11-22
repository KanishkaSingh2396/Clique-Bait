# Clique-Bait

Digital Analysis

Question 1 How many users are there?

```sql
select count(distinct user_id) as user_count from clique_bait.users
```
![image](https://user-images.githubusercontent.com/89623051/142752660-fa9ee2b7-9f76-42a3-9ffe-361147c56597.png)


Question 2 How many cookies does each user have on average?
```sql

select round(avg(count(cookie_id)) over(),2) as average_cookies
from clique_bait.users
group by user_id
limit 1
```
![image](https://user-images.githubusercontent.com/89623051/142752940-c2acda7f-6863-4285-baa0-5b301aebb078.png)


Question 3 What is the unique number of visits by all users per month?
```sql
select  date_trunc('month', event_time) as month, count(distinct visit_id) as unique_month_visiters
from clique_bait.events
group by month
order by month
``
![image](https://user-images.githubusercontent.com/89623051/142753335-85dca6fa-82bd-4052-b2ce-a512e58eb008.png)

Question 4 What is the number of events for each event type?

```sql
select 
  * 
from clique_bait.events e
join clique_bait.event_identifier ei
on e.event_type = ei.event_type
group by ei.event_type, event_name
order by ei.event_type


```

![image](https://user-images.githubusercontent.com/89623051/142754779-9a6a85f6-8b9e-4320-b98e-d305fda8e795.png)

Question 5 What is the percentage of visits which have a purchase event?

```sql
with base as (
select
  visit_id,
 max(case when event_type = 3  then 1 else 0 end) as total_visiters_purchased
from  clique_bait.events
group by visit_id)
select 
  sum(total_visiters_purchased) as total_visiters_purchased, 
  count(*)  as total_visiters,
  round(100*(sum(total_visiters_purchased)::numeric / count(*)::numeric),2) as percentage_visiters_purchased
from base
```

![image](https://user-images.githubusercontent.com/89623051/142773879-acd15733-9396-436d-98af-c5b058ea863b.png)

Question 6 What is the percentage of visits which view the checkout page but do not have a purchase event?

```sql
with base as(
select
  visit_id,
  max(case when page_id = 12 and event_type != 3 and event_type = 1 then 1 else 0 end) as checkout_flag,
  max(case when event_type = 3 then 1 else 0 end) as purchase_flag
from  clique_bait.events
group by visit_id)

select 
  round(100*(sum(case when purchase_flag = 0 then 1 else 0 end)::numeric / count(*)::numeric),2) as percentage_checkout_wtout_purchase
from base
where checkout_flag = 1
```
![image](https://user-images.githubusercontent.com/89623051/142774958-d1f3d72d-5ba2-4dd9-974a-d6a460b66078.png)

Question 7 What are the top 3 pages by number of views?

```sql
select e.page_id, page_name, sum(case when event_type = 1 then 1 else 0 end) as total_views
from clique_bait.events e
join clique_bait.page_hierarchy p
on e.page_id = p.page_id
group by e.page_id, page_name
order by total_views desc
limit 3
```
![image](https://user-images.githubusercontent.com/89623051/142796983-64c70574-a0f5-4843-95ff-54c522dff14c.png)


Question 8 What is the number of views and cart adds for each product category?
event_type = 2, add_cart , event_type = 1, page_view, group by product category

```sql
select 
  product_category,
  sum(case when event_type = 1 then 1 else 0 end)as total_views,
  sum(case when event_type = 2 then 1 else 0 end) as total_add_carts
from clique_bait.events e
join clique_bait.page_hierarchy h
on e.page_id = h.page_id
group by product_category

```
![image](https://user-images.githubusercontent.com/89623051/142797038-8d0ba38c-ee70-47ac-89a1-938d9e3cfa66.png)

Question_9 What are the top 3 products by purchases?

```sql
with cart_event AS (
  SELECT
    page_hierarchy.product_id, visit_id,
    SUM(CASE WHEN event_type = 2 THEN 1 ELSE 0 END) AS cart_add
  FROM clique_bait.events
  INNER JOIN clique_bait.page_hierarchy
    ON events.page_id = page_hierarchy.page_id
  WHERE page_hierarchy.product_id IS NOT NULL
  GROUP BY
    events.visit_id,
    page_hierarchy.product_id
    
),
visit_purchase AS (
  SELECT DISTINCT
    visit_id
  FROM clique_bait.events
  WHERE event_type = 3  -- purchase event
),
combined_product_events AS (
  SELECT
    t1.product_id,
    t1.cart_add,
    CASE WHEN t2.visit_id IS NULL THEN 1 ELSE 0 END as purchase
  FROM cart_event AS t1
  LEFT JOIN visit_purchase AS t2
    ON t1.visit_id = t2.visit_id
)
SELECT
  product_id,
  SUM(CASE WHEN cart_add = 1 AND purchase = 0 THEN 1 ELSE 0 END) AS added_and_not_purchased,
  SUM(CASE WHEN cart_add = 1 AND purchase = 1 THEN 1 ELSE 0 END) AS purchases
FROM combined_product_events
GROUP BY product_id;
```

![image](https://user-images.githubusercontent.com/89623051/142919204-cb9212e9-5d04-4222-81a3-e7b9e2d18eac.png)


Part C. Product Funnel Analysis
Table Creation Step
Using a single SQL query - create a new output table which has the following details:

How many times was each product viewed?
How many times was each product added to cart?
How many times was each product added to a cart but not purchased (abandoned)?
How many times was each product purchased?

```sql
DROP TABLE IF EXISTS product_info;
CREATE TEMP TABLE product_info AS
WITH cte_product_page_events AS (
  SELECT
    events.visit_id,
    page_hierarchy.product_id,
    page_hierarchy.page_name,
    page_hierarchy.product_category,
    SUM(CASE WHEN event_type = 1 THEN 1 ELSE 0 END) AS page_view,
    SUM(CASE WHEN event_type = 2 THEN 1 ELSE 0 END) AS cart_add
  FROM clique_bait.events
  INNER JOIN clique_bait.page_hierarchy
    ON events.page_id = page_hierarchy.page_id
  WHERE page_hierarchy.product_id IS NOT NULL
  GROUP BY
    events.visit_id,
    page_hierarchy.product_id,
    page_hierarchy.page_name,
    page_hierarchy.product_category
),
cte_visit_purchase AS (
  SELECT DISTINCT
    visit_id
  FROM clique_bait.events
  WHERE event_type = 3  -- purchase event
),
cte_combined_product_events AS (
  SELECT
    t1.visit_id,
    t1.product_id,
    t1.page_name,
    t1.product_category,
    t1.page_view,
    t1.cart_add,
    CASE WHEN t2.visit_id IS NULL THEN 1 ELSE 0 END as purchase
  FROM cte_product_page_events AS t1
  LEFT JOIN cte_visit_purchase AS t2
    ON t1.visit_id = t2.visit_id
)
SELECT
  product_id,
  page_name AS product,
  product_category,
  SUM(page_view) AS page_views,
  SUM(cart_add) AS cart_adds,
  SUM(CASE WHEN cart_add = 1 AND purchase = 0 THEN 1 ELSE 0 END) AS abandoned,
  SUM(CASE WHEN cart_add = 1 AND purchase = 1 THEN 1 ELSE 0 END) AS purchases
FROM cte_combined_product_events
GROUP BY product_id, product_category, product;
```

![image](https://user-images.githubusercontent.com/89623051/142919016-6d372cee-a0f1-4138-b574-9d6b440dc052.png)


Additionally, create another table which further aggregates the data for the above points but this time for each product category instead of individual products.

```sql
DROP TABLE IF EXISTS product_category_info;
CREATE TEMP TABLE product_category_info AS
SELECT
  product_category,
  SUM(page_views) AS page_views,
  SUM(cart_adds) AS cart_adds,
  SUM(abandoned) AS abandoned,
  SUM(purchases) AS purchases
FROM product_info
GROUP BY product_category;

SELECT * FROM product_category_info ORDER BY product_category;

```
![image](https://user-images.githubusercontent.com/89623051/142918674-77bbc0aa-bdd9-4daf-a4fc-abdb92a853d5.png)


Question 1 Which product had the most views, cart adds and purchases?
```sql
select product, page_views, cart_adds, purchases
from product_info 
order by page_views desc, cart_adds desc, purchases desc
```
![image](https://user-images.githubusercontent.com/89623051/142918477-be0fb646-3455-4aa8-86be-22ed50b3461e.png)


Question 2 Which product was most likely to be abandoned?

```sql
SELECT
  product,
   (round((purchases / cart_adds),2)) AS abandoned_likelihood
FROM product_info
ORDER BY abandoned_likelihood DESC;
```
![image](https://user-images.githubusercontent.com/89623051/142918351-fc3d3135-2507-47f1-a6aa-d08d122689c9.png)


Question 3 Which product had the highest view to purchase percentage?

```sql
SELECT
  product,
  ROUND(100 * (abandoned / page_views),2) AS view_to_purchase_percentage
FROM product_info
ORDER BY view_to_purchase_percentage desc
limit 1
```

![image](https://user-images.githubusercontent.com/89623051/142918213-a9ede5fb-70b5-4871-9652-4fe5df514d50.png)


Question 4 What is the average conversion rate from view to cart add?

```sql
SELECT
  ROUND(avg(100 *(cart_adds/page_views)) ,2) AS avg_view_to_cart
FROM product_info
ORDER BY avg_view_to_cart desc;
```
![image](https://user-images.githubusercontent.com/89623051/142918086-7f079c26-1762-40d2-9d40-7c27f5333775.png)


Question 5 What is the average conversion rate from cart add to purchase?

```sql
SELECT
  ROUND(avg(100 *(abandoned/cart_adds)) ,2) AS avg_cart_to_purchase
FROM product_info
ORDER BY avg_cart_to_purchase desc;
```
![image](https://user-images.githubusercontent.com/89623051/142917730-749d81b1-615a-4845-80f7-98924f2271f6.png)


Part D. Campaigns Analysis

Generate a table that has 1 single row for every unique visit_id record and has the following columns:

user_id

visit_id

visit_start_time: the earliest event_time for each visit

page_views: count of page views for each visit

cart_adds: count of product cart add events for each visit

purchase: 1/0 flag if a purchase event exists for each visit

campaign_name: map the visit to a campaign if the visit_start_time falls between the start_date and end_date

impression: count of ad impressions for each visit

click: count of ad clicks for each visit

(Optional column) cart_products: a comma separated text value with products added to the cart sorted by the order they were added
to the cart (hint: use the sequence_number)


```sql
SELECT
  users.user_id,
  events.visit_id,
  MIN(events.event_time) AS visit_start_time,
  SUM(CASE WHEN events.event_type = 1 THEN 1 ELSE 0 END) AS page_views,
  SUM(CASE WHEN events.event_type = 2 THEN 1 ELSE 0 END) AS cart_adds,
  MAX(CASE WHEN events.event_type = 3 THEN 1 ELSE 0 END) AS purchase,
  campaign_identifier.campaign_name,
  MAX(CASE WHEN events.event_type = 4 THEN 1 ELSE 0 END) AS impression,
  MAX(CASE WHEN events.event_type = 5 THEN 1 ELSE 0 END) AS click,
  STRING_AGG(
    CASE
      WHEN page_hierarchy.product_id IS NOT NULL AND event_type = 2
        THEN page_hierarchy.page_name
      ELSE NULL END,
    ', ' ORDER BY events.sequence_number
  ) AS cart_products
FROM clique_bait.events
INNER JOIN clique_bait.users
  ON events.cookie_id = users.cookie_id
LEFT JOIN clique_bait.campaign_identifier
  ON events.event_time BETWEEN campaign_identifier.start_date AND campaign_identifier.end_date
LEFT JOIN clique_bait.page_hierarchy
  ON events.page_id = page_hierarchy.page_id
GROUP BY
  users.user_id,
  events.visit_id,
  campaign_identifier.campaign_name;

```

![image](https://user-images.githubusercontent.com/89623051/142917230-142b600b-2cfa-4c94-9e55-db75ddb5542a.png)






