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

![](https://u.netology.ru/backend/uploads/markdown_images/image/218808/image.png)

```sql
SELECT * 
FROM (
    SELECT 
          CONCAT_WS(' ', last_name, first_name, middle_name) fio
        , dob
        , EXTRACT(YEAR FROM age(current_date,dob)) :: int years
    FROM 
        hr.person
) age
WHERE 
    years > 65
;
```
С помощью EXTRACT вычисляем возраст сотрудника и фильтруем с помощью WHERE для выбора сотрудников, которые старше 65 лет.

   
   
## Задание 2
Вывести количество вакантных должностей. (Таблица с вакансиями может содержать недостоверные данные, решение должно быть без этой таблицы).

![](https://u.netology.ru/backend/uploads/markdown_images/image/218809/image.png)

```sql
SELECT 
    COUNT(*) - (
        SELECT COUNT(DISTINCT pos_id) FROM hr.employee
        ) count
FROM 
    hr."position"
; 
```
С помощью COUNT(*) рассчитываем общее количество должностей в таблице hr."position". В подзапросе (SELECT COUNT(DISTINCT pos_id) FROM hr.employee) подсчитывается количество занятых должностей. Разница между этими двумя числами дает количество вакантных должностей. 


## Задание 3
Вывести список проектов и количество сотрудников, задействованных на этих проектах.

![](https://u.netology.ru/backend/uploads/markdown_images/image/218810/image.png)

```sql
SELECT 
      p.name
    , p.employees_id
    , p.assigned_id
    , COALESCE(ARRAY_LENGTH(ARRAY_APPEND(employees_id, assigned_id), 1), 0) emp_count 
FROM hr.projects p
GROUP BY 
    p.name, p.employees_id, p.assigned_id
;
```
Вычисляем количество сотрудников, задейцствованных на этих проектах как сумму - количество элементов в поле `employees_id` и `assigned_id` .

## Задание 4
Получить список сотрудников у которых было повышение заработной платы на 25%.

![](https://u.netology.ru/backend/uploads/markdown_images/image/218811/image.png)

```sql
SELECT emp_id, salary, previous_salary, (salary/previous_salary-1)*100 change_percent 
FROM (
    SELECT 
         emp_id
        ,salary 
        ,LAG(salary,-1) OVER(PARTITION BY emp_id ORDER BY effective_from DESC) previous_salary
    FROM hr.employee_salary) es
WHERE
(salary/previous_salary-1)*100 = 25
;
```
Условие в WHERE фильтрует те случаи, когда последовательное увеличение заработной платы сотрудника равно 25%.
   
## Задание 5
Вывести среднее значение суммы договора на каждый год, округленное до сотых.

![](https://u.netology.ru/backend/uploads/markdown_images/image/218812/image.png)

```sql
SELECT 
      EXTRACT(YEAR FROM created_at) "year"
    , ROUND(AVG(amount), 2) avg_amount
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

![](https://u.netology.ru/backend/uploads/markdown_images/image/218813/image.png)

```sql
WITH RankedSalaries AS (
    SELECT 
          e.emp_id
        , es.salary
        , p.first_name
        , p.middle_name
        , p.last_name
        , DENSE_RANK() OVER (ORDER BY es.salary ASC) rank_lowest
        , DENSE_RANK() OVER (ORDER BY es.salary DESC) rank_highest
    FROM 
        hr.employee_salary es
    JOIN 
        hr.employee e ON es.emp_id = e.emp_id
    JOIN 
        hr.person p ON e.person_id = p.person_id
)
SELECT 
      CONCAT_WS(' ', last_name, first_name, middle_name) fio
    , salary
FROM 
    RankedSalaries
WHERE 
    rank_lowest = 1
UNION
SELECT 
      CONCAT_WS(' ', last_name, first_name, middle_name) fio
    , salary
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

![](https://u.netology.ru/backend/uploads/markdown_images/image/218814/image.png)

```sql
WITH  emp_salary_last as (
    SELECT 
        DISTINCT first_value("salary") OVER (PARTITION BY emp_id ORDER BY effective_from DESC) salary
        ,emp_id
    FROM hr.employee_salary
    )
SELECT 
    e.emp_id,
    ls.salary,
    STRING_AGG(gs.grade::text, ', ' ORDER BY gs.grade) grades_as_string
FROM 
    hr.employee e
join emp_salary_last ls on e.emp_id = ls.emp_id
LEFT JOIN hr.grade_salary gs ON ls.salary BETWEEN gs.min_salary AND gs.max_salary
GROUP BY  e.emp_id, ls.salary --, esl.salary --,p.first_name, p.middle_name, p.last_name
;
```

emp_salary_last - это временный набор данных, который использует first_value для определения последнего (актуального) оклада для каждого сотрудника.
STRING_AGG(gs.grade::text, ', ' ORDER BY gs.grade) агрегирует грейды в строку, разделяя их запятой и упорядочивая по возрастанию грейда;
     
## Задание 8
Создайте представление, которое будет содержать следующую информацию:

- ФИО сотрудника
- должность сотрудника
- структурное подразделение, где числится сотрудник
- количество полных лет сотрудника
- количество месяцев, сколько сотрудник работает в компании
- текущий оклад сотрудника
- массив со списком проектов на которых задействован сотрудник

![](https://u.netology.ru/backend/uploads/markdown_images/image/218815/image.png)

```sql
CREATE OR REPLACE VIEW hr.employee_details AS
WITH emp_projects AS (
    SELECT emp_id, ARRAY_AGG(project_id) AS project_names
    FROM (
       SELECT UNNEST(ARRAY_APPEND(p.employees_id,p.assigned_id))  emp_id, p.project_id
        FROM hr.projects p
    ) emp
    GROUP BY 
        emp_id
    ), emp_salary_last AS (
    SELECT 
        DISTINCT first_value("salary") OVER (PARTITION BY emp_id ORDER BY effective_from DESC) salary
        , emp_id
    FROM hr.employee_salary
    )
SELECT 
     CONCAT_WS(' ', p.last_name, p.first_name, p.middle_name) "фио"
    ,pos.pos_title "должность"
    ,struc.unit_title "подразделение"
    ,EXTRACT(YEAR FROM age(current_date,p.dob)) :: int "кол-во лет"
    ,EXTRACT(YEAR FROM AGE(CURRENT_DATE, e.hire_date)) * 12 + EXTRACT(MONTH FROM AGE(CURRENT_DATE, e.hire_date)) AS "кол-во месяцев"
    ,esl.salary "оклад"
    ,ep.project_names "массив с проектами"
FROM hr.employee e 
JOIN hr.person p ON p.person_id = e.person_id
JOIN hr."position" pos ON e.pos_id = pos.pos_id
JOIN hr."structure" struc ON pos.unit_id = struc.unit_id
JOIN emp_salary_last esl ON esl.emp_id = e.emp_id 
JOIN emp_projects ep ON ep.emp_id = e.emp_id
;
```

`фио` - составляется из имени, отчества (если оно есть) и фамилии сотрудника;

`кол-во лет` - вычисляеv количество полных лет сотрудника;

`кол-во месяцев` - рассчитывается как полное количество месяцев работы в компании;

`оклад` -текущий оклад определяется на основе последнего записанного значения в hr.employee_salary(используется emp_salary_last);

`массив с проектами` -Массив названий проектов(используется emp_projects), в которых задействован сотрудник;
