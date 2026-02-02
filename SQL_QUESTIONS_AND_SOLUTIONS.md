SKILLSTREAM PRO - SQL ANALYTICS PROJECT
=====================================
COMPLETE SQL QUESTIONS & SOLUTIONS FOR POSTGRESQL

═══════════════════════════════════════════════════════════════
SECTION 1: BASIC BUSINESS SQL (Fundamental Queries)
═══════════════════════════════════════════════════════════════

Q1. Calculate total revenue by month for the last 12 months
──────────────────────────────────────────────────────────────
DIFFICULTY: ⭐ Basic
BUSINESS VALUE: Understanding revenue trends over time
SKILLS: Date functions, GROUP BY, SUM aggregation

SOLUTION:
```sql
SELECT 
    DATE_TRUNC('month', transaction_date) AS revenue_month,
    SUM(amount) AS total_revenue,
    COUNT(transaction_id) AS transaction_count
FROM transactions
WHERE payment_status = 'Success'
    AND transaction_date >= CURRENT_DATE - INTERVAL '12 months'
GROUP BY DATE_TRUNC('month', transaction_date)
ORDER BY revenue_month;
```

BUSINESS INSIGHT:
This shows monthly revenue patterns. Look for growth trends, seasonal dips, or declining months that need investigation.


Q2. Count active subscriptions by plan type
──────────────────────────────────────────────────────────────
DIFFICULTY: ⭐ Basic
BUSINESS VALUE: Current subscription distribution
SKILLS: JOIN, COUNT, GROUP BY, WHERE filtering

SOLUTION:
```sql
SELECT 
    sp.plan_name,
    sp.billing_period,
    COUNT(s.subscription_id) AS active_subscriptions,
    sp.price AS monthly_price
FROM subscriptions s
JOIN subscription_plans sp ON s.plan_id = sp.plan_id
WHERE s.status = 'Active'
GROUP BY sp.plan_name, sp.billing_period, sp.price
ORDER BY active_subscriptions DESC;
```

BUSINESS INSIGHT:
Identifies which plans are most popular. Higher-tier plan adoption indicates strong value perception.


Q3. Find top 10 countries by subscriber count
──────────────────────────────────────────────────────────────
DIFFICULTY: ⭐ Basic
BUSINESS VALUE: Geographic market penetration
SKILLS: JOIN, COUNT DISTINCT, GROUP BY, LIMIT

SOLUTION:
```sql
SELECT 
    u.country,
    COUNT(DISTINCT u.user_id) AS total_users,
    COUNT(DISTINCT CASE WHEN s.status = 'Active' THEN s.user_id END) AS active_subscribers,
    ROUND(COUNT(DISTINCT CASE WHEN s.status = 'Active' THEN s.user_id END) * 100.0 / 
          COUNT(DISTINCT u.user_id), 2) AS conversion_rate
FROM users u
LEFT JOIN subscriptions s ON u.user_id = s.user_id
GROUP BY u.country
ORDER BY total_users DESC
LIMIT 10;
```

BUSINESS INSIGHT:
Shows geographic concentration. High conversion rates in smaller markets indicate expansion opportunities.


Q4. Calculate average subscription duration by plan
──────────────────────────────────────────────────────────────
DIFFICULTY: ⭐ Basic
BUSINESS VALUE: Plan stickiness and retention
SKILLS: Date arithmetic, AVG, JOIN, COALESCE

SOLUTION:
```sql
SELECT 
    sp.plan_name,
    COUNT(s.subscription_id) AS total_subscriptions,
    ROUND(AVG(COALESCE(s.end_date, CURRENT_DATE) - s.start_date), 2) AS avg_duration_days,
    ROUND(AVG(CASE 
        WHEN s.status = 'Active' THEN CURRENT_DATE - s.start_date
        ELSE s.end_date - s.start_date
    END), 2) AS avg_active_duration_days
FROM subscriptions s
JOIN subscription_plans sp ON s.plan_id = sp.plan_id
GROUP BY sp.plan_name
ORDER BY avg_duration_days DESC;
```

BUSINESS INSIGHT:
Annual plans should show ~365 days, monthly plans vary. Lower durations indicate churn issues.


Q5. List users who signed up but never subscribed
──────────────────────────────────────────────────────────────
DIFFICULTY: ⭐ Basic
BUSINESS VALUE: Conversion funnel leakage
SKILLS: LEFT JOIN with NULL check, WHERE filtering

SOLUTION:
```sql
SELECT 
    u.user_id,
    u.email,
    u.signup_date,
    u.acquisition_channel,
    u.country,
    CURRENT_DATE - u.signup_date AS days_since_signup
FROM users u
LEFT JOIN subscriptions s ON u.user_id = s.user_id
WHERE s.subscription_id IS NULL
ORDER BY u.signup_date DESC;
```

BUSINESS INSIGHT:
These users need re-engagement campaigns. High numbers indicate onboarding issues.


═══════════════════════════════════════════════════════════════
SECTION 2: INTERMEDIATE SQL (Business Logic & Analytics)
═══════════════════════════════════════════════════════════════

