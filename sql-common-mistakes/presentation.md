
# SQL Common Mistakes

Julia Strauß

---

# Introduction

----

## Agenda

* Common Mistakes
	* Group By
	* Select *
	* Incorrect column in subquery
	* Scalar subquery
	* Logical vs. physical evaluation order
* Inefficient SQL
	* Indexes
	* Select random entries
* NULL

----

## Slides

* Presentation
	* https://bitbucket.int.tngtech.com/projects/MSD/repos/sql-common-mistakes/browse

---

# Common Mistakes

----

## Group By

``` 
SELECT last_name, MAX(age)
FROM persons 
GROUP BY last_name;
``` 

* Developer wants to add column for first name to query because he also wants the name associated with max age
	* Problem arises from error in reasoning
	* New column is added to SELECT

----

## Group By

```
SELECT last_name, first_name, MAX(age)
FROM persons 
GROUP BY last_name;
```
* Mainly only a problem in MySQL and SQLite
	* MySQL returns first physical row of group 
	* SQLite returns last physical row of group
* Leads to error in Oracle, SQL Server, ...

----

## Select *

```
INSERT INTO ExternalPersons 
SELECT *
FROM Persons 
WHERE status = 'external'
```
* New column added to 
	* ALTER TABLE Persons ADD age NVARCHAR2
* Next insert
	* Error: ORA-00913: too many values

----

## Select *

* Solution: Name columns explictly

```
INSERT INTO ExternePersons (om, name, status) 
SELECT om, name, status
FROM Persons
WHERE status = 'external'
```

----

## Incorrect column in subquery

* Intention: All persons born in Norway
```
SELECT name, birth_state
FROM Persons
WHERE birth_state IN (
	SELECT birth_state
	FROM StateKeys 
	WHERE staat_txl = 'Norway'
)
```
* Result: All persons in table

----

## Incorrect column in subquery

* Solution: Reference columns by named table
```
SELECT p.name, p.birth_state
FROM Persons AS p
WHERE p.birth_state IN (
	SELECT s.birth_state
	FROM StateKeys AS s 
	WHERE s.staat_txl = 'Norway'
)
```
* Makes it possible to spot mistake due to error

----

## Scalar subquery

| p_nr | product |
|:--|:--|
| 1 | Knife |
| 2 | Fork |
| 3 | Spoon |

| p_nr | factory_nr |
|:--|:--|
| 1 | 1 |
| 2 | 1 |
| 3 | 2 |

----

## Scalar subquery

```
SELECT p_nr, product, (
	SELECT factory_nr
	FROM Production AS B
	WHERE B.p_nr = A.p_nr) AS factory_nr 
FROM Products AS A;
```

| p_nr | product | factory_nr |
|:--|:--|:--|
| 1 | Knife | 1 |
| 2 | Fork | 1 |
| 3 | Spoon | 2 |

----

## Scalar subquery

* Works fine first, but one day
	* Msg 512, Level 16, State 1, Line 1\
Subquery returned more than 1 value. This is not permitted when the subquery follows =, !=, <, <= , >, >= or when the subquery is used as an expression.
* Change 
	* Knives now also produced in second factory

----

## Scalar subquery

One Possible solution

```
SELECT A.p_nr, A.produkt, B.fabrik_nr 
FROM Products AS A
JOIN Production AS B
ON A.p_nr = B.p_nr;
```


----

## Logical vs. physical evaluation order

* Logical evaluation order
```
SELECT om, COUNT(nation) AS number     -- 5
FROM nationalities                     -- 1
WHERE not_eu IS NULL                   -- 2
GROUP BY om                            -- 3
HAVING COUNT(nation) > 2.              -- 4
ORDER BY number DESC;                  -- 6
```
* e.g. can only use `number` in ORDER BY but not in HAVING

----

## Logical vs. physical evaluation order

* Problematic SQL and table `Accounts`

```
SELECT id, reference AS ref_nr 
FROM Accounts
WHERE type LIKE 'Number%'
AND CAST(reference AS INT) > 20;
```

| id | type     | value  |
|:--:|:--------:|:------:|
| 1  | Number  | 210    |
| 2  | Text     | Text   |
| 3  | Number  | 34     |

----

## Logical vs. physical evaluation order

* Next try

```
SELECT id, ref_nr FROM (
	SELECT id, CAST(reference AS INT) AS ref_nr 
	FROM Accounts
	WHERE type LIKE 'Number%') AS A 
WHERE ref_nr > 20;
```

----

## Logical vs. physical evaluation order

