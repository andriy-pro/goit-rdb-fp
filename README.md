# Фінальний проєкт — Аналіз захворюваності інфекційними хворобами

Коротко: імпорт та нормалізація даних з CSV файлу, аналіз даних (статистика
захворюваності), створення функцій для роботи з даними та візуалізація
результатів.

## Опис домашнього завдання

У рамках фінального проєкту ви отримаєте можливість застосувати всі навички
роботи з реляційними базами даних: від імпорту та нормалізації даних до
створення складних запитів та користувацьких функцій. Проєкт включає роботу з
реальними даними про захворюваність інфекційними хворобами у різних країнах
світу за період 1980-2020 років.

**Що потрібно зробити**

1. **Аналіз структури даних** - дослідити структуру імпортованих даних з CSV
   файлу, перевірити кількість записів та наявність пропущених значень.

2. **Нормалізація даних** - привести дані до третьої нормальної форми (3НФ)
   шляхом створення окремих таблиць для країн та випадків захворювань,
   встановити відповідні зв'язки між таблицями.

3. **Аналіз даних з агрегатними функціями** - виконати аналіз захворюваності за
   різними показниками: середні значення, максимуми, мінімуми, загальні суми для
   кожної країни та року.

4. **Створення колонок з різницею в роках** - додати розрахунки різниці між
   поточним роком та роком даних для кожного запису, використовуючи функції дати
   та часу.

5. **Створення та використання власної функції** - розробити функцію для
   розрахунку періодичних показників захворюваності (місячних, квартальних тощо)
   та застосувати її до даних.

---

## Корисні запити для діагностики

### Перевірка імпорту даних

```sql
-- Перевірка кількості імпортованих записів
SELECT COUNT(*) as total_records FROM infectious_cases;

-- Перевірка наявності NULL значень у ключових полях
SELECT
    COUNT(*) as total_rows,
    SUM(CASE WHEN Entity IS NULL THEN 1 ELSE 0 END) as null_entity,
    SUM(CASE WHEN Code IS NULL THEN 1 ELSE 0 END) as null_code,
    SUM(CASE WHEN Year IS NULL THEN 1 ELSE 0 END) as null_year
FROM infectious_cases;
```

### Перевірка нормалізації

```sql
-- Структура всіх таблиць після нормалізації
SHOW TABLES;
DESCRIBE countries;
DESCRIBE infectious_cases_normalized;

-- Підрахунок записів у нормалізованих таблицях
SELECT
    (SELECT COUNT(*) FROM infectious_cases) as original_records,
    (SELECT COUNT(*) FROM countries) as countries_count,
    (SELECT COUNT(*) FROM infectious_cases_normalized) as normalized_records;
```

## Скріншоти виконання

### 1. Імпорт схеми та даних

Імпортовані даних з CSV файлу `infectious_cases.csv` через мастер MySQL Workbench. Дані
містять інформацію про 177 країн за період 1980-2020 років (7271 запис).

![1_1_schema_import.png](screenshots/1_1_schema_import.png)

### 2.1 Опис структури таблиці

Аналіз структури імпортованих даних - перевірка типів колонок та кількості
записів.

```sql
DESCRIBE infectious_cases;

SELECT COUNT(*) as total_records FROM infectious_cases;
```

![2_1_describe_infectious_cases.png](screenshots/2_1_describe_infectious_cases.png)

### 2.2 Нормалізація даних

Перетворення даних до третьої нормальної форми шляхом створення окремих таблиць
для країн та випадків захворювань.

```sql
-- Створення таблиці країн
CREATE TABLE countries (
    id INT PRIMARY KEY AUTO_INCREMENT,
    entity VARCHAR(255) NOT NULL,
    code VARCHAR(10),
    UNIQUE KEY unique_entity_code (entity, code)
);

-- Заповнення таблиці країн
INSERT INTO countries (entity, code)
SELECT DISTINCT Entity, IF(Code = '', NULL, Code) as code
FROM infectious_cases
WHERE Entity IS NOT NULL;

-- Створення нормалізованої таблиці
CREATE TABLE infectious_cases_normalized (
    id INT PRIMARY KEY AUTO_INCREMENT,
    country_id INT NOT NULL,
    year INT NOT NULL,
    number_yaws VARCHAR(50),
    polio_cases VARCHAR(50),
    cases_guinea_worm VARCHAR(50),
    number_rabies VARCHAR(50),
    number_malaria VARCHAR(50),
    number_hiv VARCHAR(50),
    number_tuberculosis VARCHAR(50),
    number_smallpox VARCHAR(50),
    number_cholera_cases VARCHAR(50),
    FOREIGN KEY (country_id) REFERENCES countries(id),
    UNIQUE KEY unique_country_year (country_id, year)
);

-- Заповнення нормалізованої таблиці
INSERT INTO infectious_cases_normalized (
    country_id, year, number_yaws, polio_cases, cases_guinea_worm,
    number_rabies, number_malaria, number_hiv, number_tuberculosis,
    number_smallpox, number_cholera_cases
)
SELECT
    c.id,
    ic.Year,
    ic.Number_yaws,
    ic.polio_cases,
    ic.cases_guinea_worm,
    ic.Number_rabies,
    ic.Number_malaria,
    ic.Number_hiv,
    ic.Number_tuberculosis,
    ic.Number_smallpox,
    ic.Number_cholera_cases
FROM infectious_cases ic
JOIN countries c ON ic.Entity = c.entity AND (ic.Code = c.code OR (ic.Code IS NULL AND c.code IS NULL));
```

