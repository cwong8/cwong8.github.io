---
layout: page
title: Employee promotions and past employees
permalink: /projects/employees/sql_python/
---


## MySQL Employees Sample Database
Using the MySQL Employees sample database to practice and learn SQL/MySQL before using Python and R to analyze queried data. Information about the database can be found at [https://dev.mysql.com/doc/employee/en/](https://dev.mysql.com/doc/employee/en/). The database can be downloaded at [https://github.com/datacharmer/test_db](https://github.com/datacharmer/test_db).

### Tools used
- Package pandas for dataframes and functions such as read_sql_query
- Package numpy for general math functions
- Package MySQLdb to connect to MySQL
- Package scipy.stats for statistical testing
- Packages matplotlib and seaborn for plotting

Creating a connection to MySQL, replace "yourusername" and "yourpassword" with the username/password to your MySQL.


```python
import os
import pandas as pd
import numpy as np
import scipy.stats as stats
import matplotlib.pyplot as plt
import seaborn as sns

# Server connection to MySQL:
import MySQLdb
conn = MySQLdb.connect(host= "localhost",
                  user="yourusername",
                  passwd="yourpassword",
                  db="employees")

x = conn.cursor()
```

### Employee promotions
In this section, I answer some questions regarding promotions within this fictional company.
1. How many promotions were given? To how many employees?
2. Has any employee been promoted more than once? What is the maximum times an employee has been promoted?
3. How many employees were never promoted or left before being promoted?
4. What is the average time before an employee is promoted? Is there noticeable discrimination based on gender or age?

In MySQL I created a view named Promotions that I will reference but views cannot be created in Python:
```
CREATE OR REPLACE VIEW Promotions AS
SELECT employees.emp_no, birth_date, first_name, last_name, gender, a.from_date AS hire_date, a.title AS hire_title, a.to_date AS promotion_date, b.title AS promoted_to, b.to_date AS end_date
FROM employees JOIN titles a ON employees.emp_no = a.emp_no JOIN titles b ON a.emp_no = b.emp_no
WHERE a.to_date = b.from_date AND a.title != b.title;
```

##### 1. How many promotions were given? To how many employees?


```python
### Finding all promotions
promotions = pd.read_sql_query("""
SELECT employees.emp_no, birth_date, first_name, last_name, gender, a.from_date AS hire_date, a.title AS hire_title, a.to_date AS promotion_date, b.title AS promoted_to, b.to_date AS end_date
FROM employees JOIN titles a ON employees.emp_no = a.emp_no JOIN titles b ON a.emp_no = b.emp_no
WHERE a.to_date = b.from_date AND a.title != b.title;
""", conn)
promotions.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>emp_no</th>
      <th>birth_date</th>
      <th>first_name</th>
      <th>last_name</th>
      <th>gender</th>
      <th>hire_date</th>
      <th>hire_title</th>
      <th>promotion_date</th>
      <th>promoted_to</th>
      <th>end_date</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>10004</td>
      <td>1954-05-01</td>
      <td>Chirstian</td>
      <td>Koblick</td>
      <td>M</td>
      <td>1986-12-01</td>
      <td>Engineer</td>
      <td>1995-12-01</td>
      <td>Senior Engineer</td>
      <td>9999-01-01</td>
    </tr>
    <tr>
      <th>1</th>
      <td>10005</td>
      <td>1955-01-21</td>
      <td>Kyoichi</td>
      <td>Maliniak</td>
      <td>M</td>
      <td>1989-09-12</td>
      <td>Staff</td>
      <td>1996-09-12</td>
      <td>Senior Staff</td>
      <td>9999-01-01</td>
    </tr>
    <tr>
      <th>2</th>
      <td>10007</td>
      <td>1957-05-23</td>
      <td>Tzvetan</td>
      <td>Zielinski</td>
      <td>F</td>
      <td>1989-02-10</td>
      <td>Staff</td>
      <td>1996-02-11</td>
      <td>Senior Staff</td>
      <td>9999-01-01</td>
    </tr>
    <tr>
      <th>3</th>
      <td>10009</td>
      <td>1952-04-19</td>
      <td>Sumant</td>
      <td>Peac</td>
      <td>F</td>
      <td>1985-02-18</td>
      <td>Assistant Engineer</td>
      <td>1990-02-18</td>
      <td>Engineer</td>
      <td>1995-02-18</td>
    </tr>
    <tr>
      <th>4</th>
      <td>10009</td>
      <td>1952-04-19</td>
      <td>Sumant</td>
      <td>Peac</td>
      <td>F</td>
      <td>1990-02-18</td>
      <td>Engineer</td>
      <td>1995-02-18</td>
      <td>Senior Engineer</td>
      <td>9999-01-01</td>
    </tr>
  </tbody>
</table>
</div>



Notice how some employees have been promoted more than once (emp_no = 10009)


```python
### Number of promotions
pd.read_sql_query("""
SELECT COUNT(*)
FROM Promotions;
""", conn)
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>COUNT(*)</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>143284</td>
    </tr>
  </tbody>
</table>
</div>




```python
### Number of employees who received promotions
pd.read_sql_query("""
SELECT COUNT(DISTINCT(emp_no))
FROM Promotions;
""", conn)
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>COUNT(DISTINCT(emp_no))</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>140270</td>
    </tr>
  </tbody>
</table>
</div>



We find that 143,294 promotions were given by this company to 140,270 employees. There are more promotions given than employees which leads into the next question.

##### 2. Has any employee been promoted more than once? What is the maximum times an employee has been promoted?


```python
### Finding employees who have been promoted more than once
df = pd.read_sql_query("""
SELECT emp_no, COUNT(emp_no) AS times_promoted
FROM Promotions
GROUP BY emp_no
HAVING COUNT(*) > 1
""", conn)
df.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>emp_no</th>
      <th>times_promoted</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>10009</td>
      <td>2</td>
    </tr>
    <tr>
      <th>1</th>
      <td>10066</td>
      <td>2</td>
    </tr>
    <tr>
      <th>2</th>
      <td>10258</td>
      <td>2</td>
    </tr>
    <tr>
      <th>3</th>
      <td>10451</td>
      <td>2</td>
    </tr>
    <tr>
      <th>4</th>
      <td>10571</td>
      <td>2</td>
    </tr>
  </tbody>
</table>
</div>




```python
### Finding how many employees were promoted more than once
pd.read_sql_query("""
SELECT COUNT(*)
FROM
(SELECT emp_no, COUNT(emp_no) AS times_promoted
FROM Promotions
GROUP BY emp_no
HAVING COUNT(*) > 1) AS a;
""", conn)
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>COUNT(*)</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>3014</td>
    </tr>
  </tbody>
</table>
</div>




```python
### Finding the maximum number of promotions by a single employee
pd.read_sql_query("""
SELECT MAX(times_promoted)
FROM 
(SELECT emp_no, COUNT(emp_no) AS times_promoted
FROM Promotions
GROUP BY emp_no
HAVING COUNT(*) > 1) AS a;
""", conn)
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>MAX(times_promoted)</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>2</td>
    </tr>
  </tbody>
</table>
</div>



We find that 3,014 employees have been promoted twice, and that no employees have ever been promoted three times.

##### 3. How many employees were never promoted or left before being promoted?


```python
### Finding all employees who have never been promoted or left before a promotion
df = pd.read_sql_query("""
SELECT employees.emp_no, birth_date, first_name, last_name, gender, hire_date, title, from_date, to_date
FROM employees JOIN titles ON employees.emp_no = titles.emp_no
GROUP BY employees.emp_no
HAVING COUNT(employees.emp_no) = 1;
""", conn)
df.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>emp_no</th>
      <th>birth_date</th>
      <th>first_name</th>
      <th>last_name</th>
      <th>gender</th>
      <th>hire_date</th>
      <th>title</th>
      <th>from_date</th>
      <th>to_date</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>10001</td>
      <td>1953-09-02</td>
      <td>Georgi</td>
      <td>Facello</td>
      <td>M</td>
      <td>1986-06-26</td>
      <td>Senior Engineer</td>
      <td>1986-06-26</td>
      <td>9999-01-01</td>
    </tr>
    <tr>
      <th>1</th>
      <td>10002</td>
      <td>1964-06-02</td>
      <td>Bezalel</td>
      <td>Simmel</td>
      <td>F</td>
      <td>1985-11-21</td>
      <td>Staff</td>
      <td>1996-08-03</td>
      <td>9999-01-01</td>
    </tr>
    <tr>
      <th>2</th>
      <td>10003</td>
      <td>1959-12-03</td>
      <td>Parto</td>
      <td>Bamford</td>
      <td>M</td>
      <td>1986-08-28</td>
      <td>Senior Engineer</td>
      <td>1995-12-03</td>
      <td>9999-01-01</td>
    </tr>
    <tr>
      <th>3</th>
      <td>10006</td>
      <td>1953-04-20</td>
      <td>Anneke</td>
      <td>Preusig</td>
      <td>F</td>
      <td>1989-06-02</td>
      <td>Senior Engineer</td>
      <td>1990-08-05</td>
      <td>9999-01-01</td>
    </tr>
    <tr>
      <th>4</th>
      <td>10008</td>
      <td>1958-02-19</td>
      <td>Saniya</td>
      <td>Kalloufi</td>
      <td>M</td>
      <td>1994-09-15</td>
      <td>Assistant Engineer</td>
      <td>1998-03-11</td>
      <td>2000-07-31</td>
    </tr>
  </tbody>
</table>
</div>




```python
### Finding how many employees that have never been promoted or left before a promotion
pd.read_sql_query("""
SELECT COUNT(*)
FROM
(SELECT employees.emp_no, birth_date, first_name, last_name, gender, hire_date, title, from_date, to_date
FROM employees JOIN titles ON employees.emp_no = titles.emp_no
GROUP BY employees.emp_no
HAVING COUNT(employees.emp_no) = 1) AS a;
""", conn)
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>COUNT(*)</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>159754</td>
    </tr>
  </tbody>
</table>
</div>



We find that 159,754 employees have never been promoted or left before a promotion.

##### 4. What is the average time before an employee is promoted? Is there noticeable discrimination based on gender or age?


```python
### Finding the time before an employee is promoted
df = pd.read_sql_query("""
SELECT emp_no, gender, TIMESTAMPDIFF(DAY, hire_date, promotion_date) AS days_to_promotion
FROM promotions;
""", conn)
df.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>emp_no</th>
      <th>gender</th>
      <th>days_to_promotion</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>10004</td>
      <td>M</td>
      <td>3287</td>
    </tr>
    <tr>
      <th>1</th>
      <td>10005</td>
      <td>M</td>
      <td>2557</td>
    </tr>
    <tr>
      <th>2</th>
      <td>10007</td>
      <td>F</td>
      <td>2557</td>
    </tr>
    <tr>
      <th>3</th>
      <td>10009</td>
      <td>F</td>
      <td>1826</td>
    </tr>
    <tr>
      <th>4</th>
      <td>10009</td>
      <td>F</td>
      <td>1826</td>
    </tr>
  </tbody>
</table>
</div>




```python
### Grouping on gender and taking the average
pd.read_sql_query("""
SELECT gender, AVG(days_to_promotion)
FROM
(SELECT emp_no, gender, TIMESTAMPDIFF(DAY, hire_date, promotion_date) AS days_to_promotion
FROM promotions) AS a
GROUP BY gender;
""", conn)
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>gender</th>
      <th>AVG(days_to_promotion)</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>M</td>
      <td>2473.6378</td>
    </tr>
    <tr>
      <th>1</th>
      <td>F</td>
      <td>2469.7197</td>
    </tr>
  </tbody>
</table>
</div>



Let us visualize the distributions of days until promotion for males and females.


```python
# Group by gender
df1 = df.groupby(['gender'])

# Plot density plots
sns.kdeplot(df1.get_group('F')['days_to_promotion'])
sns.kdeplot(df1.get_group('M')['days_to_promotion'])
```




    <matplotlib.axes._subplots.AxesSubplot at 0x22ea5ff19b0>




![png](output_25_1.png)


We see they have roughly the same distribution and similar peaks, leading me to conclude that there is no discrimination between genders. Something to note is that some employees have the same number of days before a promotion (e.g. emp_no = 10005 and 10007) and the number of days is very close to a multiple of days in a year (365 days approximately). This leads me to believe that employees are promoted on or near their hire dates, most likely at their yearly reviews.


```python
### Finding the age of employees when they were promoted
df = pd.read_sql_query("""
SELECT emp_no, gender, birth_date, promotion_date, TIMESTAMPDIFF(YEAR, birth_date, promotion_date) AS age_at_promotion
FROM promotions;
""", conn)
df.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>emp_no</th>
      <th>gender</th>
      <th>birth_date</th>
      <th>promotion_date</th>
      <th>age_at_promotion</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>10004</td>
      <td>M</td>
      <td>1954-05-01</td>
      <td>1995-12-01</td>
      <td>41</td>
    </tr>
    <tr>
      <th>1</th>
      <td>10005</td>
      <td>M</td>
      <td>1955-01-21</td>
      <td>1996-09-12</td>
      <td>41</td>
    </tr>
    <tr>
      <th>2</th>
      <td>10007</td>
      <td>F</td>
      <td>1957-05-23</td>
      <td>1996-02-11</td>
      <td>38</td>
    </tr>
    <tr>
      <th>3</th>
      <td>10009</td>
      <td>F</td>
      <td>1952-04-19</td>
      <td>1990-02-18</td>
      <td>37</td>
    </tr>
    <tr>
      <th>4</th>
      <td>10009</td>
      <td>F</td>
      <td>1952-04-19</td>
      <td>1995-02-18</td>
      <td>42</td>
    </tr>
  </tbody>
</table>
</div>




```python
### Grouping on gender and taking the average
pd.read_sql_query("""
SELECT gender, AVG(age_at_promotion)
FROM 
(SELECT emp_no, gender, birth_date, promotion_date, TIMESTAMPDIFF(YEAR, birth_date, promotion_date) AS age_at_promotion
FROM promotions) AS a
GROUP BY gender;
""", conn)
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>gender</th>
      <th>AVG(age_at_promotion)</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>M</td>
      <td>38.1015</td>
    </tr>
    <tr>
      <th>1</th>
      <td>F</td>
      <td>38.0952</td>
    </tr>
  </tbody>
</table>
</div>



Let us visualize the distributions of ages of promoted employees for males and females.


```python
# Group by gender
df2 = df.groupby(['gender'])

# Plot density plots
sns.kdeplot(df2.get_group('F')['age_at_promotion'])
sns.kdeplot(df2.get_group('M')['age_at_promotion'])
```




    <matplotlib.axes._subplots.AxesSubplot at 0x1a2a4e077b8>




![png](output_30_1.png)


We see a much nicer and parametric distribution compared to the distribution of days until promotions. This bell curve distribution shape implies a Gaussian/normal distribution which we can test for. Assumption for a normality test such as the Shapiro-Wilk test are that observations are independent and identically distributed (iid) which is a fair assumption given observations are different employees for the most part. Some employees show up multiple times because they were promoted more than once (e.g. emp_no = 10009). We can obtain the first promotion for employees with multiple promotions by grouping on their employee id.


```python
# Group on emp_no to obtain the age at first promotion for all employees
df = pd.read_sql_query("""
SELECT emp_no, gender, birth_date, promotion_date, TIMESTAMPDIFF(YEAR, birth_date, promotion_date) AS age_at_promotion
FROM promotions
GROUP BY emp_no;
""", conn)
df.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>emp_no</th>
      <th>gender</th>
      <th>birth_date</th>
      <th>promotion_date</th>
      <th>age_at_promotion</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>10004</td>
      <td>M</td>
      <td>1954-05-01</td>
      <td>1995-12-01</td>
      <td>41</td>
    </tr>
    <tr>
      <th>1</th>
      <td>10005</td>
      <td>M</td>
      <td>1955-01-21</td>
      <td>1996-09-12</td>
      <td>41</td>
    </tr>
    <tr>
      <th>2</th>
      <td>10007</td>
      <td>F</td>
      <td>1957-05-23</td>
      <td>1996-02-11</td>
      <td>38</td>
    </tr>
    <tr>
      <th>3</th>
      <td>10009</td>
      <td>F</td>
      <td>1952-04-19</td>
      <td>1990-02-18</td>
      <td>37</td>
    </tr>
    <tr>
      <th>4</th>
      <td>10012</td>
      <td>M</td>
      <td>1960-10-04</td>
      <td>2000-12-18</td>
      <td>40</td>
    </tr>
  </tbody>
</table>
</div>




```python
# Testing for normality
stat, pF = stats.normaltest(df.groupby('gender').get_group('F')['age_at_promotion'])
stat, pM = stats.normaltest(df.groupby('gender').get_group('M')['age_at_promotion'])
{'gender': ['M', 'F'], 'p-values': [pM, pF]}
```




    {'gender': ['M', 'F'], 'p-values': [0.0, 0.0]}




```python
# Quantile-quantile (qq) plot of the residuals
res = stats.probplot(df2.get_group('F')['age_at_promotion'], plot = plt)
```


![png](output_34_0.png)


We see that our p-values for both genders is 0 so we reject our null hypothesis that the distribution is normal and conclude that our distribution is not Gaussian/normal. We can see from the above qq-plot that the tails of our distribution are non-existent. This means the company is not promoting (or hiring) anyone who is under a minimum age or above a maximum age (i.e. they do not promote (or hire) teenagers or people close to retirement age).

### Past employees
Let's take a look at the past employees at this company.
1. How many past employees does this company have?
2. Find the ages of new hires. What is the average hire age? Plot the distribution.
3. Find the ages of employees when they left the company. What is the average age of employees leaving the company? Plot the distribution.
4. Combine the previous two questions to find the time spent at the company by past employees. Any interesting observations?

##### 1. How many past employees does this company have?


```python
### Finding employees who have left the company. Current employees have to_date set to '9999-01-01'
df = pd.read_sql_query("""
SELECT employees.*, to_date
FROM employees JOIN titles ON employees.emp_no = titles.emp_no
GROUP BY employees.emp_no
HAVING MAX(to_date) < DATE '9999-01-01';
""", conn)
df.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>emp_no</th>
      <th>birth_date</th>
      <th>first_name</th>
      <th>last_name</th>
      <th>gender</th>
      <th>hire_date</th>
      <th>to_date</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>10008</td>
      <td>1958-02-19</td>
      <td>Saniya</td>
      <td>Kalloufi</td>
      <td>M</td>
      <td>1994-09-15</td>
      <td>2000-07-31</td>
    </tr>
    <tr>
      <th>1</th>
      <td>10011</td>
      <td>1953-11-07</td>
      <td>Mary</td>
      <td>Sluis</td>
      <td>F</td>
      <td>1990-01-22</td>
      <td>1996-11-09</td>
    </tr>
    <tr>
      <th>2</th>
      <td>10015</td>
      <td>1959-08-19</td>
      <td>Guoxiang</td>
      <td>Nooteboom</td>
      <td>M</td>
      <td>1987-07-02</td>
      <td>1993-08-22</td>
    </tr>
    <tr>
      <th>3</th>
      <td>10021</td>
      <td>1960-02-20</td>
      <td>Ramzi</td>
      <td>Erde</td>
      <td>M</td>
      <td>1988-02-10</td>
      <td>2002-07-15</td>
    </tr>
    <tr>
      <th>4</th>
      <td>10025</td>
      <td>1958-10-31</td>
      <td>Prasadram</td>
      <td>Heyers</td>
      <td>M</td>
      <td>1987-08-17</td>
      <td>1997-10-15</td>
    </tr>
  </tbody>
</table>
</div>




```python
### Finding how many past employees there are
pd.read_sql_query("""
SELECT COUNT(DISTINCT(emp_no))
FROM
(SELECT employees.*, to_date
FROM employees JOIN titles ON employees.emp_no = titles.emp_no
GROUP BY employees.emp_no
HAVING MAX(to_date) < DATE '9999-01-01') AS a;
""", conn)
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>COUNT(DISTINCT(emp_no))</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>59900</td>
    </tr>
  </tbody>
</table>
</div>



We find that 59,900 employees have left this company.

##### 2. Find the ages of new hires. What is the average hire age? Plot the distribution.


```python
### Finding the hire age of all employees
new_hire = pd.read_sql_query("""
SELECT employees.emp_no, birth_date, first_name, last_name, gender, hire_date, TIMESTAMPDIFF(YEAR, birth_date, hire_date) AS hire_age
FROM employees JOIN titles ON employees.emp_no = titles.emp_no
GROUP BY employees.emp_no;
""", conn)
df.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>emp_no</th>
      <th>birth_date</th>
      <th>first_name</th>
      <th>last_name</th>
      <th>gender</th>
      <th>hire_date</th>
      <th>to_date</th>
      <th>leaving_age</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>10008</td>
      <td>1958-02-19</td>
      <td>Saniya</td>
      <td>Kalloufi</td>
      <td>M</td>
      <td>1994-09-15</td>
      <td>2000-07-31</td>
      <td>42</td>
    </tr>
    <tr>
      <th>1</th>
      <td>10011</td>
      <td>1953-11-07</td>
      <td>Mary</td>
      <td>Sluis</td>
      <td>F</td>
      <td>1990-01-22</td>
      <td>1996-11-09</td>
      <td>43</td>
    </tr>
    <tr>
      <th>2</th>
      <td>10015</td>
      <td>1959-08-19</td>
      <td>Guoxiang</td>
      <td>Nooteboom</td>
      <td>M</td>
      <td>1987-07-02</td>
      <td>1993-08-22</td>
      <td>34</td>
    </tr>
    <tr>
      <th>3</th>
      <td>10021</td>
      <td>1960-02-20</td>
      <td>Ramzi</td>
      <td>Erde</td>
      <td>M</td>
      <td>1988-02-10</td>
      <td>2002-07-15</td>
      <td>42</td>
    </tr>
    <tr>
      <th>4</th>
      <td>10025</td>
      <td>1958-10-31</td>
      <td>Prasadram</td>
      <td>Heyers</td>
      <td>M</td>
      <td>1987-08-17</td>
      <td>1997-10-15</td>
      <td>38</td>
    </tr>
  </tbody>
</table>
</div>




```python
### Finding the average hire age
pd.read_sql_query("""
SELECT AVG(hire_age)
FROM
(SELECT employees.emp_no, birth_date, first_name, last_name, gender, hire_date, TIMESTAMPDIFF(YEAR, birth_date, hire_date) AS hire_age
FROM employees JOIN titles ON employees.emp_no = titles.emp_no
GROUP BY employees.emp_no) AS a;
""", conn)
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>AVG(hire_age)</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>30.9972</td>
    </tr>
  </tbody>
</table>
</div>



We find that employees are hired at an average age of approximately 31 years old.


```python
### Plot the distribution of ages for new hires
a = plt.hist(new_hire['hire_age'], bins = (max(new_hire['hire_age']) - min(new_hire['hire_age'])))
```


![png](output_45_0.png)



```python
new_hire
```




    (array([ 1461.,  4410.,  7059.,  9319., 11666., 14004., 15477., 17389.,
            18849., 19995., 20882., 21984., 22357., 21700., 18577., 16190.,
            13483., 11326.,  9188.,  7339.,  5765.,  4323.,  3067.,  2037.,
             1266.,   631.,   280.]),
     array([20., 21., 22., 23., 24., 25., 26., 27., 28., 29., 30., 31., 32.,
            33., 34., 35., 36., 37., 38., 39., 40., 41., 42., 43., 44., 45.,
            46., 47.]),
     <a list of 27 Patch objects>)



##### 3. Find the ages of employees when they left the company. What is the average age of employees leaving the company? Plot the distribution.


```python
### Finding the age of employees when they left the company
past_employees = pd.read_sql_query("""
SELECT employees.*, to_date, TIMESTAMPDIFF(YEAR, birth_date, to_date) AS leaving_age
FROM employees JOIN titles ON employees.emp_no = titles.emp_no
GROUP BY employees.emp_no
HAVING MAX(to_date) < '9999-01-01';
""", conn)
df.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>emp_no</th>
      <th>birth_date</th>
      <th>first_name</th>
      <th>last_name</th>
      <th>gender</th>
      <th>hire_date</th>
      <th>to_date</th>
      <th>leaving_age</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>10008</td>
      <td>1958-02-19</td>
      <td>Saniya</td>
      <td>Kalloufi</td>
      <td>M</td>
      <td>1994-09-15</td>
      <td>2000-07-31</td>
      <td>42</td>
    </tr>
    <tr>
      <th>1</th>
      <td>10011</td>
      <td>1953-11-07</td>
      <td>Mary</td>
      <td>Sluis</td>
      <td>F</td>
      <td>1990-01-22</td>
      <td>1996-11-09</td>
      <td>43</td>
    </tr>
    <tr>
      <th>2</th>
      <td>10015</td>
      <td>1959-08-19</td>
      <td>Guoxiang</td>
      <td>Nooteboom</td>
      <td>M</td>
      <td>1987-07-02</td>
      <td>1993-08-22</td>
      <td>34</td>
    </tr>
    <tr>
      <th>3</th>
      <td>10021</td>
      <td>1960-02-20</td>
      <td>Ramzi</td>
      <td>Erde</td>
      <td>M</td>
      <td>1988-02-10</td>
      <td>2002-07-15</td>
      <td>42</td>
    </tr>
    <tr>
      <th>4</th>
      <td>10025</td>
      <td>1958-10-31</td>
      <td>Prasadram</td>
      <td>Heyers</td>
      <td>M</td>
      <td>1987-08-17</td>
      <td>1997-10-15</td>
      <td>38</td>
    </tr>
  </tbody>
</table>
</div>




```python
### Finding the average age of employees who left the company
pd.read_sql_query("""
SELECT AVG(leaving_age)
FROM
(SELECT employees.*, to_date, TIMESTAMPDIFF(YEAR, birth_date, to_date) AS leaving_age
FROM employees JOIN titles ON employees.emp_no = titles.emp_no
GROUP BY employees.emp_no
HAVING MAX(to_date) < '9999-01-01') AS a;
""", conn)
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>AVG(leaving_age)</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>37.9825</td>
    </tr>
  </tbody>
</table>
</div>



We find that on average employees leave when they are approximately 38 years old.


```python
### Plot the distribution of ages for employees who have left
a = plt.hist(past_employees['leaving_age'], bins = (max(past_employees['leaving_age']) - min(past_employees['leaving_age'])))
```


![png](output_51_0.png)


Let's take a look at the distribution of ages for new and past employees.


```python
### Both distributions
a = plt.hist(new_hire['hire_age'], bins = (max(new_hire['hire_age']) - min(new_hire['hire_age'])))
a = plt.hist(past_employees['leaving_age'], bins = (max(past_employees['leaving_age']) - min(past_employees['leaving_age'])))
```


![png](output_53_0.png)


We see that there are a lot more new hires compared to employees leaving the company. This could indicate that the company has grown significantly.

##### 4. Combine the previous two questions to find the time spent at the company by past employees. Any interesting observations?


```python
### Finding past employees of this company and their ages at hire and their last day
df = pd.read_sql_query("""
SELECT a.*, b.to_date, b.leaving_age, CONCAT(TIMESTAMPDIFF(YEAR, a.hire_date, b.to_date), ' years, ', MOD(TIMESTAMPDIFF(MONTH, a.hire_date, b.to_date), 12), ' months') AS time_at_company, TIMESTAMPDIFF(DAY, a.hire_date, b.to_date) AS company_days
FROM
(SELECT employees.emp_no, birth_date, first_name, last_name, gender, hire_date, TIMESTAMPDIFF(YEAR, birth_date, hire_date) AS hire_age
FROM employees JOIN titles ON employees.emp_no = titles.emp_no
GROUP BY employees.emp_no) AS a
INNER JOIN
(SELECT employees.*, from_date, to_date, TIMESTAMPDIFF(YEAR, birth_date, to_date) AS leaving_age
FROM employees JOIN titles ON employees.emp_no = titles.emp_no
GROUP BY employees.emp_no
HAVING MAX(to_date) < '9999-01-01') AS b ON a.emp_no = b.emp_no;
""", conn)
df.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>emp_no</th>
      <th>birth_date</th>
      <th>first_name</th>
      <th>last_name</th>
      <th>gender</th>
      <th>hire_date</th>
      <th>hire_age</th>
      <th>to_date</th>
      <th>leaving_age</th>
      <th>time_at_company</th>
      <th>company_days</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>10008</td>
      <td>1958-02-19</td>
      <td>Saniya</td>
      <td>Kalloufi</td>
      <td>M</td>
      <td>1994-09-15</td>
      <td>36</td>
      <td>2000-07-31</td>
      <td>42</td>
      <td>5 years, 10 months</td>
      <td>2146</td>
    </tr>
    <tr>
      <th>1</th>
      <td>10011</td>
      <td>1953-11-07</td>
      <td>Mary</td>
      <td>Sluis</td>
      <td>F</td>
      <td>1990-01-22</td>
      <td>36</td>
      <td>1996-11-09</td>
      <td>43</td>
      <td>6 years, 9 months</td>
      <td>2483</td>
    </tr>
    <tr>
      <th>2</th>
      <td>10015</td>
      <td>1959-08-19</td>
      <td>Guoxiang</td>
      <td>Nooteboom</td>
      <td>M</td>
      <td>1987-07-02</td>
      <td>27</td>
      <td>1993-08-22</td>
      <td>34</td>
      <td>6 years, 1 months</td>
      <td>2243</td>
    </tr>
    <tr>
      <th>3</th>
      <td>10021</td>
      <td>1960-02-20</td>
      <td>Ramzi</td>
      <td>Erde</td>
      <td>M</td>
      <td>1988-02-10</td>
      <td>27</td>
      <td>2002-07-15</td>
      <td>42</td>
      <td>14 years, 5 months</td>
      <td>5269</td>
    </tr>
    <tr>
      <th>4</th>
      <td>10025</td>
      <td>1958-10-31</td>
      <td>Prasadram</td>
      <td>Heyers</td>
      <td>M</td>
      <td>1987-08-17</td>
      <td>28</td>
      <td>1997-10-15</td>
      <td>38</td>
      <td>10 years, 1 months</td>
      <td>3712</td>
    </tr>
  </tbody>
</table>
</div>



Let's take a look at the distribution of time past employees have worked at the company.


```python
### Plotting the distribution with a line at the peak
p = sns.kdeplot(df['company_days'])
x,y = p.get_lines()[0].get_data()
index = np.argmax(y)
plt.vlines(x[index], 0, y[index])
plt.show()
```


![png](output_58_0.png)



```python
# Summary statistics on days spent at the company
pd.read_sql_query("""
SELECT MIN(company_days), MAX(company_days), AVG(company_days)
FROM
(SELECT a.*, b.to_date, b.leaving_age, CONCAT(TIMESTAMPDIFF(YEAR, a.hire_date, b.to_date), ' years, ', MOD(TIMESTAMPDIFF(MONTH, a.hire_date, b.to_date), 12), ' months') AS time_at_company, TIMESTAMPDIFF(DAY, a.hire_date, b.to_date) AS company_days
FROM
(SELECT employees.emp_no, birth_date, first_name, last_name, gender, hire_date, TIMESTAMPDIFF(YEAR, birth_date, hire_date) AS hire_age
FROM employees JOIN titles ON employees.emp_no = titles.emp_no
GROUP BY employees.emp_no) AS a
INNER JOIN
(SELECT employees.*, from_date, to_date, TIMESTAMPDIFF(YEAR, birth_date, to_date) AS leaving_age
FROM employees JOIN titles ON employees.emp_no = titles.emp_no
GROUP BY employees.emp_no
HAVING MAX(to_date) < '9999-01-01') AS b ON a.emp_no = b.emp_no) AS c;
""", conn)
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>MIN(company_days)</th>
      <th>MAX(company_days)</th>
      <th>AVG(company_days)</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>0</td>
      <td>6388</td>
      <td>2555.3828</td>
    </tr>
  </tbody>
</table>
</div>



We see that peak density occurs at approximately 1880 days. On average employees spend approximately 2555 days working at the company with the maximum being 6388 days (about 17 years, 5 months). Some employees have worked less than a full day.
