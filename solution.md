**Задание 1**
```sql
WITH cohort_users AS (
  select
	  u.id AS user_id,--уникальный индефикатор пользователя
	  DATE(u.date_joined) AS reg_date,-- дата регистрации
	  TO_CHAR(u.date_joined, 'YYYY-MM') AS cohort_month -- месяц регистрации
  FROM users u
),
user_activity AS (
  SELECT DISTINCT--получаю все даты активности пользователей без дубликатов
	  ue.user_id,
	  DATE(ue.entry_at) AS activity_date
  FROM userentry ue
),
daily_retention AS (
  SELECT--соединяю когорты с активностью и вычисляю разницу в днях
	  c.cohort_month,
	  c.user_id,
	  a.activity_date,
	  (a.activity_date - c.reg_date) AS day_diff
  FROM cohort_users c
  LEFT JOIN user_activity a ON c.user_id = a.user_id
),
cohort_size AS (
  SELECT--считаю общее число пользователей в каждой когорте
    cohort_month,
    COUNT(DISTINCT user_id) AS total_users
  FROM cohort_users
  GROUP BY cohort_month
),
retention_calc AS (
  SELECT--для каждого фиксированного дня
    dr.cohort_month,
    COUNT(DISTINCT CASE WHEN dr.day_diff >= 0 THEN dr.user_id END) AS day0,
    COUNT(DISTINCT CASE WHEN dr.day_diff >= 1 THEN dr.user_id END) AS day1,
    COUNT(DISTINCT CASE WHEN dr.day_diff >= 3 THEN dr.user_id END) AS day3,
    COUNT(DISTINCT CASE WHEN dr.day_diff >= 7 THEN dr.user_id END) AS day7,
    COUNT(DISTINCT CASE WHEN dr.day_diff >= 14 THEN dr.user_id END) AS day14,
    COUNT(DISTINCT CASE WHEN dr.day_diff >= 30 THEN dr.user_id END) AS day30,
    COUNT(DISTINCT CASE WHEN dr.day_diff >= 60 THEN dr.user_id END) AS day60,
    COUNT(DISTINCT CASE WHEN dr.day_diff >= 90 THEN dr.user_id END) AS day90
  FROM daily_retention dr
  GROUP BY dr.cohort_month
)
SELECT--перевожу в проценты и округляю
  rc.cohort_month,
  ROUND(100.0 * rc.day0 / cs.total_users, 2) AS day0,
  ROUND(100.0 * rc.day1 / cs.total_users, 2) AS day1,
  ROUND(100.0 * rc.day3 / cs.total_users, 2) AS day3,
  ROUND(100.0 * rc.day7 / cs.total_users, 2) AS day7,
  ROUND(100.0 * rc.day14 / cs.total_users, 2) AS day14,
  ROUND(100.0 * rc.day30 / cs.total_users, 2) AS day30,
  ROUND(100.0 * rc.day60 / cs.total_users, 2) AS day60,
  ROUND(100.0 * rc.day90 / cs.total_users, 2) AS day90
FROM retention_calc rc
JOIN cohort_size cs ON rc.cohort_month = cs.cohort_month
ORDER BY rc.cohort_month;
```
**Выводы:**
1.Высокий retention в первые дни (day0-day7) наблюдается у всех когорт. Это говорит о том, что пользователи активно пробуют платформу сразу после регистрации.
2.Сильное падение к 30-90 дням. Большинство когорт теряют 70-90% пользователей уже к 30 дню

**Задание 2**

```sql
WITH user_totals AS (
  SELECT
    u.id,
    COALESCE(SUM(CASE WHEN tt.type IN (1,23,24,25,26,27,28,30) THEN t.value ELSE 0 END), 0) AS total_debit,--списания
    COALESCE(SUM(CASE WHEN tt.type NOT IN (1,23,24,25,26,27,28,30) THEN t.value ELSE 0 END), 0) AS total_credit--начисления
  FROM users u
  LEFT JOIN transaction t ON u.id = t.user_id
  LEFT JOIN transactiontype tt ON t.type_id = tt.type--связываю с типом транзакции
  GROUP BY u.id
)
SELECT--финальный расчет по всем пользователям
  ROUND(AVG(total_debit), 2) AS avg_debit_per_user,
  ROUND(AVG(total_credit), 2) AS avg_credit_per_user,
  ROUND(AVG(total_credit - total_debit), 2) AS avg_balance,
  PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY (total_credit - total_debit)) AS median_balance
FROM user_totals;
```
**Выводы:**
1. Средний баланс значительно выше медианного. Это говорит о сильном перекосе распределения: небольшая группа пользователей имеет крупные остатки,
    тогда как у большинства пользователей баланс скромный.
2. Средние начисления намного превышают средние списания. Это подтверждает главную проблему текущей модели монетизации.
3. Соотношение списаний к начислениям составляет примерно 1:12,5. То есть на каждую потраченную монету пользователь получает более 12 монет бесплатно.
   Это означает, что внутренняя валюта не выполняет свою функцию средства обмена-
   ее слишком легко заработать и слишком сложно потратить.

