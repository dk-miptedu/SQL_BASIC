# Итоговое домашнее задание
**Дисциплина:** Базы данных (SQL)

**Тема:** Итоговое домашнее задание

**Форма проверки:** Задание проверяется преподавателем

**Имя преподавателя:** Николай Хащанов

**Цель задания:** Проверить и закрепить полученные знания по дисциплине

**Инструменты для выполнения ДЗ:** PostgreSQL, DBeaver,  [учебная база данных (из инструкции)](https://github.com/netology-ds-team/sql-materials/tree/main/sql-mipt)

**Правила приёма работы:** Решение должно быть в формате одного sql-файла с отформатированными запросами.

## Аннотация Решения

[Файл решения в формате одного sql-файла с отформатированными запросами (скачать)](kazanskii.da.sql)

## Задание 1
Вывести список сотрудников старше 65 лет.

```sql
SELECT 
    last_name || ' ' || first_name || ' ' || COALESCE(middle_name, '') fio,
    dob, 
    EXTRACT(YEAR FROM age(dob)) years
FROM 
    hr.person
WHERE 
    dob <= CURRENT_DATE - INTERVAL '65 years'
;
```
С помощью EXTRACT вычисляем возраст сотрудника и фильтруем с помощью WHERE для выбора сотрудников, которым уже исполнилось 65 лет. 

> middle_name может быть NULL, поэтому использовал функция COALESCE.

   
   
## Задание 2
Вывести количество вакантных должностей. (Таблица с вакансиями может содержать недостоверные данные, решение должно быть без этой таблицы).

```sql
SELECT 
    COUNT(*) - (SELECT COUNT(DISTINCT pos_id) FROM hr.employee) AS count
FROM 
    hr."position"
;
```
С помощью COUNT(*) рассчитываем общее количество должностей в таблице hr."position". В подзапросе (SELECT COUNT(DISTINCT pos_id) FROM hr.employee) подсчитывается количество занятых должностей. Разница между этими двумя числами дает количество вакантных должностей. 
   
## Задание 3
Вывести список проектов и количество сотрудников, задействованных на этих проектах.

```sql
SELECT 
    p.name, 
    p.employees_id, 
    p.assigned_id,
    (coalesce(array_length(p.employees_id, 1), 0)+ COUNT(DISTINCT p.assigned_id)) emp_count 
FROM hr.projects p
GROUP BY 
    p.name, p.employees_id, p.assigned_id
;
```
Вычисляем количество сотрудников, задейцствованных на этих проектах как сумму - количество элементов в поле `employees_id` и `assigned_id` .

## Задание 4
Получить список сотрудников у которых было повышение заработной платы на 25%

```sql
SELECT 
    e.emp_id,
    es2.salary salary,
    es1.salary previous_salary,
    (es2.salary/es1.salary - 1)*100 change_percent
FROM 
    hr.employee e
JOIN 
    hr.employee_salary es1 ON e.emp_id = es1.emp_id
JOIN 
    hr.employee_salary es2 ON e.emp_id = es2.emp_id
WHERE 
    es2.effective_from > es1.effective_from
AND 
    es2.salary = es1.salary * 1.25
ORDER BY 
    e.emp_id
;
```

Происходит соединение таблиц hr.employee и hr.employee_salary дважды: один раз для получения предыдущей зарплаты (es1) и один раз для текущей зарплаты (es2).
Условие в WHERE фильтрует те случаи, где effective_from второй зарплаты больше первой на 25%.
   
## Задание 5
Вывести среднее значение суммы договора на каждый год, округленное до сотых.

```sql
SELECT 
    EXTRACT(YEAR FROM created_at) "year", 
    ROUND(AVG(amount), 2) avg_amount
FROM 
    hr.projects
GROUP BY 
    EXTRACT(YEAR FROM created_at)
ORDER BY 
    year
;   
```
Чтобы получить год из даты создания проекта используется функция EXTRACT(YEAR FROM created_at).
Вычисляем среднее значение суммы договоров с помощью AVG(amount) и округляем значение до сотых.
Группируем данные по годам с помощью GROUP BY EXTRACT(YEAR FROM created_at) и упорядочиваем результат по году (ORDER BY year).

## Задание 6
Одним запросом вывести ФИО сотрудников с самым низким и самым высоким окладами за все время.

```sql
WITH RankedSalaries AS (
    SELECT 
        e.emp_id,
        es.salary,
        p.first_name,
        p.middle_name,
        p.last_name,
        DENSE_RANK() OVER (ORDER BY es.salary ASC) rank_lowest,
        DENSE_RANK() OVER (ORDER BY es.salary DESC) rank_highest
    FROM 
        hr.employee_salary es
    JOIN 
        hr.employee e ON es.emp_id = e.emp_id
    JOIN 
        hr.person p ON e.person_id = p.person_id
)
SELECT 
    last_name || ' ' || first_name || ' ' || COALESCE(middle_name, '') fio,
    salary
FROM 
    RankedSalaries
WHERE 
    rank_lowest = 1
UNION
SELECT 
    last_name || ' ' || first_name || ' ' || COALESCE(middle_name, '') fio, 
    salary
FROM 
    RankedSalaries
WHERE 
    rank_highest = 1
;
```

RankedSalaries временный набор данных с зарплатами и информацией о сотрудниках, включая ранги для самых низких и самых высоких зарплат.
DENSE_RANK() используется для ранжирования зарплат по возрастанию (rank_lowest) и убыванию (rank_highest).
Из RankedSalaries выбираются записи с наименьшим (rank_lowest = 1) и наибольшим (rank_highest = 1) рангами.
Далее UNION объединяет эти два набора результатов.

## Задание 7
Вывести текущий оклад сотрудников и в формате строки вывести зарплатные грейды, в которые попадает текущий оклад.

```sql
WITH LatestSalaries AS (
    SELECT 
        es.emp_id,
        es.salary,
        ROW_NUMBER() OVER (PARTITION BY es.emp_id ORDER BY es.effective_from DESC) AS rn
    FROM 
        hr.employee_salary es
)
SELECT 
    e.emp_id,
    ls.salary,
    STRING_AGG(gs.grade::text, ', ' ORDER BY gs.grade) grades_as_string
FROM 
    hr.employee e
JOIN 
    LatestSalaries ls ON e.emp_id = ls.emp_id AND ls.rn = 1
LEFT JOIN 
    hr.grade_salary gs ON ls.salary BETWEEN gs.min_salary AND gs.max_salary
-- WHERE e.emp_id in (1665, 1547, 1176, 1202, 802) --Test by screenshot
GROUP BY 
    e.emp_id, ls.salary
;
```

LatestSalaries - это временный набор данных, который использует ROW_NUMBER() для определения последнего (самого актуального) оклада для каждого сотрудника.
STRING_AGG(gs.grade::text, ', ' ORDER BY gs.grade) агрегирует грейды в строку, разделяя их запятой и упорядочивая по возрастанию грейда;
     
## Задание 8
Создайте представление, которое будет содержать следующую информацию:
--
--    ФИО сотрудника
--    должность сотрудника
--    структурное подразделение, где числится сотрудник
--    количество полных лет сотрудника
--    количество месяцев, сколько сотрудник работает в компании
--    текущий оклад сотрудника
--    массив со списком проектов на которых задействован сотрудник

```sql
CREATE VIEW hr.employee_details as --задокументировать строку для проверки SELECT
SELECT 
    p.last_name || ' ' || p.first_name || ' ' || COALESCE(p.middle_name, '') "фио",
    pos.pos_title "должность",
    s.unit_title "подразделение",
    EXTRACT(YEAR FROM AGE(CURRENT_DATE, p.dob)) "кол-во лет",
    EXTRACT(YEAR FROM AGE(CURRENT_DATE, e.hire_date)) * 12 + EXTRACT(MONTH FROM AGE(CURRENT_DATE, e.hire_date)) "кол-во месяцев",
    es.salary "salary",
    (SELECT ARRAY_AGG(pr.project_id) FROM hr.projects pr WHERE pr.employees_id @> ARRAY[e.emp_id]) "массив с проектами"
FROM 
    hr.person p
JOIN 
    hr.employee e ON p.person_id = e.person_id
JOIN 
    hr.position pos ON e.pos_id = pos.pos_id
JOIN 
    hr."structure" s ON pos.unit_id = s.unit_id
JOIN 
    (SELECT emp_id, salary FROM hr.employee_salary es1 WHERE es1.effective_from = (SELECT MAX(es2.effective_from) FROM hr.employee_salary es2 WHERE es2.emp_id = es1.emp_id)) es ON e.emp_id = es.emp_id
-- WHERE p.last_name in ('Суханов', 'Баранов', 'Вишневская', 'Алексеев', '')  order by p.last_name
;
```

`фио` - составляется из имени, отчества (если оно есть) и фамилии сотрудника;
`кол-во лет` - вычисляеv количество полных лет сотрудника;
`кол-во месяцев` - рассчитывается как общее количество месяцев работы в компании;
`salary` -текущий оклад определяется на основе последнего записанного значения в hr.employee_salary;
`массив с проектами` -Массив названий проектов(используется ARRAY_AGG(pr.name)), в которых задействован сотрудник;
