## Solutions:

**1. What is the total amount each customer spent at the restaurant?**

````sql
SELECT sales.customer_id, 
       SUM(menu.price) AS total_spent
FROM dannys_diner.sales
JOIN dannys_diner.menu
ON sales.product_id = menu.product_id
GROUP BY customer_id
ORDER BY customer_id ASC
````
#### Result:

| customer_id | total_sales |
| ----------- | ----------- |
| A           | 76          |
| B           | 74          |
| C           | 36          |

***

**2. How many days has each customer visited the restaurant?**

````sql
SELECT customer_id,
       COUNT(DISTINCT order_date) AS times_visited
FROM dannys_diner.sales
GROUP BY customer_id
ORDER BY customer_id ASC;
````

#### Result:
| customer_id | visit_count |
| ----------- | ----------- |
| A           | 4           |
| B           | 6           |
| C           | 2           |

***

**3. What was the first item from the menu purchased by each customer?**

````sql
WITH sales_ranks AS 
(SELECT sales.customer_id,
        sales.order_date,
        menu.product_name,
        RANK() OVER (PARTITION BY sales.customer_id ORDER BY sales.order_date) AS rank
 FROM dannys_diner.sales
 JOIN dannys_diner.menu ON sales.product_id = menu.product_id
)
SELECT customer_id,
       product_name
FROM sales_ranks
WHERE rank = 1
GROUP BY customer_id, product_name;
````

#### Result:
| customer_id | product_name | 
| ----------- | -----------  |
| A           | curry        | 
| A           | sushi        | 
| B           | curry        | 
| C           | ramen        |

***

**4. What is the most purchased item on the menu and how many times was it purchased by all customers?**

````sql
*SELECT menu.product_name,
       COUNT(*) AS times_purchased
FROM dannys_diner.sales 
JOIN dannys_diner.menu
ON sales.product_id = menu.product_id
GROUP BY menu.product_name
ORDER BY times_purchased DESC
LIMIT 1;
````

#### Result:
| most_purchased | product_name | 
| -----------    | -----------  |
| 8              | ramen        |

***

**5. Which item was the most popular for each customer?**

````sql
WITH times_purchased AS
( SELECT sales.customer_id,
         menu.product_name,
         COUNT(menu.product_id) AS order_count,
         DENSE_RANK() OVER (PARTITION BY sales.customer_id ORDER BY COUNT(sales.customer_id) DESC) AS rank
 FROM dannys_diner.sales 
 JOIN dannys_diner.menu
 ON sales.product_id = menu.product_id
 GROUP BY sales.customer_id, menu.product_name
)
SELECT customer_id,
       product_name,
       order_count
FROM times_purchased
WHERE rank = 1;
````
#### Result:
| customer_id | product_name | order_count |
| ----------- | ----------   |------------ |
| A           | ramen        |  3          |
| B           | sushi        |  2          |
| B           | curry        |  2          |
| B           | ramen        |  2          |
| C           | ramen        |  3          |

***

**6. Which item was purchased first by the customer after they became a member?**

```sql
WITH first_purchase AS 
( SELECT sales.customer_id,
         sales.product_id,
         DENSE_RANK() OVER (PARTITION BY sales.customer_id ORDER BY sales.order_date) AS rank
  FROM dannys_diner.sales 
  JOIN dannys_diner.members
  ON sales.customer_id = members.customer_id
  WHERE members.join_date < sales.order_date
)
SELECT customer_id,
       product_name
FROM first_purchase
JOIN dannys_diner.menu
ON first_purchase.product_id = menu.product_id
WHERE rank = 1
ORDER BY 1;
```
#### Result:
| customer_id | product_name |
| ----------- | ----------   |
| A           | ramen        |
| B           | sushi        |

***

**7. Which item was purchased just before the customer became a member?**

````sql
WITH purchases_before_membership AS
(SELECT sales.customer_id,
         sales.product_id,
         DENSE_RANK() OVER (PARTITION BY sales.customer_id ORDER BY sales.order_date DESC) AS rank
 FROM dannys_diner.sales
 JOIN dannys_diner.members
 ON sales.customer_id = members.customer_id
 AND members.join_date > sales.order_date
)
SELECT customer_id,
       product_name
FROM purchases_before_membership
JOIN dannys_diner.menu
ON purchases_before_membership.product_id = menu.product_id
WHERE rank = 1
ORDER BY customer_id;
````

#### Answer:
| customer_id | product_name |
| ----------- | ----------   |
| A           | sushi        |
| B           | sushi        |

***

**8. What is the total items and amount spent for each member before they became a member?**