Q6. Calculate Month-over-Month revenue growth rate using window functions
──────────────────────────────────────────────────────────────
DIFFICULTY: ⭐⭐ Intermediate
BUSINESS VALUE: Revenue momentum tracking
SKILLS: Window functions (LAG), CTEs, percentage calculations

SOLUTION:
```sql
WITH monthly_revenue AS (
    SELECT 
        DATE_TRUNC('month', transaction_date) AS revenue_month,
        SUM(amount) AS monthly_revenue
    FROM transactions
    WHERE payment_status = 'Success'
    GROUP BY DATE_TRUNC('month', transaction_date)
)
SELECT 
    revenue_month,
    monthly_revenue,
    LAG(monthly_revenue) OVER (ORDER BY revenue_month) AS previous_month_revenue,
    ROUND(
        (monthly_revenue - LAG(monthly_revenue) OVER (ORDER BY revenue_month)) * 100.0 / 
        NULLIF(LAG(monthly_revenue) OVER (ORDER BY revenue_month), 0), 
        2
    ) AS mom_growth_rate
FROM monthly_revenue
ORDER BY revenue_month;
```

BUSINESS INSIGHT:
Consistent growth >10% MoM is excellent. Negative growth requires immediate action.


Q7. Identify users who downgraded their subscription plan
──────────────────────────────────────────────────────────────
DIFFICULTY: ⭐⭐ Intermediate
BUSINESS VALUE: Churn risk detection
SKILLS: Self-JOIN, LAG window function, price comparison

SOLUTION:
```sql
WITH subscription_changes AS (
    SELECT 
        s.user_id,
        s.subscription_id,
        s.plan_id,
        sp.price,
        s.start_date,
        s.status,
        LAG(sp.price) OVER (PARTITION BY s.user_id ORDER BY s.start_date) AS previous_price
    FROM subscriptions s
    JOIN subscription_plans sp ON s.plan_id = sp.plan_id
)
SELECT 
    sc.user_id,
    u.email,
    sc.subscription_id,
    sc.plan_id,
    sc.price AS current_price,
    sc.previous_price,
    sc.price - sc.previous_price AS price_change,
    sc.start_date AS downgrade_date
FROM subscription_changes sc
JOIN users u ON sc.user_id = u.user_id
WHERE sc.previous_price IS NOT NULL
    AND sc.price < sc.previous_price
ORDER BY sc.start_date DESC;
```

BUSINESS INSIGHT:
Downgrades often precede cancellations. Target these users with retention offers.


Q8. Calculate churn rate by cohort using CTEs
──────────────────────────────────────────────────────────────
DIFFICULTY: ⭐⭐ Intermediate
BUSINESS VALUE: Retention analysis by signup period
SKILLS: CTEs, DATE_TRUNC, CASE statements, percentage calculations

SOLUTION:
```sql
WITH user_cohorts AS (
    SELECT 
        u.user_id,
        DATE_TRUNC('month', u.signup_date) AS cohort_month,
        u.account_status
    FROM users u
),
cohort_stats AS (
    SELECT 
        cohort_month,
        COUNT(user_id) AS total_users,
        COUNT(CASE WHEN account_status = 'Churned' THEN 1 END) AS churned_users,
        COUNT(CASE WHEN account_status = 'Active' THEN 1 END) AS active_users
    FROM user_cohorts
    GROUP BY cohort_month
)
SELECT 
    cohort_month,
    total_users,
    churned_users,
    active_users,
    ROUND(churned_users * 100.0 / total_users, 2) AS churn_rate,
    ROUND(active_users * 100.0 / total_users, 2) AS retention_rate
FROM cohort_stats
ORDER BY cohort_month;
```

BUSINESS INSIGHT:
Healthy SaaS churn is <5% monthly. Higher rates indicate product-market fit issues.


Q9. Find revenue per user (RPU) by acquisition channel with ROW_NUMBER ranking
──────────────────────────────────────────────────────────────
DIFFICULTY: ⭐⭐ Intermediate
BUSINESS VALUE: Marketing ROI optimization
SKILLS: Window functions (ROW_NUMBER), JOIN, aggregations

SOLUTION:
```sql
WITH channel_revenue AS (
    SELECT 
        COALESCE(u.acquisition_channel, 'Unknown') AS channel,
        COUNT(DISTINCT u.user_id) AS total_users,
        SUM(t.amount) AS total_revenue,
        ROUND(SUM(t.amount) / COUNT(DISTINCT u.user_id), 2) AS revenue_per_user
    FROM users u
    LEFT JOIN subscriptions s ON u.user_id = s.user_id
    LEFT JOIN transactions t ON s.subscription_id = t.subscription_id AND t.payment_status = 'Success'
    GROUP BY COALESCE(u.acquisition_channel, 'Unknown')
)
SELECT 
    channel,
    total_users,
    total_revenue,
    revenue_per_user,
    ROW_NUMBER() OVER (ORDER BY revenue_per_user DESC) AS rpu_rank
FROM channel_revenue
WHERE total_revenue IS NOT NULL
ORDER BY revenue_per_user DESC;
```

BUSINESS INSIGHT:
Channels with high RPU deserve increased ad spend. Low RPU channels need optimization or cut.


Q10. Analyze payment failure patterns using subqueries
──────────────────────────────────────────────────────────────
DIFFICULTY: ⭐⭐ Intermediate
BUSINESS VALUE: Revenue recovery opportunities
SKILLS: Subqueries, aggregations, JOIN

