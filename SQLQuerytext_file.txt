USE e_commerce_F;

select * from customer_orders;
select * from driver_order;
select * from ingredients;
select * from driver;
select * from rolls;
select * from rolls_recipes;

-- PART A: ROLL METRICS

--Q1. HOW MANY ROLLS WERE ORDERED AND HOW MANY TOTAL NUMBER OF ROLLS WERE ORDERED
SELECT COUNT(roll_id) AS Rolls_Ordered FROM customer_orders;
--OR
SELECT SUM(roll_id) AS TOTAL_ROLL_ORDERED FROM customer_orders;

--Q2. HOW MANY UNIQUE CUSTOMERS ORDERS WERE MADE
SELECT customer_id, COUNT(DISTINCT order_id) AS UNIQUE_CUSTOMER FROM customer_orders
GROUP BY customer_id;
--OR
SELECT COUNT(DISTINCT customer_id) AS UNIQUE_CUSTOMER FROM customer_orders;

--Q3. HOW MANY SUCCESSFUL ORDERS DELIVERED BY EACH DRIVER (DATA CLEANING IN SUBQUERY)
SELECT driver_id, COUNT(DISTINCT order_id) AS SUCCESSFUL_ORDERS FROM
(SELECT *,
CASE WHEN cancellation IN ('Cancellation','Customer Cancellation')
THEN 'C' ELSE 'NC' END AS ORDER_DETAILS
FROM driver_order) AS A
WHERE ORDER_DETAILS = 'NC'
GROUP BY driver_id;

--Q4. HOW MANY EACH TYPE OF ROLLS WERE DELIVERED (DATA CLEANING IN SUBQUERY)
SELECT roll_id, COUNT(roll_id) FROM customer_orders AS C
INNER JOIN
(SELECT *,
CASE WHEN cancellation IN ('Cancellation','Customer Cancellation')
THEN 'C' ELSE 'NC' END AS ORDER_DETAILS
FROM driver_order) AS A
ON C.order_id = A.order_id
WHERE ORDER_DETAILS = 'NC'
GROUP BY roll_id;

--Q5. HOW MANY VEG AND NON VEG ROLLS WERE ORDERED BY EACH CUSTOMER
SELECT C.customer_id, C.roll_id, COUNT(C.roll_id) AS ROLLS_COUNT, R.roll_name FROM customer_orders AS C
INNER JOIN rolls AS R
ON C.roll_id = R.roll_id
GROUP BY C.customer_id, C.roll_id, R.roll_name

--Q6. WHAT WAS THE MAXIMUM NUMBER OF ROLLS DELIVERED IN A SINGLE ORDER
SELECT * FROM 
(SELECT *, RANK() OVER(ORDER BY COUNT_OF_ORDERS DESC) AS ORDER_RANKING FROM
(SELECT order_id, COUNT(roll_id) COUNT_OF_ORDERS FROM
(SELECT B.order_id, customer_id, roll_id FROM customer_orders AS C
INNER JOIN
(SELECT order_id FROM
(SELECT *,
CASE WHEN cancellation IN ('Customer Cancellation', 'Cancellation') THEN 'C'
ELSE 'NC'
END AS Cancel_orders
FROM driver_order) AS A
WHERE Cancel_orders = 'NC') AS B
ON B.order_id = C.order_id) AS C
GROUP BY order_id) AS D) AS E
WHERE ORDER_RANKING = 1;

--Q7. FOR EACH CUSTOMER, HOW MANY DELIVERED ROLLS HAD ATLEAST 1 CHANGE AND HOW MANY HAD NO CHANGES
WITH TEMP_CUSTOMER_ORDERS_A (order_id,customer_id,roll_id,not_include_items,extra_items_included,order_date) AS
(SELECT order_id, customer_id, roll_id,
CASE 
WHEN not_include_items IS NULL OR not_include_items = ' ' THEN '0' 
ELSE not_include_items
END AS NEW_NOT_INCLUDED_ITEMS,
CASE WHEN extra_items_included IS NULL OR extra_items_included = ' ' OR extra_items_included = 'NaN' THEN '0'
ELSE extra_items_included
END AS NEW_EXTRA_ITEMS_INCLUDED, order_date
FROM customer_orders),

TEMP_DRIVER_ORDERS_A (order_id,driver_id,pickup_time,distance,duration,cancellation) AS
(SELECT	order_id, driver_id, pickup_time, distance, duration,
CASE WHEN cancellation IN ('Cancellation','Customer Cancellation') THEN 0
ELSE 1
END AS NEW_CANCELLATIONS
FROM driver_order)

