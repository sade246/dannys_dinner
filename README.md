
-- Create Schema---
CREATE SCHEMA danny_diner;

CREATE TABLE sales (
  "customer_id" VARCHAR(1),
  "order_date" DATE,
  "product_id" INTEGER
);

INSERT INTO sales
  ("customer_id", "order_date", "product_id")
VALUES
  ('A', '2021-01-01', '1'),
  ('A', '2021-01-01', '2'),
  ('A', '2021-01-07', '2'),
  ('A', '2021-01-10', '3'),
  ('A', '2021-01-11', '3'),
  ('A', '2021-01-11', '3'),
  ('B', '2021-01-01', '2'),
  ('B', '2021-01-02', '2'),
  ('B', '2021-01-04', '1'),
  ('B', '2021-01-11', '1'),
  ('B', '2021-01-16', '3'),
  ('B', '2021-02-01', '3'),
  ('C', '2021-01-01', '3'),
  ('C', '2021-01-01', '3'),
  ('C', '2021-01-07', '3');
 

CREATE TABLE menu (
  "product_id" INTEGER,
  "product_name" VARCHAR(5),
  "price" INTEGER
);

INSERT INTO menu
  ("product_id", "product_name", "price")
VALUES
  ('1', 'sushi', '10'),
  ('2', 'curry', '15'),
  ('3', 'ramen', '12');
  

CREATE TABLE members (
  "customer_id" VARCHAR(1),
  "join_date" DATE
);

INSERT INTO members
  ("customer_id", "join_date")
VALUES
  ('A', '2021-01-07'),
  ('B', '2021-01-09');

  --**CASE STUDY #1 Danny's Diner**--

SELECT *
FROM dbo.members;

SELECT *
FROM dbo.menu;

SELECT *
FROM dbo.sales;

---total amount each customer spent---
SELECT customer_id, SUM(price) AS total_amount
FROM DBO.menu
JOIN dbo.sales
ON dbo.menu.product_id = dbo.sales.product_id
GROUP BY customer_id;

---number of days each customer visited restaurant---
SELECT customer_id, DATEDIFF(DAY, MIN(order_date), MAX(order_date)) AS date_difference
FROM dbo.sales
GROUP BY customer_id;

---first item purchased by each customer---
SELECT DISTINCT
customer_id, 
FIRST_VALUE(dbo.menu.product_name) OVER (PARTITION BY dbo.sales.customer_id ORDER BY dbo.sales.order_date) AS first_purchase
FROM dbo.sales
JOIN dbo.menu
ON dbo.sales.product_id = dbo.menu.product_id;

---most purchased item on menu and number of times all customers purchased it---
SELECT TOP 1
    dbo.menu.product_name AS most_purchased_item,
    COUNT(*) AS purchase_count
FROM dbo.menu 
JOIN dbo.sales
ON dbo.sales.product_id = dbo.menu.product_id
GROUP BY dbo.menu.product_name
ORDER BY purchase_count DESC;

---most popular item for each customer---
SELECT customer_id, product_name
FROM (
    SELECT 
        dbo.sales.customer_id,
        dbo.menu.product_name,
        COUNT(*) AS purchase_count,
        ROW_NUMBER() OVER (PARTITION BY dbo.sales.customer_id ORDER BY COUNT(*) DESC) AS row_num
    FROM dbo.sales 
    JOIN dbo.menu  ON dbo.sales.product_id = dbo.menu.product_id
    GROUP BY dbo.sales.customer_id, dbo.menu.product_name
) RankedItems
WHERE row_num = 1;

---first item purchased after becoming member---
SELECT DISTINCT
dbo.sales.customer_id, 
FIRST_VALUE(dbo.menu.product_name) OVER (PARTITION BY dbo.sales.customer_id ORDER BY dbo.sales.order_date) AS first_purchase
FROM dbo.sales
JOIN dbo.menu
ON dbo.sales.product_id = dbo.menu.product_id
JOIN dbo.members
ON dbo.sales.customer_id = dbo.members.customer_id
WHERE dbo.sales.order_date = (
    SELECT MIN(order_date)
    FROM dbo.sales
    WHERE dbo.sales.customer_id = dbo.members.customer_id AND order_date >= dbo.members.join_date
);

---item purchased right before customer became a member--
SELECT DISTINCT
dbo.sales.customer_id, 
FIRST_VALUE(dbo.menu.product_name) OVER (PARTITION BY dbo.sales.customer_id ORDER BY dbo.sales.order_date) AS first_purchase
FROM dbo.sales
JOIN dbo.menu
ON dbo.sales.product_id = dbo.menu.product_id
JOIN dbo.members
ON dbo.sales.customer_id = dbo.members.customer_id
WHERE dbo.sales.order_date = (
    SELECT MIN(order_date)
    FROM dbo.sales
    WHERE dbo.sales.customer_id = dbo.members.customer_id AND order_date < dbo.members.join_date
);

---total items and amount spent before join date---
SELECT 
    dbo.sales.customer_id,
    COUNT(dbo.sales.product_id) AS total_items,
    SUM(dbo.menu.price) AS total_amount