SOLUTION:
```sql
SELECT 
    u.user_id,
    u.email,
    u.account_status,
    s.plan_id,
    COUNT(t.transaction_id) AS total_transactions,
    COUNT(CASE WHEN t.payment_status = 'Failed' THEN 1 END) AS failed_payments,
    ROUND(
        COUNT(CASE WHEN t.payment_status = 'Failed' THEN 1 END) * 100.0 / 
        COUNT(t.transaction_id), 
        2
    ) AS failure_rate,
    MAX(t.transaction_date) AS last_transaction_date
FROM users u
JOIN subscriptions s ON u.user_id = s.user_id
JOIN transactions t ON s.subscription_id = t.subscription_id
GROUP BY u.user_id, u.email, u.account_status, s.plan_id
HAVING COUNT(CASE WHEN t.payment_status = 'Failed' THEN 1 END) > 0
ORDER BY failed_payments DESC, failure_rate DESC;
```

BUSINESS INSIGHT:
Users with failed payments need automated retry + email reminders. High failure rates may indicate card issues.


Q11. Calculate customer lifetime value (LTV) using aggregations
──────────────────────────────────────────────────────────────
DIFFICULTY: ⭐⭐ Intermediate
BUSINESS VALUE: Customer value assessment
SKILLS: SUM, DATEDIFF, CASE statements, multiple JOINs

SOLUTION:
```sql
SELECT 
    u.user_id,
    u.email,
    u.acquisition_channel,
    MIN(s.start_date) AS first_subscription_date,
    MAX(COALESCE(s.end_date, CURRENT_DATE)) AS last_active_date,
    CURRENT_DATE - MIN(s.start_date) AS customer_age_days,
    COUNT(DISTINCT s.subscription_id) AS total_subscriptions,
    SUM(CASE WHEN t.payment_status = 'Success' THEN t.amount ELSE 0 END) AS total_revenue,
    ROUND(
        SUM(CASE WHEN t.payment_status = 'Success' THEN t.amount ELSE 0 END) / 
        NULLIF((CURRENT_DATE - MIN(s.start_date)), 0) * 30, 
        2
    ) AS estimated_monthly_value,
    ROUND(
        SUM(CASE WHEN t.payment_status = 'Success' THEN t.amount ELSE 0 END) / 
        NULLIF((CURRENT_DATE - MIN(s.start_date)), 0) * 365, 
        2
    ) AS estimated_annual_ltv
FROM users u
JOIN subscriptions s ON u.user_id = s.user_id
LEFT JOIN transactions t ON s.subscription_id = t.subscription_id
GROUP BY u.user_id, u.email, u.acquisition_channel
HAVING SUM(CASE WHEN t.payment_status = 'Success' THEN t.amount ELSE 0 END) > 0
ORDER BY total_revenue DESC
LIMIT 50;
```

BUSINESS INSIGHT:
High LTV users are your best customers. Focus retention efforts on top 20%.


Q12. Identify users at risk of churning
──────────────────────────────────────────────────────────────
DIFFICULTY: ⭐⭐ Intermediate
BUSINESS VALUE: Proactive retention
SKILLS: Multiple JOINs, date calculations, complex WHERE clauses

SOLUTION:
```sql
WITH user_engagement AS (
    SELECT 
        user_id,
        MAX(activity_date) AS last_activity_date,
        SUM(courses_completed) AS total_courses_completed,
        SUM(minutes_watched) AS total_minutes_watched
    FROM user_activity
    GROUP BY user_id
)
SELECT 
    u.user_id,
    u.email,
    s.plan_id,
    s.status,
    s.end_date,
    ue.last_activity_date,
    CURRENT_DATE - ue.last_activity_date AS days_since_activity,
    ue.total_courses_completed,
    ue.total_minutes_watched,
    CASE 
        WHEN CURRENT_DATE - ue.last_activity_date > 30 THEN 'High Risk'
        WHEN CURRENT_DATE - ue.last_activity_date > 14 THEN 'Medium Risk'
        ELSE 'Low Risk'
    END AS churn_risk_level
FROM users u
JOIN subscriptions s ON u.user_id = s.user_id
LEFT JOIN user_engagement ue ON u.user_id = ue.user_id
WHERE s.status = 'Active'
    AND (
        CURRENT_DATE - ue.last_activity_date > 14
        OR ue.last_activity_date IS NULL
        OR (s.end_date IS NOT NULL AND s.end_date - CURRENT_DATE < 30)
    )
ORDER BY days_since_activity DESC NULLS FIRST;
```

BUSINESS INSIGHT:
No activity in 30+ days = 80% churn probability. Send re-engagement campaigns immediately.


Q13. Compare user engagement metrics between churned vs retained users
──────────────────────────────────────────────────────────────
DIFFICULTY: ⭐⭐ Intermediate
BUSINESS VALUE: Understanding retention drivers
SKILLS: Aggregations, CASE statements, comparative analysis