SELECT A.customer_id, A.CHANGES_DONE, COUNT(order_id) AS CHANGES_NO_CHANGES_MADE FROM
(SELECT *,
CASE WHEN not_include_items = '0' AND extra_items_included = '0'
THEN 'NO_CHANGE'
ELSE 'CHANGE' 
END AS CHANGES_DONE FROM TEMP_CUSTOMER_ORDERS_A
WHERE order_id IN 
(SELECT order_id FROM TEMP_DRIVER_ORDERS_A
WHERE cancellation = 1)) AS A
GROUP BY A.customer_id, A.CHANGES_DONE;

--Q8. HOW MANY ROLLS WERE DELIVERED THAT HAD BOTH EXCLUSIONS AND EXTRAS
WITH TEMP_CUSTOMER_ORDERS_A (order_id,customer_id,roll_id,not_include_items,extra_items_included,order_date) AS
(SELECT order_id, customer_id, roll_id,
CASE 
WHEN not_include_items IS NULL OR not_include_items = ' ' THEN '0' 
ELSE not_include_items
END AS NEW_NOT_INCLUDED_ITEMS,
CASE WHEN extra_items_included IS NULL OR extra_items_included = ' ' OR extra_items_included = 'NaN' THEN '0'
ELSE extra_items_included
END AS NEW_EXTRA_ITEMS_INCLUDED, order_date
FROM customer_orders),

TEMP_DRIVER_ORDERS_A (order_id,driver_id,pickup_time,distance,duration,cancellation) AS
(SELECT	order_id, driver_id, pickup_time, distance, duration,
CASE WHEN cancellation IN ('Cancellation','Customer Cancellation') THEN 0
ELSE 1
END AS NEW_CANCELLATIONS
FROM driver_order)

SELECT A.CHANGES_DONE, COUNT(CHANGES_DONE) AS CHANGES_NO_CHANGES_MADE FROM
(SELECT *,
CASE WHEN not_include_items = '0' OR extra_items_included = '0'
THEN 'ATLEAST 1 INCLUDED OR EXCLUDED'
ELSE 'BOTH INCLUDED EXCLUDED' 
END AS CHANGES_DONE FROM TEMP_CUSTOMER_ORDERS_A
WHERE order_id IN 
(SELECT order_id FROM TEMP_DRIVER_ORDERS_A
WHERE cancellation = 1)) AS A
GROUP BY A.CHANGES_DONE;

--Q9. WHAT WAS THE TOTAL NUMBER OF ROLLS ORDERS FOR EACH HOUR OF THE DAY
SELECT A.INTERVAL, COUNT(A.INTERVAL) FROM
(SELECT *, CONCAT(DATEPART(HOUR, order_date),'-', DATEPART(HOUR, order_date) +1) AS INTERVAL FROM customer_orders) AS A
GROUP BY A.INTERVAL;

--Q10. WHAT WAS THE NUMBER OF ORDERS FOR EACH DAY OF THE WEEK
SELECT A.DAY_NAME, COUNT(DISTINCT A.order_id)  FROM 
(SELECT *, DATENAME(WEEKDAY,order_date) AS DAY_NAME FROM customer_orders) AS A
GROUP BY A.DAY_NAME;

-- PART B: DRIVER AND CUSTOMER EXPERIENCE

--Q11. WHAT WAS THE AVERAGE TIME IN MINUTES IT TOOK FOR EACH DRIVER TO ARRIVE AT FASOOS HQ TO PICK UP THE ORDER
SELECT B.driver_id, AVG(DIFFERANCE) AVERAGE_TIME FROM
(SELECT A.*, ROW_NUMBER() OVER (PARTITION BY A.order_id ORDER BY A.DIFFERANCE) AS R_NO FROM
(SELECT C.order_id, C.customer_id, C.roll_id, C.not_include_items, C.extra_items_included, C.order_date,
D.driver_id, D.pickup_time, D.distance, D.duration, D.cancellation, DATEDIFF(MINUTE, C.order_date, D.pickup_time) AS DIFFERANCE 
FROM customer_orders AS C
INNER JOIN driver_order AS D
ON C.order_id = D.order_id
WHERE D.pickup_time IS NOT NULL) AS A) AS B
WHERE R_NO = 1
GROUP BY driver_id;

