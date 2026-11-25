# Danny's diner

## 1. What is the total amount each customer spent at the restaurant?
tables: sales, menu
### SQL query
```sql
SELECT 
	s.customer_id, 
	SUM(m.price) total_amount
FROM 
	sales s
INNER JOIN 
	menu m 
		USING(product_id)
GROUP BY 
	customer_id
ORDER BY 
	customer_id
;
```
### Result
| customer_id | total_amount |
|-------------|--------------|
| A	          |76            |
| B		      |74            |
| C		      |36            |
### Answer
- Customer A spent 76
- Customer B spent 74
- Customer C spent 36
## 2. How many days has each customer visited the restaurant?
tables: sales
### SQL query
```sql
SELECT 
	customer_id, 
	COUNT(DISTINCT order_date) "#_visit"
FROM 
	sales
GROUP BY 
	customer_id;
```
### Result
| customer_id | #_visit |
|-------------|---------|
|A	          |4        |
|B            |6        |
|C	          |2        |
### Answer
- Customer A visited 4 days
- Customer B visited 6 days 
- Customer C visited 2 days

## 3. What was the first item from the menu purchased by each customer?
tables: sales, menu
### SQL query 
```sql
SELECT customer_id,
       product_name
FROM 
    (
     SELECT DISTINCT s.customer_id, m.product_name, s.order_date,
            RANK() OVER (PARTITION BY customer_id
                         ORDER BY s.order_date)
		   AS purchased_rank
    FROM sales AS s
    INNER JOIN menu AS m USING(product_id)
    )
WHERE 
	purchased_rank = 1
ORDER BY 
	customer_id
;
```
### Result
| customer_id | product_name |
|-------------|--------------|
|A	          |sushi         |
|A	          |curry         |
|B	          |curry         |
|C	          |ramen         |
### Answer
The first time they ordered:
- Customer A ordered sushi and curry 
- Customer B ordered curry
- Customer C ordered ramen

## 4. What is the most purchased item on the menu and how many times was it purchased by all customers?
tables: sales, menu
### SQL query
```sql
WITH 
    cte AS 
        (
         SELECT product_id,
                count(product_id) total,
                RANK() OVER(ORDER BY count(product_id) DESC) "sales_rank"               
         FROM sales
         GROUP BY product_id
         )                
SELECT m.product_name,
       cte.total
FROM menu m 
INNER JOIN cte USING(product_id)
WHERE "sales_rank" = 1     
;
```
### Result
| product_name | total      |
|--------------|------------|
|ramen	       |8           |
### Answer
The most purchased item is ramen and was purchased 8 times.
	
## 5. Which item was the most popular for each customer?
tables: sales, menu
### SQL query
```sql
WITH 
    orders AS (
               SELECT s.customer_id,
                      m.product_name,
                      rank() OVER (PARTITION BY s.customer_id
                                   ORDER BY count(s.product_id) DESC) "rank"
               FROM sales s
               INNER JOIN menu m USING(product_id)
               GROUP BY s.customer_id,
                        m.product_name
               ORDER BY s.customer_id ASC
               )
SELECT customer_id, product_name
FROM orders
WHERE "rank" = 1
;
```
### Result
| customer_id	| product_name	|
|---------------|---------------|
|A	            |ramen	        |
|B	            |sushi          |
|B	            |curry 	        |
|B              |ramen          |
|C              |ramen          |
### Answer
Ramen is the most popular item for A and C. Customers B ordered all items equally.

## 6. Which item was purchased first by the customer after they became a member?
tables: sales, menu, members
### SQL query
```sql
SELECT
    s.customer_id,
	subtable.first_order_as_member,
	m.product_name
FROM 
    (
     SELECT s.customer_id,
            min(s.order_date) as first_order_as_member
     FROM sales s
     INNER JOIN members mb ON mb.customer_id = s.customer_id
     WHERE s.order_date >= mb.join_date
     GROUP BY s.customer_id
    ) AS subtable
	
INNER JOIN sales s ON s.order_date = subtable.first_order_as_member
                  AND s.customer_id = subtable.customer_id
INNER JOIN menu m ON m.product_id = s.product_id
ORDER BY customer_id
;
```
### Result
|customer_id |	first_order_as_member |	product_name |
|------------|------------------------|--------------|
|A           |	2021-01-10            |	ramen        |
|B           |	2021-01-11            |	sushi        |
### Answer
Only customers A and B ordered after becoming members and they respectively ordered ramen and sushi.

## 7. Which item was purchased just before the customer became a member?
tables: sales, menu, members
### SQL query
```sql
WITH
    non_member AS (
                   SELECT s.customer_id,
				   s.order_date,
				   m.product_name,
				   rank() OVER (PARTITION BY s.customer_id 
								ORDER BY s.order_date DESC) chronos
                   FROM sales s
                   INNER JOIN menu m ON s.product_id = m.product_id
                   INNER JOIN members mb ON s.customer_id = mb.customer_id
                   WHERE s.order_date < mb.join_date
                   )
SELECT customer_id,
	   order_date,
	   product_name
FROM non_member
WHERE chronos = 1
;
```
### Result
|customer_id	|order_date	|product_name
|-
|A	|2021-01-01	|sushi
|A	|2021-01-01	|curry
|B	|2021-01-04	|sushi
### Answer
Just before becoming members, customer A ordered sushi and curry and customer B ordered sushi.