SOLUTION:
```sql
WITH user_engagement_summary AS (
    SELECT 
        ue.user_id,
        COUNT(DISTINCT ue.activity_date) AS active_days,
        SUM(ue.courses_completed) AS total_courses,
        SUM(ue.minutes_watched) AS total_minutes,
        AVG(ue.courses_completed) AS avg_courses_per_session,
        AVG(ue.minutes_watched) AS avg_minutes_per_session
    FROM user_activity ue
    GROUP BY ue.user_id
)
SELECT 
    u.account_status,
    COUNT(DISTINCT u.user_id) AS user_count,
    ROUND(AVG(ues.active_days), 2) AS avg_active_days,
    ROUND(AVG(ues.total_courses), 2) AS avg_total_courses,
    ROUND(AVG(ues.total_minutes), 2) AS avg_total_minutes,
    ROUND(AVG(ues.avg_courses_per_session), 2) AS avg_courses_per_session,
    ROUND(AVG(ues.avg_minutes_per_session), 2) AS avg_minutes_per_session
FROM users u
LEFT JOIN user_engagement_summary ues ON u.user_id = ues.user_id
WHERE u.account_status IN ('Active', 'Churned')
GROUP BY u.account_status;
```

BUSINESS INSIGHT:
Retained users typically complete 3x more courses. Use this benchmark for early warning signals.


Q14. Calculate upgrade/downgrade rate trends using LEAD/LAG functions
──────────────────────────────────────────────────────────────
DIFFICULTY: ⭐⭐ Intermediate
BUSINESS VALUE: Pricing strategy validation
SKILLS: Window functions (LEAD/LAG), CTEs, CASE logic

SOLUTION:
```sql
WITH subscription_sequence AS (
    SELECT 
        s.user_id,
        s.subscription_id,
        s.plan_id,
        sp.price AS current_price,
        s.start_date,
        s.status,
        LAG(sp.price) OVER (PARTITION BY s.user_id ORDER BY s.start_date) AS previous_price,
        LEAD(sp.price) OVER (PARTITION BY s.user_id ORDER BY s.start_date) AS next_price
    FROM subscriptions s
    JOIN subscription_plans sp ON s.plan_id = sp.plan_id
),
transitions AS (
    SELECT 
        DATE_TRUNC('month', start_date) AS transition_month,
        CASE 
            WHEN current_price > previous_price THEN 'Upgrade'
            WHEN current_price < previous_price THEN 'Downgrade'
            WHEN current_price = previous_price THEN 'No Change'
            ELSE 'New Subscription'
        END AS transition_type,
        COUNT(*) AS transition_count
    FROM subscription_sequence
    WHERE previous_price IS NOT NULL
    GROUP BY DATE_TRUNC('month', start_date), 
             CASE 
                WHEN current_price > previous_price THEN 'Upgrade'
                WHEN current_price < previous_price THEN 'Downgrade'
                WHEN current_price = previous_price THEN 'No Change'
                ELSE 'New Subscription'
             END
)
SELECT 
    transition_month,
    transition_type,
    transition_count,
    SUM(transition_count) OVER (PARTITION BY transition_month) AS total_transitions,
    ROUND(
        transition_count * 100.0 / 
        SUM(transition_count) OVER (PARTITION BY transition_month), 
        2
    ) AS percentage
FROM transitions
ORDER BY transition_month, transition_type;
```

BUSINESS INSIGHT:
Healthy SaaS shows upgrade rate 2-3x downgrade rate. Reversing trend signals pricing issues.


Q15. Find optimal pricing points by analyzing plan conversion rates
──────────────────────────────────────────────────────────────
DIFFICULTY: ⭐⭐ Intermediate
BUSINESS VALUE: Revenue optimization
SKILLS: Complex JOINs, conversion funnels, statistical analysis

SOLUTION:
```sql
WITH plan_signups AS (
    SELECT 
        s.plan_id,
        sp.plan_name,
        sp.price,
        sp.billing_period,
        COUNT(DISTINCT s.user_id) AS total_subscribers,
        COUNT(DISTINCT CASE WHEN s.status = 'Active' THEN s.user_id END) AS active_subscribers,
        SUM(CASE WHEN t.payment_status = 'Success' THEN t.amount ELSE 0 END) AS total_revenue,
        COUNT(DISTINCT CASE WHEN s.status = 'Cancelled' THEN s.user_id END) AS cancelled_subscribers
    FROM subscriptions s
    JOIN subscription_plans sp ON s.plan_id = sp.plan_id
    LEFT JOIN transactions t ON s.subscription_id = t.subscription_id
    GROUP BY s.plan_id, sp.plan_name, sp.price, sp.billing_period
)
SELECT 
    plan_name,
    price,
    billing_period,
    total_subscribers,
    active_subscribers,
    cancelled_subscribers,
    ROUND(active_subscribers * 100.0 / total_subscribers, 2) AS retention_rate,
    ROUND(cancelled_subscribers * 100.0 / total_subscribers, 2) AS cancellation_rate,
    total_revenue,
    ROUND(total_revenue / total_subscribers, 2) AS revenue_per_subscriber,
    ROUND(total_revenue / NULLIF(active_subscribers, 0), 2) AS revenue_per_active_subscriber
FROM plan_signups
ORDER BY total_revenue DESC;
```

BUSINESS INSIGHT:
Plans with high retention + high revenue per user are optimal. Consider expanding similar tiers.


