## 🍜 Case Study Answers
<img src="https://user-images.githubusercontent.com/81607668/127727503-9d9e7a25-93cb-4f95-8bd0-20b87cb4b459.png" alt="Image" width="500" height="520">

**1. What is the total amount each customer spent at the restaurant?**

```sql
SELECT s.customer_id, SUM(price) as amount
FROM sales s
         JOIN menu m ON s.product_id = m.product_id
GROUP BY s.customer_id
ORDER BY s.customer_id;
```
**Result**
| customer_id | total_spending |
| ----------- | -------------- |
| A           | 76             |
| B           | 74             |
| C           | 36             |

***

**2. How many days has each customer visited the restaurant?**
```sql
SELECT s.customer_id, COUNT(DISTINCT s.order_date)
FROM sales s
         JOIN menu m ON s.product_id = m.product_id
GROUP BY s.customer_id;
```
**Result**
| customer_id | frequency |
| ----------- | --------- |
| A           | 4         |
| B           | 6         |
| C           | 2         |

***

**3. What was the first item from the menu purchased by each customer?**
```sql
WITH cte AS (
        SELECT s.customer_id,
                m.product_name,
                DENSE_RANK() OVER (
                    PARTITION BY m.product_name
                    ORDER BY s.order_date
                ) AS rank
        FROM sales s
        JOIN menu m ON s.product_id = m.product_id
        GROUP BY s.customer_id, m.product_name, s.order_date)
SELECT customer_id, product_name
FROM cte
WHERE rank = 1
ORDER BY customer_id;
```
**Result**
| customer_id | product_name |
| ----------- | ------------ |
| A           | curry        |
| A           | sushi        |
| B           | curry        |
| C           | ramen        |

***

**4. What is the most purchased item on the menu and how many times was it purchased by all customers?**
```sql
SELECT s.product_id,
       COUNT(s.product_id) AS frequency
FROM sales s
         JOIN menu m ON s.product_id = m.product_id
GROUP BY s.product_id
ORDER BY frequency DESC
LIMIT 1;
```
**Result**
| product_id | frequency |
| ---------- | --------- |
| 3          | 8         |

***

**5. Which item was the most popular for each customer?**
```sql
WITH cte AS (
            SELECT s.customer_id,
                    m.product_name,
                    COUNT(s.product_id),
                    DENSE_RANK() OVER (
                        PARTITION BY s.customer_id
                        ORDER BY COUNT(s.product_id) DESC
                        ) AS rank
            FROM sales s
            JOIN menu m ON s.product_id = m.product_id
            GROUP BY s.customer_id, m.product_name
)
SELECT customer_id, product_name
FROM cte
WHERE rank = 1;
```
**Result**
| customer_id | product_name |
| ----------- | ------------ |
| A           | ramen        |
| B           | sushi        |
| B           | curry        |
| B           | ramen        |
| C           | ramen        |

***

**6. Which item was purchased first by the customer after they became a member?**

```sql
WITH cte AS (
            SELECT s.customer_id,
                    s.product_id,
                    ROW_NUMBER() OVER (
                        PARTITION BY s.customer_id
                        ORDER BY s.order_date
                        ) as rank
            FROM sales s
            JOIN members mem 
            ON s.customer_id = mem.customer_id
            WHERE s.order_date > mem.join_date
            GROUP BY s.customer_id, s.product_id, s.order_date
)
SELECT customer_id, m.product_name
FROM cte
JOIN menu m ON cte.product_id = m.product_id
WHERE rank = 1
ORDER BY customer_id;
```
**Result**
| customer_id | product_name |
| ----------- | ------------ |
| A           | sushi        |
| B           | sushi        |

***

**7. Which item was purchased just before the customer became a member?**
```sql
WITH cte AS (
            SELECT mem.customer_id,
                    s.product_id,
                    s.order_date,
                    ROW_NUMBER() OVER (
                        PARTITION BY mem.customer_id
                        ORDER BY s.order_date DESC
                        ) as rank
            FROM sales s
            JOIN members mem 
            ON s.customer_id = mem.customer_id
            WHERE s.order_date < mem.join_date
            GROUP BY 
                    mem.customer_id, 
                    s.product_id, 
                    s.order_date
)
SELECT customer_id, m.product_name
FROM cte
         JOIN menu m ON m.product_id = cte.product_id
WHERE rank = 1;
```
**Result**
| customer_id | product_name |
| ----------- | ------------ |
| A           | ramen        |
| B           | sushi        |

***

**8. What is the total items and amount spent for each member before they became a member?**
```sql
SELECT mem.customer_id, 
       SUM(m.price) as total_sales, 
       COUNT(s.product_id) as total_items
FROM sales s
         JOIN members mem ON s.customer_id = mem.customer_id
         JOIN menu m ON s.product_id = m.product_id
WHERE s.order_date < mem.join_date
GROUP BY mem.customer_id
ORDER BY mem.customer_id;
```
**Result**
| customer_id | total_sales | total_items |
| ----------- | ----------- | ----------- |
| A           | 25          | 2           |
| B           | 40          | 3           |