## 8. What is the total items and amount spent for each member before they became a member?
tables: sales, menu, members
### SQL query
```sql
SELECT s.customer_id,
       count(s.product_id) total_items,
       sum(m.price) amount_spent
FROM sales s
INNER JOIN menu m ON m.product_id = s.product_id
INNER JOIN members mb ON mb.customer_id = s.customer_id
WHERE s.order_date < mb.join_date
group by s.customer_id
order by s.customer_id
;
```
### Result
|customer_id	|total_items	|amount_spent
|-
|A	|2	|25
|B	|3	|40
### Answer
Before becoming a member:
- customer A ordered 2 items and spent 25$
- customer B ordered 3 items and spent 40$

## 9.  If each $1 spent equates to 10 points and sushi has a 2x points multiplier - how many points would each customer have?
tables: sales, menu
### SQL query
```sql
WITH cte_points AS
    (
     SELECT s.customer_id,
            CASE
                WHEN m.product_id = 1 THEN m.price * 20
                ELSE m.price * 10
            END AS points
     FROM sales s
     INNER JOIN menu m USING(product_id)
     )
SELECT customer_id, sum(points) total_points
FROM cte_points
GROUP BY customer_id
ORDER BY customer_id
;
```
### Result
|customer_id	|total_points
|-
|A	|860
|B	|940
|C	|360
### Answer
Customer A earned 860 points, Customer B 940 points and customer C 360 points.

## 10. In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi - how many points do customer A and B have at the end of January?
tables:sales, menu, members
### SQL query
```sql
WITH cte_points as 
    (
     SELECT s.customer_id, 
            CASE 
                WHEN mb.join_date <= s.order_date
                    AND order_date < (mb.join_date + 7)
                        THEN m.price * 20
                        
                 WHEN m.product_id = 1
                        THEN m.price * 20
                        
                 ELSE m.price * 10
                 END AS points   
     FROM sales s
     INNER JOIN menu m ON m.product_id = s.product_id
     INNER JOIN members mb ON mb.customer_id = s.customer_id
     WHERE (s.order_date BETWEEN '2021-01-01' AND '2021-01-31')
     )
SELECT customer_id, sum(points) member_points_in_january
FROM cte_points
GROUP BY customer_id
ORDER BY customer_id
;
```
### Result
|customer_id	|member_points_in_january
|-
|A	|1370
|B	|820
### Answer
In January, customer A earned 1370 points and customer B 820 points.

## Bonus
### Join all the things
tables: sales, menu, members
### SQL query
```sql
SELECT s.customer_id,
       s.order_date,
       m.product_name,
       m.price,
       CASE 
        WHEN s.order_date >= mb.join_date
            THEN 'Y'
        ELSE 'N'
        END AS member
FROM sales s
INNER JOIN menu m ON m.product_id = s.product_id
LEFT JOIN members mb ON mb.customer_id = s.customer_id
ORDER BY s.customer_id, s.order_date, m.product_name
;
```
### Result
| customer_id	|order_date	|product_name	|price	|member|
|---------------|-----------|---------------|-------|------|
|A	 			|2021-01-01	|curry			|15		|N	   |
|A				|2021-01-01	|sushi			|10		|N	   |
|A				|2021-01-07	|curry			|15		|Y	   |
|A				|2021-01-10	|ramen			|12		|Y	   |
|A				|2021-01-11	|ramen			|12		|Y	   |
|A				|2021-01-11	|ramen			|12		|Y	   |	
|B				|2021-01-01	|curry			|15		|N	   |
|B				|2021-01-02	|curry			|15		|N	   |
|B				|2021-01-04	|sushi			|10		|N	   |
|B				|2021-01-11	|sushi			|10		|Y	   |
|B				|2021-01-16	|ramen			|12		|Y	   |
|B				|2021-02-01	|ramen			|12		|Y	   |
|C				|2021-01-01	|ramen			|12		|N	   |
|C				|2021-01-01	|ramen			|12		|N	   |
|C				|2021-01-07	|ramen			|12		|N	   |

### Rank all the things
table: sales
```sql
WITH cte_full AS 
    (
     SELECT s.customer_id,
            s.order_date,
            m.product_name,
            m.price,
            CASE 
                WHEN s.order_date >= mb.join_date
                    THEN 'Y'
            ELSE 'N'
            END AS member
     FROM sales s
     INNER JOIN menu m ON m.product_id = s.product_id
     LEFT JOIN members mb ON mb.customer_id = s.customer_id
     ORDER BY s.customer_id, s.order_date, m.product_name
     )
SELECT *,
       CASE
        WHEN member = 'Y' THEN 
                            rank() OVER (PARTITION BY customer_id, member
                                         ORDER BY customer_id, order_date) 
        ELSE NULL
        END AS ranking
FROM cte_full
;
```
## Result
|customer_id	|order_date	|product_name	|price	|member	|ranking|
|---------------|-----------|---------------|-------|-------|-------|
|A	|2021-01-01	|curry	|15	|N	|null|
|A	|2021-01-01	|sushi	|10	|N	|null|
|A	|2021-01-07	|curry	|15	|Y	|1|
|A	|2021-01-10	|ramen	|12	|Y	|2|
|A	|2021-01-11	|ramen	|12	|Y	|3|
|A	|2021-01-11	|ramen	|12	|Y	|3|
|B	|2021-01-01	|curry	|15	|N	|null|
|B	|2021-01-02	|curry	|15	|N	|null|
|B	|2021-01-04	|sushi	|10	|N	|null|
|B	|2021-01-11	|sushi	|10	|Y	|1|
|B	|2021-01-16	|ramen	|12	|Y	|2|
|B	|2021-02-01	|ramen	|12	|Y	|3|
|C	|2021-01-01	|ramen	|12	|N	|null|
|C	|2021-01-01	|ramen	|12	|N	|null|
|C	|2021-01-07	|ramen	|12	|N	|null|
