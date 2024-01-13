---
layout: post
title:  "Postgres Isolation Level By Example"
date:   2024-01-14 00:06:36 +0800
categories: postgres
permalink: postgres-isolation-level-by-example
---

As engineer, we don't put much attention on the Postgres isolation level. Most of the time relying on the default isolation level, which is `read committed` + locking is enough for most cases. 

However, in some specific cases, we need to change the isolation level either to `repeatable read` or `serialiazed`. Let's take a look at the example to understand the different between them.

Given a `bills` table for customer.

```sql
CREATE TABLE bills (
    id SERIAL PRIMARY KEY,
    date DATE,
    customer VARCHAR(20),
    total NUMERIC(10,2)
);
```

## Read Committed

Only show committed row. 

```sql
/* session 1 */                                 /* session 2 */
BEGIN;                                          BEGIN; 

INSERT INTO bills (date, customer, total) 
VALUES ('2024-01-01', 'Agnes', 10);

                                                SELECT * FROM bills;
                                                id | date | customer | total
                                                ----+------+----------+-------
                                                (0 rows)

COMMIT;

                                                SELECT * FROM bills;
                                                id |    date    | customer | total
                                                ----+------------+----------+-------
                                                1 | 2024-01-01 | Agnes    | 10.00
                                                (1 row)
```

## Repeatable Read

Repeatable read uses snapshot from Postgres MVCC. This isolation prevent us from phantom read & non repeatable read.

Add a seed data.
```sql
INSERT INTO bills (date, customer, total) VALUES ('2024-01-01', 'John', 100);
INSERT INTO bills (date, customer, total) VALUES ('2024-01-01', 'Agnes', 10);
```

**Read Committed Isolation**

```sql
/* session 1 */                                 /* session 2 */
BEGIN;                                          BEGIN; 

INSERT INTO bills (date, customer, total) 
VALUES ('2024-01-01', 'Tono', 50);
COMMIT;

                                                SELECT * FROM bills;
                                                id |    date    | customer | total
                                                ----+------------+----------+--------
                                                1 | 2024-01-01 | John     | 100.00
                                                2 | 2024-01-01 | Agnes    |  10.00
                                                3 | 2024-01-01 | Tono     |  50.00
                                                (3 rows)                
```

Customer **Tono** that is created from session 1, is visible to session 2 because of repeatable read.

**Repeatable Read Isolation**

```sql
/* session 1 */                                 /* session 2 */
BEGIN TRANSACTION ISOLATION                     BEGIN TRANSACTION ISOLATION
LEVEL REPEATABLE READ;                          LEVEL REPEATABLE READ;

INSERT INTO bills (date, customer, total) 
VALUES ('2024-01-01', 'Tono', 50);
COMMIT;

                                                SELECT * FROM bills;
                                                id |    date    | customer | total
                                                ----+------------+----------+--------
                                                1 | 2024-01-01 | John     | 100.00
                                                2 | 2024-01-01 | Agnes    |  10.00
                                                (2 rows)
```

Customer **Tono** that is created from session 1 is not visible to session 2. Because session 2 runs on his own snapshot and non repeatable read.

## Serializable

The Serializable isolation level provides the strictest transaction isolation. Postgres will execute the session one by one in serialize. To guarantee true serializability PostgreSQL uses predicate locking.

Example case is when customer's next month bill must be created from last month bill. This case require serializable isolation to prevent multiple bill for the same customer being created in the next month.

Add a seed data.
```sql
INSERT INTO bills (date, customer, total) VALUES ('2024-01-01', 'John', 100);
INSERT INTO bills (date, customer, total) VALUES ('2024-01-01', 'Agnes', 10);
```

**Repeatable Read Isolation**