FROM dbo.sales 
JOIN dbo.menu 
ON dbo.sales.product_id = dbo.menu.product_id
JOIN dbo.members 
ON dbo.sales.customer_id = dbo.members.customer_id
WHERE dbo.sales.order_date < dbo.members.join_date
GROUP BY dbo.sales.customer_id;

---total points each customer has---
SELECT dbo.sales.customer_id,
       COUNT(dbo.menu.product_id) AS total_items,
	   SUM(dbo.menu.price) AS total_amount,
	   SUM(CASE WHEN dbo.menu.product_name = 'sushi' THEN dbo.menu.price * 20 ---2x points for sushi order
	   ELSE dbo.menu.price * 10
	   END) AS total_points
FROM dbo.menu
JOIN dbo.sales
ON dbo.menu.product_id = dbo.sales.product_id
GROUP BY dbo.sales.customer_id;

---number of points for customer A and B at end of Janauary after joining program---
SELECT dbo.sales.customer_id,
       COUNT(dbo.menu.product_id) AS total_items,
	   SUM(dbo.menu.price) AS total_amount,
	   SUM(CASE 
	   WHEN dbo.sales.order_date <= DATEADD(WEEK, 1, dbo.members.join_date) THEN dbo.menu.price * 20  --- 2x points for first week 
	   WHEN dbo.menu.product_name = 'sushi' THEN dbo.menu.price * 20  ---- 2x points for sushi
	   ELSE dbo.menu.price * 10
	   END) AS total_points
FROM dbo.menu
JOIN dbo.sales
ON dbo.menu.product_id = dbo.sales.product_id
JOIN dbo.members
ON dbo.sales.customer_id = dbo.members.customer_id
WHERE dbo.sales.order_date <= '2021-01-31'
GROUP BY dbo.sales.customer_id;

---BONUS QUESTION:JOIN--
CREATE TABLE customer_info 
  ("customer_id" VARCHAR(1),
  "order_date" DATE,
  "product_name" VARCHAR(5),
  "price" INTEGER,
  "member" VARCHAR(1)
);

INSERT INTO customer_info
  ("customer_id", "order_date", "product_name", "price", "member")
VALUES
  ('A', '2021-01-01', 'curry', '15', 'N'),
  ('A', '2021-01-01', 'sushi', '10', 'N'),
  ('A', '2021-01-07', 'curry', '15', 'Y'),
  ('A', '2021-01-10', 'ramen', '12', 'Y'),
  ('A', '2021-01-11', 'ramen', '12', 'Y'),
  ('A', '2021-01-11', 'ramen', '12', 'Y'),
  ('B', '2021-01-01', 'curry', '15', 'N'),
  ('B', '2021-01-02', 'curry', '15', 'N'),
  ('B', '2021-01-04', 'sushi', '10', 'N'),
  ('B', '2021-01-11', 'sushi', '10', 'Y'),
  ('B', '2021-01-16', 'ramen', '12', 'Y'),
  ('B', '2021-02-01', 'ramen', '12', 'Y'),
  ('C', '2021-01-01', 'ramen', '12', 'N'),
  ('C', '2021-01-01', 'ramen', '12', 'N'),
  ('C', '2021-01-07', 'ramen', '12', 'N');
 
 SELECT *
 FROM dbo.customer_info;

 ---BONUS QUESTION:RANKING--
CREATE TABLE ranking_customer 
  ("customer_id" VARCHAR(1),
  "order_date" DATE,
  "product_name" VARCHAR(5),
  "price" INTEGER,
  "member" VARCHAR(1),
  "ranking" INT NULL
);

INSERT INTO ranking_customer
  ("customer_id", "order_date", "product_name", "price", "member", "ranking")
VALUES
  ('A', '2021-01-01', 'curry', '15', 'N', null),
  ('A', '2021-01-01', 'sushi', '10', 'N', null),
  ('A', '2021-01-07', 'curry', '15', 'Y', '1'),
  ('A', '2021-01-10', 'ramen', '12', 'Y', '2'),
  ('A', '2021-01-11', 'ramen', '12', 'Y', '3'),
  ('A', '2021-01-11', 'ramen', '12', 'Y', '3'),
  ('B', '2021-01-01', 'curry', '15', 'N', null),
  ('B', '2021-01-02', 'curry', '15', 'N', null),
  ('B', '2021-01-04', 'sushi', '10', 'N', null),
  ('B', '2021-01-11', 'sushi', '10', 'Y', '1'),
  ('B', '2021-01-16', 'ramen', '12', 'Y', '2'),
  ('B', '2021-02-01', 'ramen', '12', 'Y', '3'),
  ('C', '2021-01-01', 'ramen', '12', 'N', null),
  ('C', '2021-01-01', 'ramen', '12', 'N', null),
  ('C', '2021-01-07', 'ramen', '12', 'N', null);

  SELECT *
  FROM ranking_customer;

  ----8WEEKSSQLCHALLENGE#1!!!---
