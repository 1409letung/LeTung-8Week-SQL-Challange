# Entity Relationship Diagram
![alt text](image.png)

# Question
# Q1. What is the total amount each customer spent at the restaurant?

SELECT <br>
    s.customer_id, <br>
    SUM(m.price) AS total_sale <br>
FROM sales s <br>
    INNER JOIN menu m ON s.product_id = m.product_id <br>
GROUP BY s.customer_id <br>
ORDER BY s.customer_id ASC; <br>

# Q2. How many days has each customer visited the restaurant?

SELECT <br>
    s.customer_id, <br>
    COUNT(DISTINCT s.order_date) AS visit_count <br>
FROM sales s <br>
GROUP BY s.customer_id; <br>

# Q3. What was the first item from the menu purchased by each customer?

WITH ordered_sales AS ( <br>
  SELECT <br>
    s.customer_id, <br>
    s.order_date, <br>
    m.product_name, <br>
    DENSE_RANK() OVER ( <br>
      PARTITION BY s.customer_id <br>
      ORDER BY s.order_date <br>
    ) AS rank <br>
  FROM sales s <br>
    INNER JOIN menu m <br>
        ON s.product_id = m.product_id <br>
) <br>

SELECT <br>
  customer_id, <br>
  product_name <br>
FROM ordered_sales <br>
WHERE rank = 1 <br>
GROUP BY customer_id, product_name; <br>

# Q4. What is the most purchased item on the menu and how many times was it purchased by all customers?

SELECT <br>
    m.product_id, <br>
    m.product_name, <br>
    COUNT(s.product_id) AS number_of_purchases <br>
FROM menu m <br>
    INNER JOIN sales s <br>
        ON m.product_id = s.product_id <br>
GROUP BY m.product_id <br>
ORDER BY number_of_purchases DESC <br>
LIMIT 1; <br>

# Q5. Which item was the most popular for each customer?

WITH ordered_sales AS ( <br>
  SELECT <br>
    s.customer_id, <br>
    s.order_date, <br>
    m.product_name, <br>
    DENSE_RANK() OVER ( <br>
      PARTITION BY s.customer_id <br>
      ORDER BY s.order_date <br>
    ) AS rank <br>
  FROM sales s <br>
  INNER JOIN menu m <br>
    ON s.product_id = m.product_id <br>
) <br>

SELECT <br>
  customer_id, <br>
  product_name <br>
FROM ordered_sales <br>
WHERE rank = 1 <br>
GROUP BY customer_id, product_name; <br>

# Q6. Which item was purchased first by the customer after they became a member?

WITH joined_as_member AS ( <br>
  SELECT <br>
    m.customer_id, <br>
    s.product_id, <br>
    ROW_NUMBER() OVER ( <br>
      PARTITION BY m.customer_id <br>
      ORDER BY s.order_date <br>
    ) AS row_num <br>
  FROM members m <br>
  INNER JOIN sales s <br>
    ON m.customer_id = s.customer_id <br>
    AND s.order_date >= m.join_date <br>
) <br>

SELECT <br>
  customer_id, <br>
  product_name <br>
FROM joined_as_member j <br>
    INNER JOIN menu m <br>
        ON j.product_id = m.product_id <br>
WHERE row_num = 1 <br>
ORDER BY customer_id ASC; <br>

# Q7. Which item was purchased just before the customer became a member?

WITH purchased_prior_member AS ( <br>
  SELECT <br>
    m.customer_id, <br>
    s.product_id, <br>
    ROW_NUMBER() OVER ( <br>
      PARTITION BY m.customer_id <br>
      ORDER BY s.order_date DESC <br>
    ) AS rank <br>
  FROM members m <br>
    INNER JOIN sales s <br>
        ON m.customer_id = s.customer_id <br>
        AND s.order_date < m.join_date <br>
) <br>

SELECT <br>
  p.customer_id, <br>
  m.product_name <br>
FROM purchased_prior_member p <br>
    INNER JOIN menu m <br>
        ON p.product_id = m.product_id <br>
WHERE rank = 1 <br>
ORDER BY p.customer_id ASC; <br>

# Q8. What is the total items and amount spent for each member before they became a member?

SELECT <br>
  s.customer_id, <br>
  COUNT(s.product_id) AS total_items, <br>
  SUM(mn.price) AS total_sales <br>
FROM sales s <br>
    INNER JOIN members m <br>
        ON s.customer_id = m.customer_id <br>
        AND s.order_date < m.join_date <br>
    INNER JOIN menu mn <br>
        ON s.product_id = mn.product_id <br>
GROUP BY s.customer_id <br>
ORDER BY s.customer_id; <br>

# Q9. If each $1 spent equates to 10 points and sushi has a 2x points multiplier — how many points would each customer have?

WITH points_cte AS ( <br>
  SELECT <br>
    product_id, <br>
    CASE <br>
      WHEN product_id = 1 THEN price * 20 <br>
      ELSE price * 10 <br>
    END AS points <br>
  FROM menu <br>
) <br>

SELECT <br>
  sales.customer_id, <br>
  SUM(p_cte.points) AS total_points <br>
FROM sales s <br>
    INNER JOIN points_cte p_cte <br>
        ON s.product_id = p_cte.product_id <br>
GROUP BY s.customer_id <br>
ORDER BY s.customer_id; <br>

# Q10. In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi — how many points do customer A and B have at the end of January?

WITH dates_cte AS ( <br>
  SELECT <br>
    customer_id, <br>
    join_date, <br>
    join_date + 6 AS valid_date, <br>
    DATE_TRUNC('month', '2021-01-31'::DATE) + interval '1 month' - interval '1 day' AS last_date <br>
  FROM members <br>
) <br>

SELECT <br>
  sales.customer_id, <br>
  SUM(CASE <br>
    WHEN mn.product_name = 'sushi' <br>
        THEN 2 * 10 * mn.price <br>
    WHEN s.order_date BETWEEN d.join_date AND d.valid_date <br>
        THEN 2 * 10 * mn.price <br>
    ELSE 10 * mn.price <br>
  END) AS points <br>
FROM sales s <br>
    INNER JOIN dates_cte d <br>
        ON s.customer_id = d.customer_id <br>
        AND d.join_date <= s.order_date <br>
        AND s.order_date <= d.last_date <br>
    INNER JOIN menu mn <br>
        ON s.product_id = mn.product_id <br>
GROUP BY s.customer_id; <br>
