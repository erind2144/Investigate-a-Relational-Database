--#1
/* What movie rating is most popular by rental hour? */

SELECT rating, rentals,
  CASE WHEN hour BETWEEN 1 AND 11 THEN hour || ' ' || 'am'
    WHEN hour = 12 THEN '12 pm'
    WHEN hour BETWEEN 13 AND 23 THEN hour - 12 || ' ' || 'pm'
    ELSE '12 am' END AS hour
FROM (
SELECT f.rating, DATE_PART('hour', r.rental_date) AS hour,
  COUNT(*) AS rentals
FROM film AS f
JOIN inventory AS i
ON f.film_id = i.film_id
JOIN rental AS r
ON r.inventory_id = i.inventory_id
GROUP BY 1, 2) AS sub;

--#2
/* Do longer movies get rented longer? */

WITH sub AS (SELECT f.title, f.length, r.rental_date, r.return_date,
      CAST((CAST(DATE_TRUNC('hour', r.return_date) AS timestamp) -
          CAST(DATE_TRUNC('hour', r.rental_date) AS timestamp)) AS text) AS rental_duration
    FROM film AS f
    JOIN inventory AS i
    ON f.film_id = i.film_id
    JOIN rental AS r
    ON i.inventory_id = r.inventory_id),

    sub2 AS (SELECT *,
      CASE WHEN rental_duration LIKE '%day%' THEN CAST(LEFT(rental_duration, STRPOS(rental_duration, ' ') - 1) AS integer)
      ELSE 0 END AS days,
      CASE WHEN LENGTH(rental_duration) > 7 THEN CAST(LEFT(RIGHT(rental_duration, 8), 2) AS integer)
      ELSE 0 END AS hours
    FROM sub)


SELECT title, CAST(length AS dec) / 60 AS movie_length, AVG((days * 24) + hours) AS avg_rental_hours
FROM sub2
GROUP BY 1, 2;

--#3
/* Which movies generate the most rental revenue within their respective categories? */

WITH sub AS (SELECT c.name, f.title, SUM(p.amount) AS revenue
    FROM category AS c
    JOIN film_category AS fc
    ON c.category_id = fc.category_id
    JOIN film AS f
    ON f.film_id = fc.film_id
    JOIN inventory AS i
    ON i.film_id = f.film_id
    JOIN rental AS r
    ON r.inventory_id = i.inventory_id
    JOIN payment AS p
    ON p.rental_id = r.rental_id
    GROUP BY 1, 2)

SELECT *
FROM (
SELECT *,
  DENSE_RANK() OVER (PARTITION BY name ORDER BY revenue DESC) AS category_rank
FROM sub) AS sub2
WHERE category_rank <= 3;

--#4
/* What types of movies do our top customers prefer? */

WITH sub AS (SELECT ca.name, cu.customer_id, cu.first_name || ' ' || cu.last_name AS customer_name,
      COUNT(*) AS num_rentals, SUM(p.amount) AS spend
    FROM customer AS cu
    JOIN rental AS r
    ON cu.customer_id = r.customer_id
    JOIN payment AS p
    ON p.rental_id = r.rental_id
    JOIN inventory AS i
    ON i.inventory_id = r.inventory_id
    JOIN film AS f
    ON f.film_id = i.film_id
    JOIN film_category AS fc
    ON fc.film_id = f.film_id
    JOIN category AS ca
    ON ca.category_id = fc.category_id
    GROUP BY 1, 2, 3)

SELECT *
FROM (
SELECT *, SUM(spend) OVER (PARTITION BY customer_id) AS total_spend,
  SUM(num_rentals) OVER (PARTITION BY customer_id) AS total_rentals
FROM sub) AS sub2
WHERE total_spend >= 100
ORDER BY 6 DESC, 4 DESC;