![2_2_normalization.png](screenshots/2_2_normalization.png)

### 3. Аналіз даних з агрегатними функціями

Розрахунок статистичних показників захворюваності для кожної країни: середні
значення, максимуми, мінімуми та загальні суми випадків сказу.

```sql
SELECT
    c.entity,
    c.code,
    AVG(CAST(NULLIF(icn.number_rabies, '') AS DECIMAL(10,2))) as avg_rabies,
    MIN(CAST(NULLIF(icn.number_rabies, '') AS DECIMAL(10,2))) as min_rabies,
    MAX(CAST(NULLIF(icn.number_rabies, '') AS DECIMAL(10,2))) as max_rabies,
    SUM(CAST(NULLIF(icn.number_rabies, '') AS DECIMAL(10,2))) as total_rabies,
    COUNT(CASE WHEN NULLIF(icn.number_rabies, '') IS NOT NULL THEN 1 END) as valid_records
FROM infectious_cases_normalized icn
JOIN countries c ON icn.country_id = c.id
WHERE NULLIF(icn.number_rabies, '') IS NOT NULL
GROUP BY c.entity
ORDER BY avg_rabies DESC
LIMIT 10;
```

![3_data_analysis.png](screenshots/3_data_analysis.png)

### 4. Створення колонок з різницею в роках

Додавання розрахунків різниці між поточним роком та роком даних для кожного
запису.

```sql
SELECT
    icn.id,
    icn.year,
    MAKEDATE(icn.year, 1) as first_january_date,
    CURDATE() as today_date,
    TIMESTAMPDIFF(YEAR, MAKEDATE(icn.year, 1), CURDATE()) as years_difference
FROM infectious_cases_normalized icn
ORDER BY icn.year DESC
LIMIT 10;
```

![4_year_difference.png](screenshots/4_year_difference.png)

### 5.1 Створення власної функції

Розробка функції для розрахунку періодичних показників захворюваності.

```sql
DELIMITER //

DROP FUNCTION IF EXISTS calculate_period_cases;

CREATE FUNCTION calculate_period_cases(annual_cases VARCHAR(50), period_divider INT)
RETURNS DECIMAL(10,2)
DETERMINISTIC
BEGIN
    DECLARE numeric_cases DECIMAL(10,2);
    DECLARE period_cases DECIMAL(10,2);

    -- Перевіряємо, чи є числове значення
    IF annual_cases = '' OR annual_cases IS NULL THEN
        RETURN NULL;
    END IF;

    -- Перетворюємо на число
    SET numeric_cases = CAST(annual_cases AS DECIMAL(10,2));

    -- Ділимо на період (12 - місяці, 4 - квартали, 2 - півріччя)
    SET period_cases = numeric_cases / period_divider;

    RETURN period_cases;
END //

DELIMITER ;
```

![5_1_custom_function.png](screenshots/5_1_custom_function.png)

### 5.2 Використання власної функції

Застосування створеної функції для розрахунку місячних середніх показників
захворюваності.

```sql
SELECT
    c.entity,
    icn.year,
    icn.number_rabies,
    calculate_period_cases(icn.number_rabies, 12) as monthly_avg_rabies
FROM infectious_cases_normalized icn
JOIN countries c ON icn.country_id = c.id
WHERE NULLIF(icn.number_rabies, '') IS NOT NULL
ORDER BY icn.year DESC, calculate_period_cases(icn.number_rabies, 12) DESC
LIMIT 10;
```

![5_2_function_usage.png](screenshots/5_2_function_usage.png)
