---
title: "Querying hackerspace donations with SQL"
date: 2026-10-27
tags: ["sql", "data", "data-analysis"]
slug: querying-hackerspace-donations-with-sql
---

I am reading [SQL for Data Analytics](https://www.amazon.com/dp/1492088781) to improve
my skills in SQL and data analysis. While going through the first few chapters, I came across 
a passage stating that data analysis goes beyond simply arriving at the correct number;
it is about having curiosity and formulating questions to uncover the "why" behind the numbers.

I then began to think on where I might find data that would allow me to ask questions and 
exercise this curiosity. As it turns out, at my [local hackerspace](https://lhc.net.br), we 
have a database that records all financial transactions and donations.

Most of the donations we receive come from people who purchase snacks or drinks from our fridge.
Occasionally, we also receive one-off contributions, given without any specific reason. I became 
curious about who the top donors were this year and the total value of their contributions.

Based on this information, I began to wonder whether these individuals who are not yet members
might be more inclined to accept an invitation to join. To explore this, I started running some 
tests using basic SQL.

## Data structure

For this analysis I will search for data in two tables: `bookkeeping_transaction` that contains all transactions
and `bookkeeping_category` that is an auxiliary table that contains the transaction categories (so we can filter
only the donations).

```sql
CREATE TABLE bookkeeping_category (
  id INT PRIMARY KEY,
  name VARCHAR(64) NOT NULL
);

CREATE TABLE bookkeeping_transaction (
  id INT PRIMARY KEY,
  date DATE NOT NULL,
  description VARCHAR(256) NOT NULL,
  amount NUMERIC(10, 2) NOT NULL,
  category_id INT REFERENCES bookkeeping_category(id)
);
```

## How many donations?

We are interested in donation transactions ("_Doação_" in Portuguese), so we filter them
using a `SELECT` statement on `bookkeeping_transaction` table together with a `JOIN` clause 
to `bookkeeping_category` table so I can restrict to the transactions of the right category
("_Doação_") within the year of 2026.

```sql
SELECT
  count(*)
FROM
  bookkeeping_transaction bt
JOIN
  bookkeeping_category bc ON bc.id = bt.category_id 
WHERE
  bc.name = 'Doação'
  AND bt.date BETWEEN '2026-01-01' AND '2026-12-31'
```

> Note on the use of `BETWEEN`, the range is *inclusive*, so both dates (2026-01-01 and 2026-12-31) **are included**
> in the results.

## How many donations per person?

We need to groups the transactions to person responsible for each donation. We have a known
pattern for the description of these transactions (mostly):

```
Transf Pix recebida - Margaret Hamilton - 123.456.789-10
```

`Transf Pix recebida` informs how the money was receiveid in our bank account. `Margaret Hamilton`
is the name of the donor and `123.456.789-10` is the brazilian government ID.

Every time a person donates, the transaction description will be the same, just replacing
the amount and date, so if we group them by `description` and count the number of transaction, we are
able to identify how many donations a person made in the period.

Using the filters from the previous query, grouping by `description` (`GROUP BY`), aggregating by the number
of transactions (`count()`), ordering by the total number of transactions for each 
person (`ORDER BY` and `DESC`) and limitig (`LIMIT`) to the top 10, it is possible to know which persons made
more donations within the year:

```sql
SELECT
  bt.description,
  count(bt.description) AS total_transactions
FROM
  bookkeeping_transaction bt
JOIN
  bookkeeping_category bc ON bc.id = bt.category_id 
WHERE
  bc.name = 'Doação'
  AND bt.date BETWEEN '2026-01-01' AND '2026-12-31'
GROUP BY
  bt.description
ORDER BY
  total_transactions DESC
LIMIT 10
```

```
                               description                      | total_transactions
----------------------------------------------------------------+--------------------
 Transf Pix recebida - GRACE HOPPER - 111.111.111-11            |                155
 Transf Pix recebida - Ada Lovelace King - 222.222.222-22       |                 29
 Transf Pix recebida - MARGARET HAMILTON - 333.333.333-33       |                 19
 Transf Pix recebida - BARBARA LISKOV - 444.444.444-44          |                 14
 Transf Pix recebida - Frances Elizabeth Allen - 555.555.555-55 |                 13
 Transf Pix recebida - RADIA PERLMAN - 666.666.666-66           |                 12
 Transf Pix recebida - Shafi Goldwasser - 777.777.777-77        |                 11
 Transf Pix recebida - Karen Sparck Jones - 888.888.888-88      |                 11
 Transf Pix recebida - Jean Jennings Bartik - 999.999.999-99    |                 11
 Transf Pix recebida - Adele Goldberg - 000.000.000-00          |                 10
```

We can see that `GRACE HOPPER` donated 155 times during the year. It doesn't mean that she donated the 
higher amount in total.

## How much was donated per person?

Instead of aggregating with `count()`, counting the number of rows each person has, we will sum (`sum()`) the amount
donated. The query is exactly the same, just changing the aggregation function:

```sql
SELECT
  bt.description,
  sum(bt.amount) AS total_amount
FROM
  bookkeeping_transaction bt
JOIN
  bookkeeping_category bc ON bc.id = bt.category_id 
WHERE
  bc.name = 'Doação'
  AND bt.date BETWEEN '2026-01-01' AND '2026-12-31'
GROUP BY
  bt.description
ORDER BY
  total_amount DESC
LIMIT 10
```

```
                        description                          | total_amount
-------------------------------------------------------------+--------------
 Transf Pix recebida - Jean Jennings Bartik - 999.999.999-99 |       517.00
 Transf Pix recebida - Ada Lovelace King - 222.222.222-22    |       494.49
 Transf Pix recebida - Fei-Fei Li - 101.010.101-01           |       415.60
 Transf Pix recebida - Lynn Conway - 202.020.202-02          |       410.00
 Transf Pix recebida - SOPHIE WILSON - 303.030.303-03        |       408.00
 Transf Pix recebida - BARBARA LISKOV - 444.444.444-44       |       397.00
 Transf Pix recebida - Anita Borg - 505.050.505-05           |       384.00
 Transf Pix recebida - MARY KENNETH KELLER - 606.060.606-06  |       350.00
 Transf Pix recebida - CYNTHIA DWORK - 707.070.707-07        |       300.00
 Transf Pix recebida - Karen Sparck Jones - 888.888.888-88   |       272.00 
```

Now we can see who donated the highest amount over the entire period, even if they did not make the 
highest number of donation transactions. The names and the ordering differ as expected since a single 
high-value donation can far exceed hundreds of small-value donations.

## Data cleaning

We can't rely entirely on this grouping, as for the same person we may have slightly different descriptions
just varying the capitalization of it such as:

```
Transf Pix recebida - Margaret Hamilton - 123.456.789-10
```

and

```
Transf Pix recebida - MARGARET HAMILTON - 333.333.333-33
```

If we keep grouping like before each transaction will be considering that we are looking for different
people. Converting the `description` field to uppercase will resolve the issue; this way, we can 
group based on the same values, regardless of how they originally appear. This can be done using the `upper()`
fucntion:

```sql
SELECT
  upper(bt.description),
  sum(bt.amount) AS total_amount
FROM
  bookkeeping_transaction bt
JOIN
  bookkeeping_category bc ON bc.id = bt.category_id 
WHERE
  bc.name = 'Doação'
  AND bt.date BETWEEN '2026-01-01' AND '2026-12-31'
GROUP BY
  upper(bt.description)
ORDER BY
  total_amount DESC
LIMIT 10
```

Notice that we use `upper(bt.description)` in two places within the query. As the functions applied to a field become
more complex, this duplication makes the SQL query harder to read. `GROUP BY` clause allows for positional group,
where we specify the position of the field (or fields) upon which we wish to perform the grouping. The following is equivalent
to the previous query:

```sql
SELECT
  upper(bt.description),
  sum(bt.amount) AS total_amount
FROM
  bookkeeping_transaction bt
JOIN
  bookkeeping_category bc ON bc.id = bt.category_id 
WHERE
  bc.name = 'Doação'
  AND bt.date BETWEEN '2026-01-01' AND '2026-12-31'
GROUP BY
  1
ORDER BY
  total_amount DESC
LIMIT 10
```

```
                            upper                            | total_amount
-------------------------------------------------------------+--------------
 TRANSF PIX RECEBIDA - JEAN JENNINGS BARTIK - 999.999.999-99 |       517.00
 TRANSF PIX RECEBIDA - ADA LOVELACE KING - 222.222.222-22    |       502.49
 TRANSF PIX RECEBIDA - ELIZABETH FEINLER - 808.080.808-08    |       474.00
 TRANSF PIX RECEBIDA - KAREN SPARCK JONES - 888.888.888-88   |       464.00
 TRANSF PIX RECEBIDA - FEI-FEI LI - 101.010.101-01           |       415.60
 TRANSF PIX RECEBIDA - LYNN CONWAY - 202.020.202-02          |       410.00
 TRANSF PIX RECEBIDA - SOPHIE WILSON - 303.030.303-03        |       408.00
 TRANSF PIX RECEBIDA - BARBARA LISKOV - 444.444.444-44       |       397.00
 TRANSF PIX RECEBIDA - ANITA BORG - 505.050.505-05           |       384.00
 TRANSF PIX RECEBIDA - MARY KENNETH KELLER - 606.060.606-06  |       350.00
```

Analyzing the results, we can now see that `ADA LOVELACE KING` donated a total of `502.49` (it 
was `494.49` before data cleaning), and `ELIZABETH FEINLER` appeared in the list, which may indicate that 
there are transactions citing this person with different capitalization that affected her position in
the initial analysis.

## String processing

`TRANSF PIX RECEBIDA - MARY KENNETH KELLER - 606.060.606-06` is not an ideal representation of the
desired outcome, as we are only interested in the person's name. Therefore, we can proceed with 
data processing by using a regular expression and the `substring()` function to parse this string
 and return only the names of the individuals in the results:

```sql
SELECT
  substring(
    upper(bt.description) FROM ' - [0-9\. ]*([A-Z .*]*) - '
  ) AS donor_name,
  sum(bt.amount) AS total_amount
FROM
  bookkeeping_transaction bt
JOIN
  bookkeeping_category bc ON bc.id = bt.category_id 
WHERE
  bc.name = 'Doação'
  AND bt.date BETWEEN '2026-01-01' AND '2026-12-31'
  AND substring(
    upper(bt.description) FROM ' - [0-9\. ]*([A-Z .*]*) - '
  ) IS NOT NULL
GROUP BY
  1
ORDER BY
  total_amount DESC
LIMIT 10
```

```
      donor_name      | total_amount
----------------------+--------------
 [NULL]               |      4571.65
 JEAN JENNINGS BARTIK |       517.00
 ADA LOVELACE KING    |       502.49
 ELIZABETH FEINLER    |       474.00
 KAREN SPARCK JONES   |       464.00
 FEI-FEI LI           |       415.60
 LYNN CONWAY          |       410.00
 SOPHIE WILSON        |       408.00
 BARBARA LISKOV       |       397.00
 ANITA BORG           |       384.00
```

The first row in the results does not show a name, but rather a null value. This means there are rows for
which the regular expression did not find a match. When the `substring(field FROM pattern)` function fails to 
find the pattern within the field's content, it returns a null value instead of an empty string.

As we group by the extracted name, all transactions that do not find a match end up in a single NULL group, 
which appears as an empty value. It is likely that the descriptions for these transactions do not follow
the `Transf Pix received - NAME - DOCUMENT` format, or that the regular expression does not account for 
all possible variations—such as accented characters (Ã, É, Ç), apostrophes, or hyphens.

We must correct the regular expression until all values are grouped correctly, or we could simply ignore 
them by excluding NULL values from the results. However, doing so would mean leaving out `4,571.65` in donations, 
a figure that is not insignificant:

```sql
SELECT
  substring(
    upper(bt.description) FROM ' - [0-9\. ]*([A-Z .*]*) - '
  ) AS donor_name,
  sum(bt.amount) AS total_amount
FROM
  bookkeeping_transaction bt
JOIN
  bookkeeping_category bc ON bc.id = bt.category_id 
WHERE
  bc.name = 'Doação'
  AND bt.date BETWEEN '2026-01-01' AND '2026-12-31'
  AND substring(
    upper(bt.description) FROM ' - [0-9\. ]*([A-Z .*]*) - '
  ) IS NOT NULL
GROUP BY
  1
ORDER BY
  total_amount DESC
LIMIT 10
```

We can use the `coalesce()` function to replace NULL values with a text string, effectively treating 
all other transactions that do not match the regular expression as a single value. We provide to 
the `coalesce()` function two arguments: the first is the name of the field (or field expression) to 
be displayed, and the second is the value the field should take if the first one is NULL.

```sql
SELECT
  coalesce(
    substring(
      upper(bt.description) FROM ' - [0-9\. ]*([A-Z .*]*) - '
    ),
    'OTHER DONORS'
  ) AS donor_name,
  sum(bt.amount) AS total_amount
FROM
  bookkeeping_transaction bt
JOIN
  bookkeeping_category bc ON bc.id = bt.category_id 
WHERE
  bc.name = 'Doação'
  AND bt.date BETWEEN '2026-01-01' AND '2026-12-31'
GROUP BY
  1
ORDER BY
  total_amount DESC
LIMIT 10
```

```
      donor_name      | total_amount
----------------------+--------------
 OTHER DONORS         |      4571.65
 JEAN JENNINGS BARTIK |       517.00
 ADA LOVELACE KING    |       502.49
 ELIZABETH FEINLER    |       474.00
 KAREN SPARCK JONES   |       464.00
 FEI-FEI LI           |       415.60
 LYNN CONWAY          |       410.00
 SOPHIE WILSON        |       408.00
 BARBARA LISKOV       |       397.00
 ANITA BORG           |       384.00
```

But that amount is still significant, which means that our description pattern is not being enough 
to group all persons.

Digging into the data, there is the second pattern for the donations:

```BERTHA LUTZT - 12345678910```

We need a better regular expression. As we know the first expression is good enough for a well 
established pattern and extending a regular expression to be more general can be very difficult, we 
can use the fact that `coalesce()` function can receive virtually any number or arguments.

It starts evaluating the arguments from left to right and stops at the first non-NULL argument. We
can try different regular expressions in order, and return a default value if none match the string.

```sql
SELECT
  coalesce(
    substring(
      upper(bt.description) FROM ' - [0-9\. ]*([A-Z .*]*) - '
    ),  -- FIRST PATTERN ("TRANSF PIX RECEBIDA - NAME - ID")
    substring(
      upper(bt.description) FROM '^([A-ZÇÁÃÕ ]+)'
    ),  -- SECOND PATTERN ("NAME - ID")
    'OTHER DONORS'
  ) AS donor_name,
  sum(bt.amount) AS total_amount
FROM
  bookkeeping_transaction bt
JOIN
  bookkeeping_category bc ON bc.id = bt.category_id 
WHERE
  bc.name = 'Doação'
  AND bt.date BETWEEN '2026-01-01' AND '2026-12-31'
GROUP BY
  1
ORDER BY
  total_amount DESC
LIMIT 10
```

```
      donor_name      | total_amount
----------------------+--------------
 JEAN JENNINGS BARTIK |       517.00
 ADA LOVELACE KING    |       502.49
 ELIZABETH FEINLER    |       474.00
 KAREN SPARCK JONES   |       464.00
 FEI-FEI LI           |       415.60
 LYNN CONWAY          |       410.00
 SOPHIE WILSON        |       408.00
 BARBARA LISKOV       |       397.00
 ANITA BORG           |       384.00
 BERTHA LUTZ          |       324.00
```

Now, after all this data processing using SQL, our results are robust and consistent. Every donation 
amount is linked to an individual, which allows us to use this information to answer our initial question.

# Conclusion

I started this whole thing with the idea that if we knew who the people making the largest financial 
donations (excluding membership fees) to the hackerspace were, we could invite them to become members.

Now, after a lot of SQL, we have real, clean, and structured data that can help test this idea. I don’t 
know if I’ll actually manage to bring in any new members for the hackerspace, but it was a lot of fun 
playing around with the data like this!