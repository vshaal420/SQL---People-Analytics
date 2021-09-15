# SQL---People-Analytics
# Problem Statement

**HR Analytica** has asked asked specifically to generate database views that HR Analytica team can use for 2 key dashboards, reporting solutions and ad-hoc analytics requests.

# Data Exploration

### Table Indexes 

We can query our tables and their index information by accessing the pg_indexes table as shown below:

```sql
SELECT *
FROM pg_indexes
WHERE schemaname = 'employees';
```

In the output of the above query, we can see that all tables have unique indexes.

### Individual Table Analysis

##### EMployee

```sql
SELECT *
FROM employees.employee
LIMIT 5;
```

##### Department

```sql
SELECT *
FROM employees.department;
```

##### Department Employees

```sql
SELECT *
FROM employees.department_employee
LIMIT 5;
```

##### Department Manager

```sql 
SELECT *
FROM employees.department_manager
LIMIT 5;
```

##### Salary

Investigating salary table using only one employee information.

```sql 
SELECT *
FROM employees.salary
WHERE employee_id = 10001
ORDER BY from_date DESC;
```

Lets look at the duplicate values in this tables as there can be multiple records for one employee.

```sql
WITH employee_id_cte AS (
SELECT
 employee_id,
 COUNT(*) AS row_count
FROM employees.salary
GROUP BY employee_id
)
SELECT
 row_count,
 COUNT(DISTINCT employee_id) AS employee_count
FROM employee_id_cte
GROUP BY row_count
ORDER BY row_count DESC;
```

The majority of employee records in this salary table will have more than 1 record so we should be careful when joining this table with the others.

# Analysis

### Data Cleaning

Firstly - we will need to adjust all of our relevant date fields due to the data issue identified by HR Analytica.

We will be incrementing all of the date fields except the arbitrary end date of 9999-01-01 - we will also need to cast our results back to a DATE data type as 
PostgreSQL interval addition forces the data type to a TIMESTAMP which we’d like to avoid to keep our data as similar to the original as possible.
We will be implementing our adjusted datasets as materialized views with exact original indexes as per the original tables in the employees schema.

```sql
DROP SCHEMA IF EXISTS mv_employees CASCADE;
CREATE SCHEMA mv_employees;

-- department
DROP MATERIALIZED VIEW IF EXISTS mv_employees.department;
CREATE MATERIALIZED VIEW mv_employees.department AS
SELECT * FROM employees.department;

-- department employee
DROP MATERIALIZED VIEW IF EXISTS mv_employees.department_employee;
CREATE MATERIALIZED VIEW mv_employees.department_employee AS
SELECT
  employee_id,
  department_id,
  (from_date + interval '18 years')::DATE AS from_date,
  CASE
    WHEN to_date <> '9999-01-01' THEN (to_date + interval '18 years')::DATE
    ELSE to_date
    END AS to_date
FROM employees.department_employee;

-- department manager
DROP MATERIALIZED VIEW IF EXISTS mv_employees.department_manager;
CREATE MATERIALIZED VIEW mv_employees.department_manager AS
SELECT
  employee_id,
  department_id,
  (from_date + interval '18 years')::DATE AS from_date,
  CASE
    WHEN to_date <> '9999-01-01' THEN (to_date + interval '18 years')::DATE
    ELSE to_date
    END AS to_date
FROM employees.department_manager;

-- employee
DROP MATERIALIZED VIEW IF EXISTS mv_employees.employee;
CREATE MATERIALIZED VIEW mv_employees.employee AS
SELECT
  id,
  (birth_date + interval '18 years')::DATE AS birth_date,
  first_name,
  last_name,
  gender,
  (hire_date + interval '18 years')::DATE AS hire_date
FROM employees.employee;

-- salary
DROP MATERIALIZED VIEW IF EXISTS mv_employees.salary;
CREATE MATERIALIZED VIEW mv_employees.salary AS
SELECT
  employee_id,
  amount,
  (from_date + interval '18 years')::DATE AS from_date,
  CASE
    WHEN to_date <> '9999-01-01' THEN (to_date + interval '18 years')::DATE
    ELSE to_date
    END AS to_date
FROM employees.salary;

-- title
DROP MATERIALIZED VIEW IF EXISTS mv_employees.title;
CREATE MATERIALIZED VIEW mv_employees.title AS
SELECT
  employee_id,
  title,
  (from_date + interval '18 years')::DATE AS from_date,
  CASE
    WHEN to_date <> '9999-01-01' THEN (to_date + interval '18 years')::DATE
    ELSE to_date
    END AS to_date
FROM employees.title;

-- Index Creation
CREATE UNIQUE INDEX ON mv_employees.employee USING btree (id);
CREATE UNIQUE INDEX ON mv_employees.department_employee USING btree (employee_id, department_id);
CREATE INDEX        ON mv_employees.department_employee USING btree (department_id);
CREATE UNIQUE INDEX ON mv_employees.department USING btree (id);
CREATE UNIQUE INDEX ON mv_employees.department USING btree (dept_name);
CREATE UNIQUE INDEX ON mv_employees.department_manager USING btree (employee_id, department_id);
CREATE INDEX        ON mv_employees.department_manager USING btree (department_id);
CREATE UNIQUE INDEX ON mv_employees.salary USING btree (employee_id, from_date);
CREATE UNIQUE INDEX ON mv_employees.title USING btree (employee_id, title, from_date);
```

### Current Snapshot Analysis

#### Analysis Plan