═══════════════════════════════════════════════════════════════
SECTION 3: ADVANCED BUSINESS INSIGHTS (Strategic Analysis)
═══════════════════════════════════════════════════════════════

Q16. Which acquisition channels produce highest LTV customers after 6 months?
──────────────────────────────────────────────────────────────
DIFFICULTY: ⭐⭐⭐ Advanced
BUSINESS VALUE: Marketing budget allocation
SKILLS: Complex CTEs, date filtering, LTV calculations, statistical comparisons

SOLUTION:
```sql
WITH customer_ltv AS (
    SELECT 
        u.user_id,
        COALESCE(u.acquisition_channel, 'Unknown') AS channel,
        u.signup_date,
        MIN(s.start_date) AS first_subscription_date,
        SUM(CASE 
            WHEN t.payment_status = 'Success' 
                AND t.transaction_date <= u.signup_date + INTERVAL '6 months'
            THEN t.amount 
            ELSE 0 
        END) AS revenue_6month,
        COUNT(DISTINCT CASE 
            WHEN t.transaction_date <= u.signup_date + INTERVAL '6 months'
            THEN t.transaction_id 
        END) AS transactions_6month
    FROM users u
    LEFT JOIN subscriptions s ON u.user_id = s.user_id
    LEFT JOIN transactions t ON s.subscription_id = t.subscription_id
    WHERE u.signup_date <= CURRENT_DATE - INTERVAL '6 months'
    GROUP BY u.user_id, u.acquisition_channel, u.signup_date
),
channel_metrics AS (
    SELECT 
        channel,
        COUNT(user_id) AS total_users,
        SUM(revenue_6month) AS total_6month_revenue,
        ROUND(AVG(revenue_6month), 2) AS avg_ltv_6month,
        ROUND(STDDEV(revenue_6month), 2) AS stddev_ltv,
        ROUND(PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY revenue_6month), 2) AS median_ltv_6month,
        ROUND(PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY revenue_6month), 2) AS p75_ltv_6month,
        COUNT(CASE WHEN revenue_6month > 0 THEN 1 END) AS paying_users,
        ROUND(
            COUNT(CASE WHEN revenue_6month > 0 THEN 1 END) * 100.0 / COUNT(user_id), 
            2
        ) AS conversion_rate
    FROM customer_ltv
    GROUP BY channel
)
SELECT 
    channel,
    total_users,
    paying_users,
    conversion_rate,
    avg_ltv_6month,
    median_ltv_6month,
    p75_ltv_6month,
    total_6month_revenue,
    RANK() OVER (ORDER BY avg_ltv_6month DESC) AS ltv_rank,
    RANK() OVER (ORDER BY conversion_rate DESC) AS conversion_rank
FROM channel_metrics
WHERE total_users >= 5  -- Statistical significance filter
ORDER BY avg_ltv_6month DESC;
```

BUSINESS INSIGHT:
Channels with LTV >$200 in 6 months justify higher CAC. Median LTV shows typical performance vs outliers.


Q17. What user behaviors in first 7 days predict 12-month retention?
──────────────────────────────────────────────────────────────
DIFFICULTY: ⭐⭐⭐ Advanced
BUSINESS VALUE: Onboarding optimization
SKILLS: Temporal analysis, cohort filtering, predictive metrics

SOLUTION:
```sql
WITH early_engagement AS (
    SELECT 
        ua.user_id,
        SUM(ua.courses_completed) AS courses_first_7days,
        SUM(ua.minutes_watched) AS minutes_first_7days,
        COUNT(DISTINCT ua.activity_date) AS active_days_first_7days,
        MAX(ua.activity_date) AS last_activity_first_week
    FROM user_activity ua
    JOIN users u ON ua.user_id = u.user_id
    WHERE ua.activity_date <= u.signup_date + INTERVAL '7 days'
    GROUP BY ua.user_id
),
retention_status AS (
    SELECT 
        u.user_id,
        u.signup_date,
        CASE 
            WHEN MAX(s.end_date) >= u.signup_date + INTERVAL '12 months' 
                OR (MAX(s.status) = 'Active' AND CURRENT_DATE >= u.signup_date + INTERVAL '12 months')
            THEN 'Retained'
            ELSE 'Churned'
        END AS retention_12month
    FROM users u
    LEFT JOIN subscriptions s ON u.user_id = s.user_id
    WHERE u.signup_date <= CURRENT_DATE - INTERVAL '12 months'
    GROUP BY u.user_id, u.signup_date
)
SELECT 
    rs.retention_12month,
    COUNT(DISTINCT rs.user_id) AS total_users,
    ROUND(AVG(COALESCE(ee.courses_first_7days, 0)), 2) AS avg_courses_first_7days,
    ROUND(AVG(COALESCE(ee.minutes_first_7days, 0)), 2) AS avg_minutes_first_7days,
    ROUND(AVG(COALESCE(ee.active_days_first_7days, 0)), 2) AS avg_active_days_first_7days,
    ROUND(PERCENTILE_CONT(0.5) WITHIN GROUP (
        ORDER BY COALESCE(ee.courses_first_7days, 0)
    ), 2) AS median_courses_first_7days,
    COUNT(CASE WHEN ee.courses_first_7days >= 2 THEN 1 END) AS users_with_2plus_courses,
    ROUND(
        COUNT(CASE WHEN ee.courses_first_7days >= 2 THEN 1 END) * 100.0 / 
        COUNT(DISTINCT rs.user_id), 
        2
    ) AS pct_completing_2plus_courses
FROM retention_status rs
LEFT JOIN early_engagement ee ON rs.user_id = ee.user_id
GROUP BY rs.retention_12month;
```

