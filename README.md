# SQL analyzing data
This project focuses on collecting and analyzing data to track account creation dynamics and email engagement metrics (sends, opens, and clicks). It evaluates user behavior across key categories, including messaging intervals, account verification status, and subscription lifecycles.
-- CTE по повідомленнях
WITH email_metrics AS (
 SELECT
  DATE_ADD(s.date, INTERVAL ems.sent_date DAY) AS date,
  sp.country AS country,
  a.send_interval,
  a.is_verified,
  a.is_unsubscribed,
  COUNT(DISTINCT ems.id_message) AS sent_msg,
  COUNT(DISTINCT eo.id_message) AS open_msg,
  COUNT(DISTINCT ev.id_message) AS visit_msg,
  0 AS account_cnt
 FROM `DA.email_sent` ems
 JOIN `DA.account_session` acs ON ems.id_account = acs.account_id
 JOIN `DA.account` a ON a.id = acs.account_id
 LEFT JOIN `DA.session_params` sp ON sp.ga_session_id = acs.ga_session_id
 JOIN `DA.session` s ON acs.ga_session_id = s.ga_session_id
 LEFT JOIN `DA.email_open` eo ON ems.id_message = eo.id_message
 LEFT JOIN `DA.email_visit` ev ON ems.id_message = ev.id_message
 GROUP BY date, country, a.send_interval, a.is_verified, a.is_unsubscribed
),
-- CTE по акаунтах
registrations AS (
 SELECT
  s.date AS date,
  sp.country AS country,
  a.send_interval,
  a.is_verified,
  a.is_unsubscribed,
  0 AS sent_msg,
  0 AS open_msg,
  0 AS visit_msg,
  COUNT(DISTINCT acs.account_id) AS account_cnt
 FROM `DA.account_session` acs
 JOIN `DA.account` a ON a.id = acs.account_id
 JOIN `DA.session` s ON acs.ga_session_id = s.ga_session_id
 LEFT JOIN `DA.session_params` sp ON sp.ga_session_id = acs.ga_session_id
 GROUP BY date, country, a.send_interval, a.is_verified, a.is_unsubscribed
),
-- Об'єднання
combined AS (
 SELECT * FROM email_metrics
 UNION ALL
 SELECT * FROM registrations
),
-- Агрегація
aggregated AS (
 SELECT
  date,
  country,
  send_interval,
  is_verified,
  is_unsubscribed,
  SUM(sent_msg) AS sent_msg,
  SUM(open_msg) AS open_msg,
  SUM(visit_msg) AS visit_msg,
  SUM(account_cnt) AS account_cnt
 FROM combined
 GROUP BY date, country, send_interval, is_verified, is_unsubscribed
),
-- Крок 1: Розрахунок загальних показників по країнах (без ранжування)
country_totals AS (
    SELECT
        *,
        -- Загальні суми по країні
        SUM(account_cnt) OVER (PARTITION BY country) AS total_country_account_cnt,
        SUM(sent_msg) OVER (PARTITION BY country) AS total_country_sent_cnt
    FROM aggregated
),
-- Крок 2: Ранжування на основі вже обчислених загальних сум
final_metrics AS (
 SELECT
  *,
  -- Рейтинг акаунтів: використовуємо обчислений total_country_account_cnt
  DENSE_RANK() OVER (ORDER BY total_country_account_cnt DESC) AS rank_total_country_account_cnt,
  -- Рейтинг листів: використовуємо обчислений total_country_sent_cnt
  DENSE_RANK() OVER (ORDER BY total_country_sent_cnt DESC) AS rank_total_country_sent_cnt
 FROM country_totals
)
-- Фінальний запит
SELECT
    date,
    country,
    send_interval,
    is_verified,
    is_unsubscribed,
    account_cnt,
    sent_msg,
    open_msg,
    visit_msg,
    total_country_account_cnt,
    total_country_sent_cnt,
    rank_total_country_account_cnt,
    rank_total_country_sent_cnt
FROM final_metrics
WHERE rank_total_country_sent_cnt <= 10 OR rank_total_country_account_cnt <= 10
ORDER BY date;
