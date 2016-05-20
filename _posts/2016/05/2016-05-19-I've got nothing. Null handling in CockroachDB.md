---
title: "I've got nothing. NULL handling in CockroachDB"
categories: 
- cockroachdb
keywords: cockroachdb null
summary: "Not all SQL databases are created equal. Let's see how CockroachDB handles NULL values."
thumb: cockroachdb_logo_100x100.png
---

![Feature Image](/images/cockroachdb/cockroachdb_logo_200x200.png)

[CockroachDB](https://www.cockroachlabs.com/) is the new kid on the block and is growing fast. It's Beta software under rapid development and it's SQL layer is maturing fast.

While [SQL](https://en.wikipedia.org/wiki/SQL) is a standard, it has gone through many revisions and contains optional functionality that various vendors have chosen to implement, or not. The standard is also incomplete and ambiguous in many places, leading to different interpretations.

One of the design objectives with CockroachDB SQL is to make it as close to the PostgreSQL flavor of SQL as possible. CockroachDB has implemented the PostgreSQL wire protocol which enables any PostgreSQL client to operate with the database.
In terms of overall compatibility against the SQL standard, PostgreSQL is one of the more compliant implementations.   

This post looks at how CockroachDB handles *NULL* values and how it compares with the SQL standard and other SQL implementations.

The good people over at [SQLite](https://www.sqlite.org/) have a [script](https://www.sqlite.org/nulls.html) for testing *NULL*s and their functionality. I've used the contents of the script for my investigation into CockroachDB. The tests were carried out on Linux using version beta-20160512 of CockroachDB.

Let's create a table with some test data.

~~~sql
CREATE TABLE t1(
  a INT, 
  b INT, 
  c INT
);

INSERT INTO t1 VALUES(1, 0, 0);
INSERT INTO t1 VALUES(2, 0, 1);
INSERT INTO t1 VALUES(3, 1, 0);
INSERT INTO t1 VALUES(4, 1, 1);
INSERT INTO t1 VALUES(5, NULL, 0);
INSERT INTO t1 VALUES(6, NULL, 1);
INSERT INTO t1 VALUES(7, NULL, NULL);

SELECT * FROM t1;
+---+------+------+
| a |  b   |  c   |
+---+------+------+
| 1 |    0 |    0 |
| 2 |    0 |    1 |
| 3 |    1 |    0 |
| 4 |    1 |    1 |
| 5 | NULL |    0 |
| 6 | NULL |    1 |
| 7 | NULL | NULL |
+---+------+------+
~~~
Ok, so that's the first observation. The CockroachDB built-in client shows *NULL*s using the word **NULL** and not just an empty field, which is good because distinguishing between a *NULL* and a STRING column that contains an empty string ("") would be difficult.


## NULLs and Logic

What happens when a *NULL* value is used in a logic comparison? Like in the `WHERE` clause of a SQL statement.

~~~sql
SELECT * FROM t1 WHERE b < 10;
+---+---+---+
| a | b | c |
+---+---+---+
| 1 | 0 | 0 |
| 2 | 0 | 1 |
| 3 | 1 | 0 |
| 4 | 1 | 1 |
+---+---+---+

SELECT * FROM t1 WHERE NOT b > 10;
+---+---+---+
| a | b | c |
+---+---+---+
| 1 | 0 | 0 |
| 2 | 0 | 1 |
| 3 | 1 | 0 |
| 4 | 1 | 1 |
+---+---+---+

SELECT * FROM t1 WHERE b < 10 OR c = 1;
+---+------+---+
| a |  b   | c |
+---+------+---+
| 1 |    0 | 0 |
| 2 |    0 | 1 |
| 3 |    1 | 0 |
| 4 |    1 | 1 |
| 6 | NULL | 1 |
+---+------+---+

SELECT * FROM t1 WHERE b < 10 AND c = 1;
+---+---+---+
| a | b | c |
+---+---+---+
| 2 | 0 | 1 |
| 4 | 1 | 1 |
+---+---+---+

SELECT * FROM t1 WHERE NOT (b < 10 AND c = 1);
+---+------+---+
| a |  b   | c |
+---+------+---+
| 1 |    0 | 0 |
| 3 |    1 | 0 |
| 5 | NULL | 0 |
+---+------+---+

SELECT * FROM t1 WHERE NOT (c = 1 AND b < 10);
+---+------+---+
| a |  b   | c |
+---+------+---+
| 1 |    0 | 0 |
| 3 |    1 | 0 |
| 5 | NULL | 0 |
+---+------+---+

~~~

In summary, anything compared to a *NULL* is *NULL* (and therefore not TRUE). This behavior is consistent with PostgresSQL as well as all other major RDBMS's.


## NULLs and Arithmetic 

What sort of results do you get when a *NULL*s is part of an arithmetic expression?

~~~sql
SELECT a, b, c, b*0, b*c, b+c FROM t1;
+---+------+------+-------+-------+-------+
| a |  b   |  c   | b * 0 | b * c | b + c |
+---+------+------+-------+-------+-------+
| 1 |    0 |    0 |     0 |     0 |     0 |
| 2 |    0 |    1 |     0 |     0 |     1 |
| 3 |    1 |    0 |     0 |     0 |     1 |
| 4 |    1 |    1 |     0 |     1 |     2 |
| 5 | NULL |    0 | NULL  | NULL  | NULL  |
| 6 | NULL |    1 | NULL  | NULL  | NULL  |
| 7 | NULL | NULL | NULL  | NULL  | NULL  |
+---+------+------+-------+-------+-------+
~~~

In summary, any arithmetic operation involving a *NULL* will yield a *NULL* result.


## NULLs and Aggregate Functions

Aggregate functions are those that operate on a set of rows and return a single value. I've repeated the contents of t1 here to make it easier to understand the results.

~~~sql
SELECT * FROM t1;
+---+------+------+
| a |  b   |  c   |
+---+------+------+
| 1 |    0 |    0 |
| 2 |    0 |    1 |
| 3 |    1 |    0 |
| 4 |    1 |    1 |
| 5 | NULL |    0 |
| 6 | NULL |    1 |
| 7 | NULL | NULL |
+---+------+------+

SELECT COUNT(*), COUNT(b), SUM(b), AVG(b), MIN(b), MAX(b) FROM t1;
+----------+----------+--------+--------------------+--------+--------+
| COUNT(*) | COUNT(b) | SUM(b) |       AVG(b)       | MIN(b) | MAX(b) |
+----------+----------+--------+--------------------+--------+--------+
|        7 |        4 |      2 | 0.5000000000000000 |      0 |      1 |
+----------+----------+--------+--------------------+--------+--------+
~~~

Points to note:
- *NULL* values are **not** included in the `COUNT()` of a column that contains *NULL*s. 
- *NULL* values are **not** considered as high or low values in `MIN()` or `MAX()`.
- The `AVG()` is defined as `SUM()/COUNT()` which for column b is 2/4 not 2/7.


## *NULL*s as Distinct Values

Are *NULL* values considered distinct from other values?

~~~sql
SELECT DISTINCT b FROM t1;
+------+
|  b   |
+------+
|    0 |
|    1 |
| NULL |
+------+
~~~

The answer is yes. *NULL* values are included in the list of distinct values from a column containing *NULL*s.

However, counting the number of distinct values excludes them, which is consistent with the `COUNT()` function.

~~~sql
SELECT COUNT(DISTINCT b) FROM t1;
+-------------------+
| count(DISTINCT b) |
+-------------------+
|                 2 |
+-------------------+
~~~


## *NULL*s and Set Operations

How *NULL*s behave in a UNION query.

~~~sql
SELECT b FROM t1 UNION SELECT b FROM t1;
+------+
|  b   |
+------+
|    0 |
|    1 |
| NULL |
+------+
~~~

## *NULL*s and Sorting

Where do *NULL*s sit when sorting by a column containing *NULL* values? The core SQL standard doesn't explicitly define a sort order for *NULL*s but an optional extension to the [SQL:2003](https://en.wikipedia.org/wiki/SQL:2003) standard provided the `NULLS FIRST` and `NULLS LAST` addition to the `ORDER BY` clause.

By default, CockroachDB  orders *NULL*s **lower**  than the first non-*NULL* value, which is the same as using the `ORDER BY ... ASC` option. Which, by the way, is the same as MySQL and SQL Server.

CockroachDB hasn't implemented this optional extension so you can't change the position of the *NULL*s in the sorting order. I suspect this will be added to CockroachDB SQL in future as PostgreSQL provides the `NULLS FIRST` and `NULLS LAST` clauses.

As an aside, PostgreSQL orders *NULL*s **higher** than the last  non-*NULL* value, which is the same as using the `ORDER BY ... ASC NULLS LAST` or `ORDER BY ... DESC NULLS FIRST` options. Which, by the way, is the same as Oracle but **different** from CockroachDB. I also suspect this placement of *NULL*s within the sort order will be made the same as PostgreSQL soon.

~~~shell
SELECT * FROM t1 ORDER BY b;
+---+------+------+
| a |  b   |  c   |
+---+------+------+
| 6 | NULL |    1 |
| 5 | NULL |    0 |
| 7 | NULL | NULL |
| 1 |    0 |    0 |
| 2 |    0 |    1 |
| 4 |    1 |    1 |
| 3 |    1 |    0 |
+---+------+------+

SELECT * FROM t1 ORDER BY b DESC;
+---+------+------+
| a |  b   |  c   |
+---+------+------+
| 4 |    1 |    1 |
| 3 |    1 |    0 |
| 2 |    0 |    1 |
| 1 |    0 |    0 |
| 7 | NULL | NULL |
| 6 | NULL |    1 |
| 5 | NULL |    0 |
+---+------+------+
~~~

## *NULL*s and Unique Constraints 

Let's find out if *NULL* values are considered unique.

~~~sql
CREATE TABLE t2(a INT, b INT UNIQUE);

INSERT INTO t2 VALUES(1, 1);
INSERT INTO t2 VALUES(2, NULL);
INSERT INTO t2 VALUES(3, NULL);

SELECT * FROM t2;
+---+------+
| a |  b   |
+---+------+
| 1 |    1 |
| 2 | NULL |
| 3 | NULL |
+---+------+
~~~

This shows that *NULL*s are **not** considered unique as the `UNIQUE` constraint on column b was not violated when two rows with *NULL* values were inserted. Be aware that if a table has a `UNIQUE` constraint on column(s) that are optional (nullable), it is still possible to insert duplicate rows that appear to violate the constraint if they contain a *NULL* value in at least one of the columns. This is because *NULL*s are never considered equal and hence don't violate the uniqueness constraint.


## *NULL*s and Your Application

Unfortunately, how your version of SQL handles *NULL* values is only part of the puzzle. How your application code (and often more importantly, what it's written in) has a part to play in processing *NULL* values. That's a whole different story but one that you should know if you're pulling data from any SQL database.


