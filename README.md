# Amazon Prime Video Advanced SQL Project

![Prime Video Logo](amazon-prime-video-logo.avif)


[Click Here to get Dataset](https://www.kaggle.com/datasets/shivamb/amazon-prime-movies-and-tv-shows)

## Overview
This project analyses the Amazon Prime Video catalogue using **SQL**. It covers creating the schema, auditing missing data (many directors, countries and dates are null), splitting multi-value genre and cast columns, and answering business questions of easy, medium and advanced difficulty. The primary goals are to practise advanced SQL and to understand how Prime's catalogue is structured by rating, genre and decade.

## Project Steps
1. **Data Exploration**: understand the columns and spot the messy ones (multi-value text, dates stored as text, missing values).
2. **Schema and Load**: create the table(s) and import the CSV.
3. **Querying the Data**: solve the 14 questions below in order (**easy**, **medium**, **advanced**).
4. **Insights and Visualization**: summarise findings and optionally build a dashboard.

---

## Schema
```sql
DROP TABLE IF EXISTS amazon_prime;
CREATE TABLE amazon_prime (
    show_id      VARCHAR(10),
    type         VARCHAR(10),
    title        VARCHAR(250),
    director     VARCHAR(550),
    casts        VARCHAR(1050),
    country      VARCHAR(550),
    date_added   VARCHAR(55),
    release_year INT,
    rating       VARCHAR(15),   -- '13+', '16+', '18+', 'ALL', '7+' ...
    duration     VARCHAR(15),
    listed_in    VARCHAR(250),
    description  VARCHAR(550)
);
```
> Prime data has many missing `country`, `director` and `date_added` values, which makes it a good dataset for data-quality checks.

## 14 Practice Questions

### Easy Level
1. Count the number of Movies vs TV Shows on Prime Video.
```sql
SELECT type, COUNT(*) AS total_titles
FROM amazon_prime
GROUP BY type;
```
2. List all titles released in 2019.
```sql
SELECT title, type
FROM amazon_prime
WHERE release_year = 2019
ORDER BY title;
```
3. Count the number of titles for each rating.
```sql
SELECT rating, COUNT(*) AS total_titles
FROM amazon_prime
GROUP BY rating
ORDER BY total_titles DESC;
```
4. Find all TV shows with more than 3 seasons.
```sql
SELECT title, duration
FROM amazon_prime
WHERE type = 'TV Show'
  AND SPLIT_PART(duration, ' ', 1)::INT > 3
ORDER BY SPLIT_PART(duration, ' ', 1)::INT DESC;
```
5. List all titles that are listed under the Documentary genre.
```sql
SELECT title, type, release_year
FROM amazon_prime
WHERE listed_in ILIKE '%Documentary%';
```

### Medium Level
6. Find the top 10 directors with the most titles on Prime Video.
```sql
SELECT TRIM(UNNEST(STRING_TO_ARRAY(director, ','))) AS director,
       COUNT(*) AS total_titles
FROM amazon_prime
WHERE director IS NOT NULL
GROUP BY 1
ORDER BY total_titles DESC
LIMIT 10;
```
7. Calculate the average movie duration (in minutes) for each of the last 10 release years.
```sql
SELECT release_year,
       ROUND(AVG(SPLIT_PART(duration, ' ', 1)::INT), 1) AS avg_minutes
FROM amazon_prime
WHERE type = 'Movie' AND duration IS NOT NULL
  AND release_year >= (SELECT MAX(release_year) - 9 FROM amazon_prime)
GROUP BY release_year
ORDER BY release_year DESC;
```
8. Find the top 5 genres by number of titles.
```sql
SELECT TRIM(UNNEST(STRING_TO_ARRAY(listed_in, ','))) AS genre,
       COUNT(*) AS total_titles
FROM amazon_prime
GROUP BY 1
ORDER BY total_titles DESC
LIMIT 5;
```
9. Run a data-quality audit: how many titles have a missing director, cast, country or date_added?
```sql
SELECT COUNT(*) AS total_rows,
       COUNT(*) FILTER (WHERE director   IS NULL) AS missing_director,
       COUNT(*) FILTER (WHERE casts      IS NULL) AS missing_cast,
       COUNT(*) FILTER (WHERE country    IS NULL) AS missing_country,
       COUNT(*) FILTER (WHERE date_added IS NULL) AS missing_date_added
FROM amazon_prime;
```
10. Group titles into audience segments: Kids (ALL, 7+), Teen (13+, 16+) and Adult (18+), and count each segment.
```sql
SELECT CASE
         WHEN rating IN ('ALL', '7+')   THEN 'Kids'
         WHEN rating IN ('13+', '16+')  THEN 'Teen'
         WHEN rating = '18+'            THEN 'Adult'
         ELSE 'Other / Unrated'
       END AS audience_segment,
       COUNT(*) AS total_titles
FROM amazon_prime
GROUP BY 1
ORDER BY total_titles DESC;
```

### Advanced Level
11. Find the top 3 genres within each rating using `DENSE_RANK`.
```sql
WITH genre_rating AS (
    SELECT rating,
           TRIM(UNNEST(STRING_TO_ARRAY(listed_in, ','))) AS genre
    FROM amazon_prime
    WHERE rating IS NOT NULL
),
counted AS (
    SELECT rating, genre, COUNT(*) AS total_titles,
           DENSE_RANK() OVER (PARTITION BY rating ORDER BY COUNT(*) DESC) AS rnk
    FROM genre_rating
    GROUP BY rating, genre
)
SELECT rating, genre, total_titles
FROM counted
WHERE rnk <= 3
ORDER BY rating, rnk;
```
12. Find the most frequent cast member in each decade of release.
```sql
WITH actors AS (
    SELECT (release_year / 10) * 10 AS decade,
           TRIM(UNNEST(STRING_TO_ARRAY(casts, ','))) AS actor
    FROM amazon_prime
    WHERE casts IS NOT NULL
),
counted AS (
    SELECT decade, actor, COUNT(*) AS appearances,
           RANK() OVER (PARTITION BY decade ORDER BY COUNT(*) DESC) AS rnk
    FROM actors
    GROUP BY decade, actor
)
SELECT decade, actor, appearances
FROM counted
WHERE rnk = 1
ORDER BY decade;
```
13. Use a CTE to find genres whose average movie duration is above the overall average movie duration.
```sql
WITH movie_genres AS (
    SELECT TRIM(UNNEST(STRING_TO_ARRAY(listed_in, ','))) AS genre,
           SPLIT_PART(duration, ' ', 1)::INT AS minutes
    FROM amazon_prime
    WHERE type = 'Movie' AND duration IS NOT NULL
)
SELECT genre, ROUND(AVG(minutes), 1) AS avg_minutes
FROM movie_genres
GROUP BY genre
HAVING AVG(minutes) > (SELECT AVG(minutes) FROM movie_genres)
ORDER BY avg_minutes DESC;
```
14. Calculate the running total of titles added to Prime Video by year.
```sql
WITH yearly AS (
    SELECT EXTRACT(YEAR FROM TO_DATE(TRIM(date_added), 'Month DD, YYYY'))::INT AS year_added,
           COUNT(*) AS titles_added
    FROM amazon_prime
    WHERE date_added IS NOT NULL
    GROUP BY 1
)
SELECT year_added,
       titles_added,
       SUM(titles_added) OVER (ORDER BY year_added) AS running_total
FROM yearly
ORDER BY year_added;
```

---

## Technology Stack
- **Database**: PostgreSQL
- **SQL Concepts**: DDL, DML, Aggregations, CTEs, Window Functions (`DENSE_RANK`, running totals), `UNNEST` / `STRING_TO_ARRAY`, `CASE`, `FILTER`, data-quality checks
- **Tools**: pgAdmin 4 (or any SQL editor), PostgreSQL (via Docker, Homebrew or direct installation)

## How to Run the Project
1. Install PostgreSQL and pgAdmin (if not already installed).
2. Create a database, e.g. `CREATE DATABASE streaming_sql;`.
3. Download the dataset from the link at the top and run the `CREATE TABLE` script above.
4. Import the data:
   ```sql
   \copy amazon_prime FROM 'amazon_prime_titles.csv' WITH (FORMAT csv, HEADER true);
   ```
5. Execute the queries in order and compare the results with your own exploratory analysis.
6. Explore indexing and query optimisation techniques.

## Key Insights You Can Report
- How complete the data is (missing director, cast, country, date).
- Audience segments (Kids, Teen, Adult) based on ratings.
- Top genres within each rating and the most frequent cast per decade.
- How the catalogue has grown over time (running total).

---

## Next Steps
- **Visualize the Data**: Build a Power BI or Tableau dashboard (genre mix by rating, catalogue growth).
- **Clean the Data**: Fill or flag missing values and document your assumptions.
- **Optimize**: Add indexes on `type`, `rating` and `release_year`.
- **Expand**: Compare with Netflix and Disney+ (see the cross-platform project).