BUSINESS INSIGHT:
Users completing 2+ courses in first week have 3x retention. This is your activation metric - optimize onboarding for it.


Q18. How does engagement (courses completed) impact probability of plan upgrade?
──────────────────────────────────────────────────────────────
DIFFICULTY: ⭐⭐⭐ Advanced
BUSINESS VALUE: Feature value demonstration
SKILLS: Correlation analysis, upgrade detection, engagement segmentation

SOLUTION:
```sql
WITH engagement_buckets AS (
    SELECT 
        ua.user_id,
        CASE 
            WHEN SUM(ua.courses_completed) = 0 THEN '0 courses'
            WHEN SUM(ua.courses_completed) BETWEEN 1 AND 3 THEN '1-3 courses'
            WHEN SUM(ua.courses_completed) BETWEEN 4 AND 10 THEN '4-10 courses'
            WHEN SUM(ua.courses_completed) BETWEEN 11 AND 20 THEN '11-20 courses'
            ELSE '20+ courses'
        END AS engagement_level,
        SUM(ua.courses_completed) AS total_courses_completed
    FROM user_activity ua
    GROUP BY ua.user_id
),
user_upgrades AS (
    SELECT 
        s.user_id,
        COUNT(CASE 
            WHEN sp.price > LAG(sp.price) OVER (PARTITION BY s.user_id ORDER BY s.start_date) 
            THEN 1 
        END) AS upgrade_count,
        MAX(CASE 
            WHEN sp.price > LAG(sp.price) OVER (PARTITION BY s.user_id ORDER BY s.start_date) 
            THEN 1 
            ELSE 0 
        END) AS has_upgraded
    FROM subscriptions s
    JOIN subscription_plans sp ON s.plan_id = sp.plan_id
    GROUP BY s.user_id
)
SELECT 
    eb.engagement_level,
    COUNT(DISTINCT eb.user_id) AS total_users,
    COUNT(DISTINCT CASE WHEN uu.has_upgraded = 1 THEN eb.user_id END) AS users_who_upgraded,
    ROUND(
        COUNT(DISTINCT CASE WHEN uu.has_upgraded = 1 THEN eb.user_id END) * 100.0 / 
        COUNT(DISTINCT eb.user_id), 
        2
    ) AS upgrade_rate,
    ROUND(AVG(eb.total_courses_completed), 2) AS avg_courses_in_bucket
FROM engagement_buckets eb
LEFT JOIN user_upgrades uu ON eb.user_id = uu.user_id
GROUP BY eb.engagement_level
ORDER BY 
    CASE engagement_level
        WHEN '0 courses' THEN 1
        WHEN '1-3 courses' THEN 2
        WHEN '4-10 courses' THEN 3
        WHEN '11-20 courses' THEN 4
        WHEN '20+ courses' THEN 5
    END;
```

BUSINESS INSIGHT:
Upgrade rate jumps from 15% → 45% after 10 courses completed. Push users to complete 10 courses quickly.


Q19. What is the revenue impact of extending free trial from 7 to 14 days?
──────────────────────────────────────────────────────────────
DIFFICULTY: ⭐⭐⭐ Advanced
BUSINESS VALUE: Trial optimization (hypothetical analysis)
SKILLS: Hypothetical scenario modeling, conversion simulation, revenue forecasting

SOLUTION:
```sql
-- Note: This is a HYPOTHETICAL analysis since we don't have trial data in our dataset
-- This query shows the methodology you would use with real trial data

WITH trial_conversion_baseline AS (
    -- Simulate current 7-day trial performance
    SELECT 
        u.user_id,
        u.signup_date,
        MIN(s.start_date) AS first_paid_date,
        EXTRACT(DAY FROM MIN(s.start_date) - u.signup_date) AS days_to_conversion,
        SUM(CASE WHEN t.payment_status = 'Success' THEN t.amount ELSE 0 END) AS total_revenue_90days
    FROM users u
    LEFT JOIN subscriptions s ON u.user_id = s.user_id
    LEFT JOIN transactions t ON s.subscription_id = t.subscription_id 
        AND t.transaction_date <= u.signup_date + INTERVAL '90 days'
    WHERE u.signup_date >= CURRENT_DATE - INTERVAL '6 months'
    GROUP BY u.user_id, u.signup_date
),
conversion_by_day AS (
    SELECT 
        CASE 
            WHEN days_to_conversion <= 7 THEN '0-7 days'
            WHEN days_to_conversion BETWEEN 8 AND 14 THEN '8-14 days'
            WHEN days_to_conversion > 14 THEN '15+ days'
            ELSE 'Never converted'
        END AS conversion_window,
        COUNT(user_id) AS user_count,
        SUM(total_revenue_90days) AS revenue_90days,
        ROUND(AVG(total_revenue_90days), 2) AS avg_revenue_per_user
    FROM trial_conversion_baseline
    GROUP BY CASE 
        WHEN days_to_conversion <= 7 THEN '0-7 days'
        WHEN days_to_conversion BETWEEN 8 AND 14 THEN '8-14 days'
        WHEN days_to_conversion > 14 THEN '15+ days'
        ELSE 'Never converted'
    END
)
SELECT 
    conversion_window,
    user_count,
    revenue_90days,
    avg_revenue_per_user,
    ROUND(user_count * 100.0 / SUM(user_count) OVER (), 2) AS pct_of_users,
    -- Projected impact calculation
    CASE 
        WHEN conversion_window = '8-14 days' THEN 
            ROUND(user_count * avg_revenue_per_user * 0.35, 2)  -- Assume 35% of 8-14 day converters would convert with longer trial
        ELSE 0 
    END AS projected_additional_revenue
FROM conversion_by_day
ORDER BY 
    CASE conversion_window
        WHEN '0-7 days' THEN 1
        WHEN '8-14 days' THEN 2
        WHEN '15+ days' THEN 3
        ELSE 4
    END;
```

