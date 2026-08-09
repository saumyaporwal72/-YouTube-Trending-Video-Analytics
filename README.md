# -YouTube-Trending-Video-Analytics
>
```sql
SELECT 
COUNT(*) 
FROM youtube_data
WHERE views IS NULL 
   OR likes IS NULL 
   OR comments IS NULL;
```  
   
## check duplicate
'''sql
SELECT video_id, COUNT(*)
FROM youtube_data
GROUP BY video_id
HAVING COUNT(*) > 1;
'''
>
## Check negative values (very important in interviews)
'''sql
SELECT *
FROM youtube_data
WHERE views < 0 
   OR likes < 0 
   OR comments < 0;
'''
>
## Total number of videos
'''sql
select count(*) from youtube_data;
'''
-- Total views
'''sql
select sum(views) as total_views from youtube_data;   
'''
--  Average engagement rate
'''sql
select avg(engagement_rate) from youtube_data;
'''
-- Top 10 most viewed videos
'''sql
select video_id, video_title, views
from youtube_data
order by views desc
limit 10;
''' 
 --  Distinct countries
 '''sql
 select distinct(country) from youtube_data;
'''
-- Trending %
'''sql
SELECT 
SUM(CASE WHEN is_trending = 'Yes' THEN 1 ELSE 0 END) * 100.0 / COUNT(*) AS trending_percentage
FROM youtube_data;
'''
-- Category performance
'''sql
SELECT category,
SUM(views) AS total_views,
AVG(engagement_rate) AS avg_engagement
FROM youtube_data
GROUP BY category
ORDER BY total_views DESC;
'''

-- country-wise demand
'''sql
SELECT country,
SUM(views) AS total_views
FROM youtube_data
GROUP BY country
ORDER BY total_views DESC;
'''
-- Duration optimization
'''sql
SELECT 
CASE 
    WHEN duration_seconds < 300 THEN 'Short'
    WHEN duration_seconds BETWEEN 300 AND 900 THEN 'Medium'
    ELSE 'Long'
END AS video_type,
AVG(engagement_rate) AS avg_engagement
FROM youtube_data
GROUP BY video_type;
'''
-- Engagement efficiency
'''sql
SELECT 
video_id,
(likes + comments) / nullif(views,0) AS engagement_efficiency
FROM youtube_data;
'''

-- Month publishing trend
'''sql
SELECT 
YEAR(publish_date) AS year,
MONTH(publish_date) AS month,
COUNT(*) AS total_videos
FROM youtube_data
GROUP BY YEAR(publish_date), MONTH(publish_date)
ORDER BY year, month;
'''
-- Videos longer than average duration
'''sql
SELECT *
FROM youtube_data
WHERE duration_seconds > (
    SELECT AVG(duration_seconds) FROM youtube_data
);
'''
-- Top Video in Each Category
'''sql
SELECT *
FROM (
    SELECT *,
           RANK() OVER (PARTITION BY category ORDER BY views DESC) AS rnk
    FROM youtube_data
) t
WHERE rnk = 1;
'''
-- Channel Contribution %
'''sql
SELECT channel_name,
SUM(views) * 100.0 / SUM(SUM(views)) OVER() AS view_percentage
FROM youtube_data
GROUP BY channel_name;
'''
-- Probability of Trending by Category
'''sql
SELECT category,
SUM(CASE WHEN is_trending = 'Yes' THEN 1 ELSE 0 END) / COUNT(*) AS trending_probability
FROM youtube_data
GROUP BY category
ORDER BY trending_probability DESC;
'''
-- Moving Average of Views (Window Function)
'''sql
SELECT 
date(publish_date),
views,
AVG(views) OVER (
    ORDER BY publish_date 
    ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
) AS moving_avg_7days
FROM youtube_data;
'''
-- Identify Outliers (High Performing Videos)
'''sql
SELECT *
FROM youtube_data
WHERE views > (
    SELECT AVG(views) + 2 * STDDEV(views)
    FROM youtube_data
);
'''
-- A/Btesting
-- Step 1: Create Duration Buckets
''''sql
SELECT 
CASE 
    WHEN duration_seconds < 300 THEN 'Short'
    WHEN duration_seconds BETWEEN 300 AND 900 THEN 'Medium'
    ELSE 'Long'
END AS duration_type,
AVG(engagement_rate) AS avg_engagement,
COUNT(*) AS total_videos
FROM youtube_data
GROUP BY duration_type;
'''sql
-- Combine duration + category
'''sql
SELECT 
category,
CASE 
    WHEN duration_seconds < 300 THEN 'Short'
    WHEN duration_seconds BETWEEN 300 AND 900 THEN 'Medium'
    ELSE 'Long'
END AS duration_type,
SUM(CASE WHEN is_trending='Yes' THEN 1 ELSE 0 END) / COUNT(*) 
AS trending_probability
FROM youtube_data
GROUP BY category, duration_type;
'''
-- Write a query to find top 3 videos per country.
'''sql
SELECT *
FROM (
    SELECT *,
           ROW_NUMBER() OVER(PARTITION BY country ORDER BY views DESC) AS rn
    FROM youtube_data
) t
WHERE rn <= 3;
'''