* Possible solution
```
SELECT id, reference AS ref_nr 
FROM Accounts
WHERE typ LIKE 'Number%'
AND (CASE 
	WHEN reference NOT LIKE '%[^0-9]%'
	THEN CAST(reference AS INT) END) > 20;
```
* Learnings
	* Physical evaluation order can differ from logical one
	* Don't save different data types in the same column

---

# Inefficient SQL

----

## Indexes

* Too few indexes
	* Search on non-indexed columns need to do Full Table Scan to get result

----

## Indexes

* Too many indexes
	* Additional index on Primary Key column
		* Most DBMS create index for PK column
	* Index on long String-like columns like VARCHAR(300)
		* depend on situation
		* Index can get huge
		* columns like `content`, `description` etc. are less likely to be searched

----

## Indexes

* Too many indexes which are not used in search
	* Indexes on columns with little value variations like gender
	* Duplicated Indexes
		* Example
			* Index on surname, firstname
			* Index on surname
	* Order is relevant

----

## Indexe

* Queries, which cannot be supported by specific index
	* Assumption: Index for (surname, firstname)
	* Query:

```	
SELECT * FROM Persons 
ORDER BY firstname, surname;
SELECT * FROM Persons 
WHERE surname='Ben' OR firstname='Ben';

SELECT * FROM Persons 
WHERE surname like '%Ben%';
SELECT * FROM Persons 
WHERE SUBSTRING(surname,1,4)='Ben';
```

----

## Select random entries

```
SELECT *
FROM Products
ORDER BY RAND()
LIMIT 1;
```

* Common solution
	* Bad performance
		* Needs to sort whole table
		* Cannot use any indexes
	* Nearly all work for nothing because only one value is used

----

## Select random entries

* Alternatives
	* Use knowledge over data
		* how are keys distributed (are there spaces, etc.)
	* Use special vendor functions
		* row_number()
		* Sample-function

```
SELECT * FROM ( SELECT *
FROM Persons SAMPLE(1)
ORDER BY DBMS_RANDOM.VALUE) WHERE rownum<=1
```

---

# NULL

----

## NULL in SQL

* SQL has ternary logic
	* NULL means no value exists
		* it is a state and not a value
* If one would ask how many books does Peter own?
	* NULL would mean we don't know how many books he owns, not that he doesn't own books
* All real RDMS support the presentation of unknown or missing data

----

## Counting with NULL

```
SELECT 'GERMAN' AS STATE, COUNT(*) AS COUNT 
FROM Persons WHERE state ='000' 
UNION ALL
SELECT 'NON_GERMAN', COUNT(*) FROM Persons 
WHERE NOT(state ='000')
UNION ALL
SELECT 'ALL', COUNT(*) FROM Persons
```

| STATE | COUNT |
|:--|--:|
| GERMAN | 2396 |
| NON_GERMAN | 1367 |
| ALL | 13109 |

----

## Comparison with NULL

| Expression | Result |
|:--|:--|
| `NULL AND TRUE` | NULL |
| `NULL AND FALSE` | FALSE |
| `NULL OR TRUE` | TRUE |
| `NULL OR FALSE` | NULL |
| `NOT(NULL)` | NULL |

----

## Comparison with NULL

* NULL cannot be compared with `=` or `<>` because the result can never be true
* Even NULL cannot be compared with NULL
* Attention: Oracle treats empty string like NULL
	* Common Mistake: `trim(description)=''`

----

## String Concatenation with NULL

* Differs between vendors
	* Oracle treats NULL like empty string so concatenation successful
	* e.g. SQL Server results in NULL string if any of the concatenated values where null

[//]: # (Resulting NULL makes more sense because concatenating something unknown to something known can only result in unknown)

----

## Sorting with NULL

* Differs between different vendors
	* e.g. Oracle sorts NULL to the end, SQL Server to the beginning
	* Can change behavior by using NULLS FIRST (Oracle)

----

## Aggregate with NULL

| i | j |
|:--:|:--:|
| 200 | 200 |
| NULL | 0 |

```
SELECT AVG(i),AVG(j)
FROM example
```

----

## Codds Proposal

* NULLs are not equal, but not distinct
* Definition:
	* "any two values that are equal to one another, or any two Nulls [...] are [...] not distinct“
* Affected SQL commands
	* GROUP BY
	* PARTITION BY
	* UNION, INTERSECT, EXCEPT
	* DISTINCT keyword
* On the grounds of Codds 1979 ProposaL

---

# Source

* Bill Karwin: SQL Antipatterns
* John L. Viescas, Douglas J. Steele, Ben G. Clothier: Effective SQL: 61 Specific Ways to Write Better SQL
* https://en.wikipedia.org/wiki/Null_%28SQL%29
* https://www.red-gate.com/simple-talk/sql/t-sql-programming/ten-common-sql-programming-mistakes/
* Trial & Error ;-)