**9. If each $1 spent equates to 10 points and sushi has a 2x points multiplier — how many points would each customer have?**
```sql
WITH calculate_point AS (
    SELECT s.product_id,
           s.customer_id,
           CASE
                WHEN m.product_name = 'sushi' THEN m.price * 20
                ELSE m.price * 10
                END as point
    FROM sales s
    JOIN menu m
    ON s.product_id = m.product_id
)
SELECT customer_id, SUM(point) as total_point
FROM calculate_point
GROUP BY customer_id
ORDER BY customer_id;
```
**Result**
| customer_id | total_point |
| ----------- | ----------- |
| A           | 860         |
| B           | 940         |
| C           | 360         |

***

**10. In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi — how many points do customer A and B have at the end of January?**

```sql
WITH dates_cte AS (SELECT customer_id,
                          join_date,
                          join_date + 6          AS valid_date,
                          DATE_TRUNC(
                                  'month', '2021-01-31'::DATE)
                              + interval '1 month'
                              - interval '1 day' AS last_date
                   FROM dannys_diner.members)

SELECT sales.customer_id,
    SUM(CASE
        WHEN menu.product_name = 'sushi' 
            THEN 20 * menu.price
        WHEN sales.order_date 
            BETWEEN dates.join_date AND dates.valid_date 
            THEN 20 * menu.price
        ELSE 10 * menu.price END) AS points
FROM dannys_diner.sales
        INNER JOIN dates_cte AS dates
        ON sales.customer_id = dates.customer_id
                AND dates.join_date <= sales.order_date
                AND sales.order_date <= dates.last_date
         INNER JOIN dannys_diner.menu
                ON sales.product_id = menu.product_id
GROUP BY sales.customer_id;
```
**Result**
| customer_id | points |
| ----------- | ------ |
| B           | 320    |
| A           | 1020   |

***

## BONUS QUESTIONS
**Join All The Things**

**Recreate the table with: customer_id, order_date, product_name, price, member (Y/N)**
```sql
SELECT s.customer_id,
       s.order_date,
       m.product_name,
       m.price,
       CASE
           WHEN s.order_date >= mem.join_date THEN 'Y'
           ELSE 'N'
           END as member
FROM sales s
         JOIN menu m ON s.product_id = m.product_id
         JOIN members mem ON s.customer_id = mem.customer_id
ORDER BY customer_id, order_date;
```
**Result**
| customer_id | order_date | product_name | price | member |
| ----------- | ---------- | ------------ | ----- | ------ |
| A           | 2021-01-01 | sushi        | 10    | N      |
| A           | 2021-01-01 | curry        | 15    | N      |
| A           | 2021-01-07 | curry        | 15    | Y      |
| A           | 2021-01-10 | ramen        | 12    | Y      |
| A           | 2021-01-11 | ramen        | 12    | Y      |
| A           | 2021-01-11 | ramen        | 12    | Y      |
| B           | 2021-01-01 | curry        | 15    | N      |
| B           | 2021-01-02 | curry        | 15    | N      |
| B           | 2021-01-04 | sushi        | 10    | N      |
| B           | 2021-01-11 | sushi        | 10    | Y      |
| B           | 2021-01-16 | ramen        | 12    | Y      |
| B           | 2021-02-01 | ramen        | 12    | Y      |
| C           | 2021-01-01 | ramen        | 12    | N      |
| C           | 2021-01-01 | ramen        | 12    | N      |
| C           | 2021-01-07 | ramen        | 12    | N      |

***

**Rank All The Things**

**Danny also requires further information about the ```ranking``` of customer products, but he purposely does not need the ranking for non-member purchases so he expects null ```ranking``` values for the records when customers are not yet part of the loyalty program.**
```sql
WITH cte AS (
    SELECT 
        s.customer_id,
        s.order_date,
        m.product_name,
        m.price,
        CASE
            WHEN s.order_date >= mem.join_date THEN 'Y'
            ELSE 'N'
            END as member
    FROM sales s
    LEFT JOIN members mem ON s.customer_id = mem.customer_id
    JOIN menu m ON s.product_id = m.product_id
    ORDER BY customer_id, order_date)

SELECT *,
       CASE
           WHEN member = 'Y' THEN DENSE_RANK() OVER (
               PARTITION BY customer_id, member
               ORDER BY order_date
               )
           ELSE NULL
           END as ranking
FROM cte;
```
**Result**
| customer_id | order_date | product_name | price | member |
| ----------- | ---------- | ------------ | ----- | ------ |
| A           | 2021-01-01 | sushi        | 10    | N      |
| A           | 2021-01-01 | curry        | 15    | N      |
| A           | 2021-01-07 | curry        | 15    | Y      |
| A           | 2021-01-10 | ramen        | 12    | Y      |
| A           | 2021-01-11 | ramen        | 12    | Y      |
| A           | 2021-01-11 | ramen        | 12    | Y      |
| B           | 2021-01-01 | curry        | 15    | N      |
| B           | 2021-01-02 | curry        | 15    | N      |
| B           | 2021-01-04 | sushi        | 10    | N      |
| B           | 2021-01-11 | sushi        | 10    | Y      |
| B           | 2021-01-16 | ramen        | 12    | Y      |
| B           | 2021-02-01 | ramen        | 12    | Y      |
| C           | 2021-01-01 | ramen        | 12    | N      |
| C           | 2021-01-01 | ramen        | 12    | N      |
| C           | 2021-01-07 | ramen        | 12    | N      |