-- Find videos whose engagement is above category average.
'''sql
SELECT *
FROM youtube_data y
WHERE engagement_rate > (
    SELECT AVG(engagement_rate)
    FROM youtube_data
    WHERE category = y.category
);
'''
-- How would you detect if a channel is consistently high performing
'''sql
SELECT channel_name,
AVG(views) AS avg_views,
STDDEV(views) AS consistency
FROM youtube_data
GROUP BY channel_name;
-- low std and high avg
'''
-- Find videos that are top 10% by engagement within each category.
'''sql
SELECT *
FROM (
    SELECT *,
           NTILE(10) OVER (PARTITION BY category ORDER BY engagement_rate DESC) AS bucket
    FROM youtube_data
) t
WHERE bucket = 1;
'''
-- Detect sudden spike in views (day-over-day growth).
'''sql
SELECT *,
views - LAG(views) OVER (ORDER BY publish_date) AS view_growth
FROM youtube_data;
''' 
 -- cohort Analysis
 '''sql
 SELECT 
    channel_name,
    MIN(DATE(publish_date)) AS first_upload
FROM youtube_data
GROUP BY channel_name;
'''
-- Consecutive Streak Logic
-- Find channels with 3+ consecutive upload days.
'''sql
WITH ranked AS (
    SELECT 
        channel_name,
        DATE(publish_date) AS d,
        ROW_NUMBER() OVER (
            PARTITION BY channel_name ORDER BY DATE(publish_date)
        ) AS rn
    FROM youtube_data
),
grp AS (
    SELECT *,
           DATE_SUB(d, INTERVAL rn DAY) AS grp_key
    FROM ranked
)
SELECT channel_name, COUNT(*) AS streak_length
FROM grp
GROUP BY channel_name, grp_key
HAVING COUNT(*) >= 3;

-- Median Using Window (Real Way)
-- MySQL doesn’t support percentile:
'''sql
SELECT AVG(engagement_rate)
FROM (
    SELECT engagement_rate,
           ROW_NUMBER() OVER (ORDER BY engagement_rate) AS rn,
           COUNT(*) OVER () AS total
    FROM youtube_data
) t
WHERE rn IN (FLOOR((total+1)/2), FLOOR((total+2)/2));
'''
-- Cohort Analysis (Creator Maturity)
-- Question: Do older channels perform better?
'''sql
WITH first_upload AS (
    SELECT 
        channel_name,
        MIN(DATE(publish_date)) AS first_date
    FROM youtube_data
    GROUP BY channel_name
)
SELECT 
    y.channel_name,
    DATEDIFF(DATE(y.publish_date), f.first_date) AS days_since_first_upload,
    AVG(y.engagement_rate) AS avg_engagement
FROM youtube_data y
JOIN first_upload f 
ON y.channel_name = f.channel_name
GROUP BY y.channel_name, days_since_first_upload;
'''
-- Find videos that are above category median engagement.
'''sql
WITH ranked AS (
    SELECT *,
           ROW_NUMBER() OVER (PARTITION BY category ORDER BY engagement_rate) AS rn,
           COUNT(*) OVER (PARTITION BY category) AS cnt
    FROM youtube_data
)
SELECT *
FROM ranked
WHERE rn >= FLOOR(cnt/2);
'''
-- Detect 3 consecutive days where engagement dropped for a channel.
'''sql
WITH lagged AS (
    SELECT *,
           engagement_rate -
           LAG(engagement_rate) OVER (
               PARTITION BY channel_name 
               ORDER BY publish_date
           ) AS change
    FROM youtube_data
)
SELECT *
FROM lagged
WHERE change < 0;
'''