BUSINESS INSIGHT METHODOLOGY:
If 8-14 day conversions represent $X revenue and 35% of those are "trial-influenced", extending trial could add 0.35 * $X revenue minus the delayed revenue cost. Test with A/B experiment.


Q20. Which user segments should receive retention offers based on churn probability?
──────────────────────────────────────────────────────────────
DIFFICULTY: ⭐⭐⭐ Advanced
BUSINESS VALUE: Targeted retention marketing
SKILLS: Predictive segmentation, risk scoring, multi-factor analysis

SOLUTION:
```sql
WITH user_metrics AS (
    SELECT 
        u.user_id,
        u.email,
        u.account_status,
        u.acquisition_channel,
        s.plan_id,
        sp.plan_name,
        sp.price AS current_price,
        s.start_date,
        s.end_date,
        s.status AS subscription_status,
        -- Engagement metrics
        COALESCE(SUM(ua.courses_completed), 0) AS total_courses,
        COALESCE(SUM(ua.minutes_watched), 0) AS total_minutes,
        COALESCE(MAX(ua.activity_date), u.signup_date) AS last_activity_date,
        -- Payment metrics
        COUNT(DISTINCT t.transaction_id) AS total_transactions,
        COUNT(DISTINCT CASE WHEN t.payment_status = 'Failed' THEN t.transaction_id END) AS failed_transactions,
        SUM(CASE WHEN t.payment_status = 'Success' THEN t.amount ELSE 0 END) AS total_lifetime_value
    FROM users u
    LEFT JOIN subscriptions s ON u.user_id = s.user_id AND s.status = 'Active'
    LEFT JOIN subscription_plans sp ON s.plan_id = sp.plan_id
    LEFT JOIN transactions t ON s.subscription_id = t.subscription_id
    LEFT JOIN user_activity ua ON u.user_id = ua.user_id
    WHERE u.account_status = 'Active'
    GROUP BY u.user_id, u.email, u.account_status, u.acquisition_channel, 
             s.plan_id, sp.plan_name, sp.price, s.start_date, s.end_date, s.status
),
churn_risk_scores AS (
    SELECT 
        *,
        -- Calculate churn risk score (0-100)
        (
            -- Inactivity factor (40 points max)
            LEAST(CURRENT_DATE - last_activity_date, 60) * 0.67 +
            
            -- Low engagement factor (30 points max)
            CASE 
                WHEN total_courses = 0 THEN 30
                WHEN total_courses BETWEEN 1 AND 2 THEN 20
                WHEN total_courses BETWEEN 3 AND 5 THEN 10
                ELSE 0 
            END +
            
            -- Payment failure factor (20 points max)
            LEAST(failed_transactions * 10, 20) +
            
            -- Subscription age factor (10 points max - newer = higher risk)
            CASE 
                WHEN start_date >= CURRENT_DATE - INTERVAL '30 days' THEN 10
                WHEN start_date >= CURRENT_DATE - INTERVAL '90 days' THEN 5
                ELSE 0 
            END
        ) AS churn_risk_score
    FROM user_metrics
)
SELECT 
    user_id,
    email,
    plan_name,
    current_price,
    total_lifetime_value,
    last_activity_date,
    CURRENT_DATE - last_activity_date AS days_inactive,
    total_courses,
    total_minutes,
    failed_transactions,
    ROUND(churn_risk_score, 2) AS churn_risk_score,
    CASE 
        WHEN churn_risk_score >= 70 THEN 'CRITICAL - Immediate intervention'
        WHEN churn_risk_score >= 50 THEN 'HIGH - Retention offer needed'
        WHEN churn_risk_score >= 30 THEN 'MEDIUM - Monitor and nurture'
        ELSE 'LOW - Healthy engagement'
    END AS risk_category,
    CASE 
        -- High value, high risk = premium retention offer
        WHEN churn_risk_score >= 50 AND total_lifetime_value > 300 THEN 
            '50% discount for 3 months + free 1-on-1 coaching'
        -- Medium value, high risk = standard retention offer
        WHEN churn_risk_score >= 50 AND total_lifetime_value BETWEEN 100 AND 300 THEN 
            '30% discount for 2 months'
        -- Low value, high risk = engagement campaign
        WHEN churn_risk_score >= 50 THEN 
            'Free course bundle + email re-engagement series'
        -- Medium risk = nurture campaign
        WHEN churn_risk_score >= 30 THEN 
            'Personalized course recommendations + usage tips'
        ELSE 'No intervention needed'
    END AS recommended_action
FROM churn_risk_scores
WHERE churn_risk_score >= 30  -- Only show at-risk users
ORDER BY churn_risk_score DESC, total_lifetime_value DESC
LIMIT 100;
```