--Q12. IS THERE ANY RELATIONSHIP BETWEEN THE NUMBER OF ROLLS AND HOW LONG THE ORDER TAKES TO PREPARE
SELECT *, AVG_TIME_TO_PREPARE/ROLL_COUNT*1.0 AS TIME_TO_PREP_SINGLE_ROLL FROM 
(SELECT order_id, COUNT(roll_id) AS ROLL_COUNT, SUM(DIFFERANCE*1.0) AS SUM_TIME, SUM(DIFFERANCE)/COUNT(roll_id)*1.0 AS AVG_TIME_TO_PREPARE FROM
(SELECT C.order_id, C.customer_id, C.roll_id, C.not_include_items, C.extra_items_included, C.order_date,
D.driver_id, D.pickup_time, D.distance, D.duration, D.cancellation, DATEDIFF(MINUTE, C.order_date, D.pickup_time) AS DIFFERANCE 
FROM customer_orders AS C
INNER JOIN driver_order AS D
ON C.order_id = D.order_id
WHERE D.pickup_time IS NOT NULL) AS A
GROUP BY order_id) AS B;

--Q13. WHAT WAS THE AVERAGE DISTANCE TRAVELLED FOR EACH OF THE CUSTOMER
SELECT B.customer_id , SUM(B.DISTANCE)/COUNT(B.order_id) AS DIST_SUM FROM
(SELECT *, ROW_NUMBER() OVER (PARTITION BY A.order_id ORDER BY distance) AS R_NO FROM 
(SELECT C.roll_id, C.not_include_items, C.extra_items_included, C.order_date,
D.driver_id, D.pickup_time, C.order_id, C.customer_id, CAST(TRIM(REPLACE(LOWER(D.distance),'km','')) AS DECIMAL (10,2)) AS distance, D.duration, D.cancellation 
FROM customer_orders AS C
INNER JOIN driver_order AS D
ON C.order_id = D.order_id
WHERE D.pickup_time IS NOT NULL) AS A) AS B
WHERE  R_NO = 1
GROUP BY B.customer_id;

--Q14. WHAT WAS THE DIFFERENCE BETWEEN THE LONGEST AND THE SHORTEST DELIVERY TIME FOR ALL THE ORDERS
SELECT MAX(A.DURATION) - MIN(A.DURATION) TIME_DIFFERENCE FROM
(SELECT 
CAST(CASE WHEN duration LIKE '%min%' THEN LEFT(duration, CHARINDEX('m',duration)-1)
ELSE duration
END AS INTEGER) AS DURATION
FROM driver_order
WHERE duration IS NOT NULL) AS A;

--Q15. WHAT WAS THE AVERAGE SPEED FOR EACH DRIVER FOR EACH DELIVERY AND DO YOU NOTICE ANY TREND FOR THESE VALUES
SELECT B.order_id, B.driver_id, B.DISTANCE, B.DURATION, B.SPEED_KMPH, COUNT(C.roll_id) AS ROLL_COUNT FROM customer_orders AS C
INNER JOIN
(SELECT A.*, DISTANCE/DURATION * 60 AS SPEED_KMPH FROM
(SELECT order_id, driver_id,
CAST(TRIM(REPLACE(LOWER(distance),'km','')) AS FLOAT) AS DISTANCE,
CAST(CASE WHEN duration LIKE '%m%' THEN LEFT (duration, CHARINDEX('m', duration) -1) ELSE duration END AS FLOAT) AS DURATION
FROM driver_order
WHERE duration IS NOT NULL) AS A) AS B
ON C.order_id = B.order_id
GROUP BY B.order_id, B.driver_id, B.DISTANCE, B.DURATION, B.SPEED_KMPH;

--Q16. WHAT IS SUCCESSFUL DELIVERY PERCENTAGE FOR EACH DRIVER
SELECT *, (SUCCESFUL*1.0/TOTAL*1.0)*100 AS SUCCESS_PERCENTAGE FROM
(SELECT A.driver_id, SUM(A.CANCELLED_STATUS) AS SUCCESFUL, COUNT(A.CANCELLED_STATUS) AS TOTAL FROM
(SELECT driver_id,
CASE WHEN lOWER(cancellation) LIKE '%cancel%' THEN 0 ELSE 1
END AS CANCELLED_STATUS
FROM driver_order) AS A
GROUP BY A.driver_id) AS B;