```sql
WITH purchases_before_membership AS
( SELECT sales.customer_id,
         sales.product_id
  FROM dannys_diner.sales 
  JOIN dannys_diner.members
  ON sales.customer_id = members.customer_id
  AND join_date > order_date
)
SELECT customer_id,
       COUNT(product_name) AS total_items,
       '$'|| SUM(price) AS amount_spent
FROM purchases_before_membership
JOIN dannys_diner.menu
ON purchases_before_membership.product_id = menu.product_id
GROUP BY 1
ORDER BY customer_id;
```
```sql
SELECT sales.customer_id,
       COUNT(product_name) AS total_items,
       '$'|| SUM(price) AS amount_spent
FROM dannys_diner.sales
JOIN dannys_diner.members ON sales.customer_id = members.customer_id
AND join_date > order_date
JOIN dannys_diner.menu ON sales.product_id = menu.product_id
GROUP BY sales.customer_id
ORDER BY sales.customer_id;
```

#### Result:
| customer_id | total_items | total_sales |
| ----------- | ----------  | ----------  |
| A           | 2           |  $25        |
| B           | 3           |  $40        |

***
**9. If each $1 spent equates to 10 points and sushi has a 2x points multiplier — how many points would each customer have?**

```sql
SELECT customer_id,
       SUM(CASE WHEN product_name = 'sushi' THEN price * 20
                ELSE price * 10 END) AS total_points
FROM dannys_diner.sales
JOIN dannys_diner.menu
ON sales.product_id = menu.product_id
GROUP BY customer_id
ORDER BY customer_id;
```

#### Result:
| customer_id | total_points | 
| ----------- | ----------   |
| A           | 860          |
| B           | 940          |
| C           | 360          |

***

**10. In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi — how many points do customer A and B have at the end of January?**

```sql
WITH last_date_cte AS
(SELECT customer_id,
        join_date,
        join_date + INTERVAL '6 days' AS last_programe_date
 FROM dannys_diner.members
)
SELECT sales.customer_id,
       SUM(CASE WHEN order_date BETWEEN join_date AND last_programe_date THEN 20 * price
            WHEN product_name = 'sushi' THEN 20 * price
            ELSE 10 * price END) AS total_points
FROM dannys_diner.sales
JOIN dannys_diner.menu 
ON sales.product_id = menu.product_id
JOIN last_date_cte cte
ON sales.customer_id = cte.customer_id
WHERE order_date BETWEEN '2021-01-01' AND '2021-01-31'
AND join_date <= order_date
GROUP BY sales.customer_id
ORDER BY sales.customer_id;
```

#### Result:
| customer_id | total_points | 
| ----------- | ----------   |
| A           | 1020         |
| B           | 320          |

***
## BONUS QUESTIONS

**Join All The Things**

**Recreate the table with: customer_id, order_date, product_name, price, member (Y/N)**

```sql
SELECT sales.customer_id,
       sales.order_date,
       menu.product_name,
       menu.product_name,
       menu.price,
       CASE WHEN join_date > order_date THEN 'N' 
            WHEN join_date <= order_date THEN 'Y' 
       ELSE 'N' END AS member
FROM dannys_diner.sales 
JOIN dannys_diner.menu ON sales.product_id = menu.product_id
LEFT JOIN dannys_diner.members ON sales.customer_id = members.customer_id;
```
 
#### Answer: 
| customer_id | order_date | product_name | price | member |
| ----------- | ---------- | -------------| ----- | ------ |
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
WITH program_cte AS
(SELECT sales.customer_id,
        sales.order_date,
        menu.product_name,
        menu.product_name,
        menu.price,
        CASE WHEN join_date > order_date THEN 'N' 
             WHEN join_date <= order_date THEN 'Y' 
        ELSE 'N' END AS member
 FROM dannys_diner.sales 
 JOIN dannys_diner.menu ON sales.product_id = menu.product_id
 LEFT JOIN dannys_diner.members ON sales.customer_id = members.customer_id
)
SELECT *,
       CASE WHEN member = 'Y' THEN RANK() OVER (PARTITION BY customer_id, member ORDER BY order_date)
       ELSE NULL END AS rank
FROM program_cte
```

#### Answer: 
| customer_id | order_date | product_name | price | member | ranking | 
| ----------- | ---------- | -------------| ----- | ------ |-------- |
| A           | 2021-01-01 | sushi        | 10    | N      | NULL    |
| A           | 2021-01-01 | curry        | 15    | N      | NULL    |
| A           | 2021-01-07 | curry        | 15    | Y      | 1       |
| A           | 2021-01-10 | ramen        | 12    | Y      | 2       |
| A           | 2021-01-11 | ramen        | 12    | Y      | 3       |
| A           | 2021-01-11 | ramen        | 12    | Y      | 3       |
| B           | 2021-01-01 | curry        | 15    | N      | NULL    |
| B           | 2021-01-02 | curry        | 15    | N      | NULL    |
| B           | 2021-01-04 | sushi        | 10    | N      | NULL    |
| B           | 2021-01-11 | sushi        | 10    | Y      | 1       |
| B           | 2021-01-16 | ramen        | 12    | Y      | 2       |
| B           | 2021-02-01 | ramen        | 12    | Y      | 3       |
| C           | 2021-01-01 | ramen        | 12    | N      | NULL    |
| C           | 2021-01-01 | ramen        | 12    | N      | NULL    |
| C           | 2021-01-07 | ramen        | 12    | N      | NULL    |

***