**Задание 3**

```sql
with -- задание 3 часть 1 решенные задачи и попытки
  task_stats AS (
    SELECT
      user_id,
      COUNT(DISTINCT CASE WHEN is_false = 0 THEN problem_id END) AS solved_tasks,
      COUNT(*) AS total_attempts
    FROM codesubmit
    GROUP BY user_id
),
test_stats AS (
  SELECT--уникальные тесты и общее число попыток
    user_id,
    COUNT(DISTINCT test_id) AS started_tests,
    COUNT(*) AS total_test_attempts
  FROM teststart
  GROUP BY user_id
),
active_users AS (
  SELECT user_id FROM task_stats--активные пользователи
  UNION
  SELECT user_id FROM test_stats
)
SELECT--финальные показатели
-- 1. Среднее решённых задач (только решавшие)
  (SELECT ROUND(AVG(solved_tasks), 2) FROM task_stats WHERE solved_tasks > 0) AS avg_solved_tasks,
-- 2. Среднее начатых тестов (только начинавшие)
  (SELECT ROUND(AVG(started_tests), 2) FROM test_stats WHERE started_tests > 0) AS avg_tests_started,
-- 3. Среднее попыток на задачу (только решавшие)
  (SELECT ROUND(AVG(total_attempts * 1.0 / solved_tasks), 2) FROM task_stats WHERE solved_tasks > 0) AS avg_attempts_per_solved_task,
-- 4. Среднее попыток на тест (только начинавшие)
  (SELECT ROUND(AVG(total_test_attempts * 1.0 / started_tests), 2) FROM test_stats WHERE started_tests > 0) AS avg_attempts_per_test,
-- 5. Доля активных от всех пользователей
  (SELECT ROUND(100.0 * COUNT(*) / (SELECT COUNT(*) FROM users), 2) FROM active_users) AS pct_active_users;
```
```sql
WITH purchases AS (
  select -- задание 3 часть 2 метрики покупок
    t.user_id,
    tt.type AS operation_type,
    COUNT(*) AS purchase_cnt
  FROM transaction t
  JOIN transactiontype tt ON t.type_id = tt.type--связь по типу транзакции
  WHERE tt.type IN (1,23,24,25,26,27,28,30) -- все списания
  GROUP BY t.user_id, tt.type
)
SELECT
-- 1. Сколько человек открывало задачи (type = 23)
  COUNT(DISTINCT CASE WHEN operation_type = 23 THEN user_id END) AS users_opened_tasks,
-- 2. Сколько человек открывало тесты (type = 24)
  COUNT(DISTINCT CASE WHEN operation_type = 24 THEN user_id END) AS users_opened_tests,
-- 3. Сколько человек открывало подсказки (type = 25)
  COUNT(DISTINCT CASE WHEN operation_type = 25 THEN user_id END) AS users_opened_hints,
-- 4. Сколько человек открывало решения (type = 26)
  COUNT(DISTINCT CASE WHEN operation_type = 26 THEN user_id END) AS users_opened_solutions,
-- 5. Сколько всего фактов открытий (сумма покупок по этим типам)
  COALESCE(SUM(CASE WHEN operation_type IN (23,24,25,26) THEN purchase_cnt ELSE 0 END), 0) AS total_openings,
-- 6. Сколько человек купило хотя бы что-то из перечисленного (23-26)
  COUNT(DISTINCT CASE WHEN operation_type IN (23,24,25,26) THEN user_id END) AS users_bought_anything,
-- 7. Сколько человек имеют хотя бы одну транзакцию (любую, включая начисления)
  (SELECT COUNT(DISTINCT user_id) FROM transaction) AS users_with_any_transaction
FROM purchases;
```
**Выводы:**
1. Задачи- основной вид контента. Пользователи активно их решают, но делают много попыток.
Внедрение платных подсказок или решений может быть востребовано и приносить доход.
2. Тесты менее популярны, но проходятся легко - их можно оставить бесплатными.
3. Высокая доля активных  пользователей - хорошая база для монетизации через подписку.
4. Важно, что решения не покупал никто, возможно они бесплатные или очень дорогие.
5. Задачи покупают чаще всего.Но при этом видно что только каждый 5-й что-то покупает.
6. Подсказки купили 151 человек - значительно меньше, чем задачи.
   Возможно, они плохо заметны в интерфейсе, либо пользователи предпочитают тратить попытки, а не монеты.
7. Тесты за монеты открывают редко. Возможно их мало или они не представляют ценности.
8. Всего факторов открытий - 2216. В среднем на одного покупателя приходится около 3,7 покупки.
   Это говорит о том, что те, кто начинает покупать, делают это неоднократно.
   Но основная масса пользователей имеет только начисления и никогда не тратят.