```sql
/* session 1 */                                 /* session 2 */
BEGIN TRANSACTION ISOLATION                     BEGIN TRANSACTION ISOLATION
LEVEL REPEATABLE READ;                          LEVEL REPEATABLE READ;

INSERT INTO bills (date, customer, total) 
(
    SELECT 
        date + INTERVAL '1 MONTH', 
        customer, 
        total+100 
    FROM bills 
    WHERE customer='John' 
    ORDER BY date DESC LIMIT 1
);

                                                INSERT INTO bills (date, customer, total) 
                                                (
                                                    SELECT 
                                                        date + INTERVAL '1 MONTH', 
                                                        customer, 
                                                        total+100 
                                                    FROM bills 
                                                    WHERE customer='John' 
                                                    ORDER BY date DESC LIMIT 1
                                                );

COMMIT;                                         COMMIT;

SELECT * FROM bills;
 id |    date    | customer | total
----+------------+----------+--------
  1 | 2024-01-01 | John     | 100.00
  2 | 2024-01-01 | Agnes    |  10.00
  3 | 2024-02-01 | John     | 200.00
  4 | 2024-02-01 | John     | 200.00
(4 rows)

                                                SELECT * FROM bills;
                                                id |    date    | customer | total
                                                ----+------------+----------+--------
                                                1 | 2024-01-01 | John     | 100.00
                                                2 | 2024-01-01 | Agnes    |  10.00
                                                3 | 2024-02-01 | John     | 200.00
                                                4 | 2024-02-01 | John     | 200.00
                                                (4 rows)
```

Session 1 & Session 2 commits are accepted. This cause an issue for **John**, because on Feb 2024, he has 2 bills to pay.

**Serializable**

```sql
/* session 1 */                                 /* session 2 */
BEGIN TRANSACTION ISOLATION                     BEGIN TRANSACTION ISOLATION
LEVEL SERIALIZABLE;                             LEVEL SERIALIZABLE;

INSERT INTO bills (date, customer, total) 
(
    SELECT 
        date + INTERVAL '1 MONTH', 
        customer, 
        total+100 
    FROM bills 
    WHERE customer='John' 
    ORDER BY date DESC LIMIT 1
);

                                                INSERT INTO bills (date, customer, total) 
                                                (
                                                    SELECT 
                                                        date + INTERVAL '1 MONTH', 
                                                        customer, 
                                                        total+100 
                                                    FROM bills 
                                                    WHERE customer='John' 
                                                    ORDER BY date DESC LIMIT 1
                                                );

COMMIT;                                         COMMIT;

                                                ERROR:  could not serialize access due to read/write dependencies among transactions
                                                DETAIL:  Reason code: Canceled on identification as a pivot, during commit attempt.
                                                HINT:  The transaction might succeed if retried.

SELECT * FROM bills;
 id |    date    | customer | total
----+------------+----------+--------
  1 | 2024-01-01 | John     | 100.00
  2 | 2024-01-01 | Agnes    |  10.00
  3 | 2024-02-01 | John     | 200.00
(3 rows)

                                                SELECT * FROM bills;
                                                id |    date    | customer | total
                                                ----+------------+----------+--------
                                                1 | 2024-01-01 | John     | 100.00
                                                2 | 2024-01-01 | Agnes    |  10.00
                                                3 | 2024-02-01 | John     | 200.00
                                                (3 rows)
```

When transaction isolation level is serialized, then the commit execution will be done in order. Session 1 commit first & success. Session 2 get an error because the data already changes in Session 1.


## Lesson Learned
Lesson learned during my experience working with Postgres transaction in the last 7 years.

1. Only changes the isolation level if you know what you are doing. Otherwise, keep it as default & apply row/table lock when you need strong consistency.

2. The default isolation level in Postgres is `read committed`. For people come from MySQL, the mental model needs to be changed, since MySQL default isolation level is `repeatable read`.

3. When setting the isolation level to `repeatable read` or `serialized`, handle the retrial in your code.

4. Keep monitor your query performance, long running transaction & deadlock.


## Reference
- [https://www.postgresql.org/docs/current/transaction-iso.html](https://www.postgresql.org/docs/current/transaction-iso.html)
- [https://pgdash.io/blog/postgres-transactions.html](https://pgdash.io/blog/postgres-transactions.html)
- [https://mkdev.me/posts/transaction-isolation-levels-with-postgresql-as-an-example](https://mkdev.me/posts/transaction-isolation-levels-with-postgresql-as-an-example)