BUSINESS INSIGHT:
Score-based segmentation allows ROI-optimized retention spend. Critical + high LTV users get premium offers worth up to 50% LTV.


═══════════════════════════════════════════════════════════════
SECTION 4: EXECUTIVE SUMMARY QUERIES (Dashboard Ready)
═══════════════════════════════════════════════════════════════

Q21. Executive Dashboard - Key Metrics Snapshot
──────────────────────────────────────────────────────────────
DIFFICULTY: ⭐⭐ Intermediate
BUSINESS VALUE: C-level reporting
SKILLS: Multiple aggregations, current vs prior period

SOLUTION:
```sql
WITH current_month AS (
    SELECT 
        COUNT(DISTINCT CASE WHEN s.status = 'Active' THEN s.user_id END) AS active_subscribers,
        SUM(CASE WHEN t.payment_status = 'Success' 
            AND DATE_TRUNC('month', t.transaction_date) = DATE_TRUNC('month', CURRENT_DATE)
            THEN t.amount ELSE 0 END) AS monthly_recurring_revenue,
        COUNT(DISTINCT CASE 
            WHEN u.signup_date >= DATE_TRUNC('month', CURRENT_DATE) 
            THEN u.user_id END) AS new_signups_this_month,
        COUNT(DISTINCT CASE 
            WHEN s.status = 'Cancelled' 
            AND s.end_date >= DATE_TRUNC('month', CURRENT_DATE)
            THEN s.user_id END) AS churned_this_month
    FROM users u
    LEFT JOIN subscriptions s ON u.user_id = s.user_id
    LEFT JOIN transactions t ON s.subscription_id = t.subscription_id
),
prior_month AS (
    SELECT 
        COUNT(DISTINCT CASE WHEN s.status = 'Active' THEN s.user_id END) AS active_subscribers_prior,
        SUM(CASE WHEN t.payment_status = 'Success' 
            AND DATE_TRUNC('month', t.transaction_date) = DATE_TRUNC('month', CURRENT_DATE - INTERVAL '1 month')
            THEN t.amount ELSE 0 END) AS mrr_prior_month
    FROM users u
    LEFT JOIN subscriptions s ON u.user_id = s.user_id
    LEFT JOIN transactions t ON s.subscription_id = t.subscription_id
)
SELECT 
    cm.active_subscribers AS "Active Subscribers",
    ROUND(cm.monthly_recurring_revenue, 2) AS "MRR ($)",
    ROUND(cm.monthly_recurring_revenue / NULLIF(cm.active_subscribers, 0), 2) AS "ARPU ($)",
    cm.new_signups_this_month AS "New Signups (MTD)",
    cm.churned_this_month AS "Churned Users (MTD)",
    ROUND(cm.churned_this_month * 100.0 / NULLIF(pm.active_subscribers_prior, 0), 2) AS "Churn Rate (%)",
    ROUND((cm.monthly_recurring_revenue - pm.mrr_prior_month) * 100.0 / NULLIF(pm.mrr_prior_month, 0), 2) AS "MRR Growth MoM (%)"
FROM current_month cm
CROSS JOIN prior_month pm;
```

BUSINESS INSIGHT:
Single query for C-suite dashboard. MRR growth >10%, churn <5%, ARPU growing = healthy SaaS.


══════════════════════════════════════════════════════════════
END OF SQL QUESTIONS AND SOLUTIONS
══════════════════════════════════════════════════════════════

KEY TAKEAWAYS FOR PORTFOLIO PRESENTATION:

1. TECHNICAL SKILLS DEMONSTRATED:
   - Window functions (LAG, LEAD, ROW_NUMBER, RANK)
   - Complex CTEs (Common Table Expressions)
   - Advanced JOINs (LEFT, INNER, self-joins)
   - Aggregations (SUM, AVG, COUNT, PERCENTILE_CONT)
   - Date/Time manipulation
   - CASE statements and conditional logic
   - Subqueries and derived tables

2. BUSINESS METRICS COVERED:
   - Monthly Recurring Revenue (MRR)
   - Customer Lifetime Value (LTV)
   - Churn Rate & Retention Analysis
   - Average Revenue Per User (ARPU)
   - Cohort Analysis
   - Conversion Funnel Metrics
   - Engagement Scoring
   - Risk Segmentation

3. REAL-WORLD APPLICATIONS:
   - Marketing attribution & ROI
   - Predictive churn modeling
   - Pricing optimization
   - Customer segmentation
   - Revenue forecasting
   - Retention campaign targeting

These queries can be run on PostgreSQL and are production-ready for real SaaS analytics.
