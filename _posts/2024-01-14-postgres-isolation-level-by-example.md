---
layout: post
title:  "Postgres Isolation Level By Example"
date:   2024-01-14 00:06:36 +0800
categories: postgres
permalink: postgres-isolation-level-by-example
---

As engineer, we don't put much attention on the Postgres isolation level. We rely on the default isolation level, which is `read committed`. 

However, in some specific cases, we need to change the isolation level either to `repeatable read` or `serialiazed`. Let's take a look at the example get understand the different between them.

Given a `bills` table for customer.

```sql
CREATE TABLE bills (
    id SERIAL PRIMARY KEY,
    date DATE,
    customer VARCHAR(20),
    total NUMERIC(10,2)
);
```

<h2>Read Committed</h2>

Only committed row will be shown in other transaction.

<table>
    <tr>
        <th>Session 1</th>
        <th>Session 2</th>
    <tr>
    <tr>
        <td><pre>BEGIN;</pre></td>
        <td><pre>BEGIN;</pre></td>
    </tr>
    <tr>
        <td>
<pre>
INSERT INTO bills (date, customer, total) 
VALUES ('2024-01-01', 'Agnes', 10);
</pre>
        </td>
        <td></td>
    </tr>
    <tr>
        <td></td>
        <td>
<pre>
SELECT * FROM bills;

 id | date | customer | total
----+------+----------+-------
(0 rows)
</pre> 
        </td>
    </tr>

    <tr>
        <td><pre>COMMIT;</pre></td>
        <td></td>
    </tr>
    <tr>
        <td></td>
        <td>
<pre>
SELECT * FROM bills;

 id |    date    | customer | total
----+------------+----------+-------
  1 | 2024-01-01 | Agnes    | 10.00
(1 row)
</pre> 
        </td>
    </tr>
</table>


<h2>Repeatable Read</h2>

Repeatable read uses snapshot from Postgres MVCC. This isolation prevent us from phantom read & non repeatable read.

```sql
INSERT INTO bills (date, customer, total) VALUES ('2024-01-01', 'John', 100);
INSERT INTO bills (date, customer, total) VALUES ('2024-01-01', 'Agnes', 10);
```

<b>read committed isolation</b>
<table>
    <tr>
        <th>Session 1</th>
        <th>Session 2</th>
    <tr>
    <tr>
        <td><pre>BEGIN;</pre></td>
        <td><pre>BEGIN;</pre></td>
    </tr>
    <tr>
        <td>
<pre>
INSERT INTO bills (date, customer, total) 
VALUES ('2024-01-01', 'Tono', 50);

COMMIT;
</pre>
        </td>
        <td></td>
    </tr>
    <tr>
        <td></td>
        <td>
<pre>SELECT * FROM bills;

 id |    date    | customer | total
----+------------+----------+--------
  1 | 2024-01-01 | John     | 100.00
  2 | 2024-01-01 | Agnes    |  10.00
  3 | 2024-01-01 | Tono     |  50.00
(3 rows)
</pre>
        </td>
    </tr>
</table>

Customer Tono that is created from session 1, leaks to session 2.

<b>repeatable read isolation</b>
<table>
    <tr>
        <th>Session 1</th>
        <th>Session 2</th>
    <tr>
    <tr>
        <td><pre>BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;</pre></td>
        <td><pre>BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;</pre></td>
    </tr>
    <tr>
        <td>
<pre>
INSERT INTO bills (date, customer, total) 
VALUES ('2024-01-01', 'Tono', 50);

COMMIT;
</pre>
        </td>
        <td></td>
    </tr>
    <tr>
        <td></td>
        <td>
<pre>SELECT * FROM bills;

 id |    date    | customer | total
----+------------+----------+--------
  1 | 2024-01-01 | John     | 100.00
  2 | 2024-01-01 | Agnes    |  10.00
(2 rows)
</pre>
        </td>
    </tr>
</table>

Customer Tono that is created from session 1 is not visible to session 2.

<h2>Serialized</h2>

<table>
</table>
