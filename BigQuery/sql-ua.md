### 1. Зберігання даних та вибірка (Google BigQuery)

На першому етапі сирі дані з Kaggle були імпортовані до хмарного сховища **Google BigQuery**. Замість простого вивантаження, за допомогою **SQL** було реалізовано етап попередньої підготовки та інженерії ознак :

* **Оптимізація вибірки:** Відібрано лише релевантні для бізнес-аналізу метрики, що знижує витрати на збереження та обробку даних.
* **Обробка пропущених значень (Imputation):** Пропуски в стовпці `bmi` (індекс маси тіла) були автоматично заповнені середнім значенням по всій вибірці з округленням до двох знаків після коми.
* **Створення аналітичних метрик:** 
  * `age_group` та `age_category` — для дослідження вікових ризиків інсульту.
  * `glucose_status` — медичне групування пацієнтів за рівнем цукру (діабет, предіабет, норма тощо).
  * `bmi_category` — класифікація ваги (ожиріння, надлишкова вага, норма) на основі вже відновлених даних.

```sql
WITH prepared_data AS (
  SELECT
    gender,
    age,
    hypertension,
    heart_disease,
    ever_married,
    work_type,
    avg_glucose_level,
    smoking_status,
    stroke,
    -- Заповнюємо пропуски середнім значенням BMI
    COALESCE(bmi, (SELECT ROUND(AVG(bmi), 2) FROM `my-project-180500-493108.stroke.stroke` )) AS bmi_filled
  FROM `my-project-180500-493108.stroke.stroke`
)

SELECT
  *,
  -- Вікові категорії
  CASE 
    WHEN age < 1 THEN 'Infant'
    WHEN age BETWEEN 1 AND 18 THEN 'Child/Adolescent'
    ELSE 'Adult' 
  END AS age_category,
  -- Медичний статус рівня глюкози
  CASE 
    WHEN avg_glucose_level < 70 THEN 'Hypoglycemia/Low'
    WHEN avg_glucose_level BETWEEN 70 AND 99 THEN 'Normal'
    WHEN avg_glucose_level BETWEEN 100 AND 125 THEN 'Prediabetes'
    WHEN avg_glucose_level >= 126 THEN 'Diabetes'
    ELSE 'Critical/High'
  END AS glucose_status,
  -- Категорії ІМТ на основі заповнених даних
  CASE 
    WHEN bmi_filled < 18.5 THEN 'Underweight'
    WHEN bmi_filled BETWEEN 18.5 AND 24.9 THEN 'Normal'
    WHEN bmi_filled BETWEEN 25 AND 29.9 THEN 'Overweight'
    WHEN bmi_filled >= 30 THEN 'Obese'
    ELSE 'Unknown'
  END AS bmi_category,
  -- Сегментація за віковими групами
  CASE 
    WHEN age < 20 THEN '0-19'
    WHEN age BETWEEN 20 AND 39 THEN '20-39'
    WHEN age BETWEEN 40 AND 59 THEN '40-59'
    WHEN age BETWEEN 60 AND 79 THEN '60-79'
    ELSE '80+'
  END AS age_group
FROM prepared_data;
