---
title: "Interviewing a database using SQL"
date: 2025-02-24
tags: ["sql", "database", "data"]
slug: interviewing-a-database-using-sql
---

I'm one of the co-founders of [Laboratório Hacker de Campinas](https://lhc.net.br/) (aka LHC), a hackerspace located in Campinas, Brazil. We organize several open events that are published in our [public calendar](https://eventos.lhc.net.br/).

After more than two years using this calendar, we have now a small dataset about our events that can be **interviewed** to answer some questions that may help us to plan the use of our space more efficiently.

I wanted to know when (and how) the space is being used and if I can find some pattern that would help us to schedule our future events in a way that will bring more people to visit us.

Before starting the "interview" of our database, I prepared some question that I expect to answer using SQL.

1. How many events did we have?
1. What is the month of the year that the most events occurred?
1. Is there any day of the week when an event is most likely to be happening in space?

## Our database

We use [Gancio](https://gancio.org) as a tool to manage our calendar ([I wrote about it before]({{< ref "20240208-configuring-self-hosted-calendar-for-small-community" >}})). Events data is stored in a SQLite database so I downloaded it and started to understand how it is structured.

The important tables to answer our questions are `events` and `places`. The first stores all information about each event, the second provide a list of venues where our events happen (the majority are in our hackerspace, but some are online or in a different venue).

A simplified definition of these tables (enough to follow this post) can be seen next (check [Gancio's](https://gancio.org) documentation to see the complete schema of these tables). You can also [download this simplified database](../../gancio.sqlite) if you want to try the comands too.

```sql
CREATE TABLE "places" (
    "id"    INTEGER,
    "name"  VARCHAR(255) NOT NULL,
    PRIMARY KEY("id")
);

CREATE TABLE "events" (
    "id"    INTEGER,
    "title" VARCHAR(255),
    "slug"  VARCHAR(255) UNIQUE,
    "start_datetime"    INTEGER,
    "placeId"   INTEGER,
    PRIMARY KEY("id"),
    FOREIGN KEY("placeId") REFERENCES "places"("id")
);
```

After understanding our data schema, we can start to interview our data. We will use [SQL](https://en.wikipedia.org/wiki/SQL) as the language to ask our questions.

So let's open the database in the terminal!

```bash
$ sqlite3 gancio.sqlite
SQLite version 3.44.1 2023-11-02 11:14:43
Enter ".help" for usage hints.
sqlite>
```

## How many events did we have?

We can answer that one with a [**SELECT**](https://www.sqlite.org/lang_select.html) statement and [**COUNT()**](https://www.sqlite.org/lang_aggfunc.html#aggfunclist) aggregate function.

```sql
-- Returning all events
SELECT * FROM events;
```

```sql
-- Counting all events
SELECT COUNT(*) FROM events;
```

The last query returns **145**, the number of events that happened all the time. Show the results grouped by year seems to be more useful in our analysis. This requires to perform a data cleanup in `start_datetime` field.

```bash
sqlite> SELECT title, start_datetime FROM events LIMIT 5;
1° Churrasco e Vinil|1684270800
Tutorial: Raspando Dados da Internet com Python|1688212800
Reunião mensal do LHC, dia 24 de maio de 2023|1684967400
Encontro GruPy-Campinas|1685793600
Reunião mensal do LHC, dia 19 de Junho de 2023|1687213800
```

We can notice that the event date is stored as a [Unix time](https://en.wikipedia.org/wiki/Unix_time), a format that is not easy to identify the year of the event.

We have [date and time functions](https://www.sqlite.org/lang_datefunc.html) in SQLite that can be used to convert this field to a datetime. Other databases, such as PostgreSQL oy MySQL, have similar functions to handle dates and times (check the docs of each DB to discover them).

In our case we will use `datetime()` function.

```sql
SELECT
    start_datetime,
    datetime(start_datetime, 'unixepoch')
FROM
    events
LIMIT 5;
```

The first 5 results of this query will look like:

```bash
1684270800|2023-05-16 21:00:00
1688212800|2023-07-01 12:00:00
1684967400|2023-05-24 22:30:00
1685793600|2023-06-03 12:00:00
1687213800|2023-06-19 22:30:00
```

Using `strftime()` function, we can extract only the year of each event as a column:

```sql
SELECT
    start_datetime,
    datetime(start_datetime, 'unixepoch'),
    strftime(
        '%Y',
        datetime(start_datetime, 'unixepoch')
    ) AS event_year
FROM
    events
LIMIT 5;
```

```bash
1684270800|2023-05-16 21:00:00|2023
1688212800|2023-07-01 12:00:00|2023
1684967400|2023-05-24 22:30:00|2023
1685793600|2023-06-03 12:00:00|2023
1687213800|2023-06-19 22:30:00|2023
```

Now we have a `event_year` column that we can use to group our results and have the counting of events separated by year. We need to add [**GROUP BY**](https://www.sqlite.org/lang_select.html#resultset) clause to our initial query to get that.

```sql
SELECT
    strftime(
        '%Y',
        datetime(start_datetime, 'unixepoch')
    ) AS event_year,
    COUNT(*) AS num_events
FROM
    events
GROUP BY
    event_year;
```

And we have the following results:

```
2023|30
2024|88
2025|27
```

We had **30 events** in 2023, **88** in 2024 and **27** in 2025. As we started using Gancio only in May, 2023, our dataset doesn't have more than one entire year of data other than 2024. So we can't affirm that there is a growing trend in the number of events. When 2025 is over we will have more data to compare.

## What is the month of the year that the most events occurred?

Maybe we can do a better analysis if we consider the year and the month (so we can compare the months that we have data in all years). We can group our results by multiple fields such as:

```sql
SELECT
    strftime('%Y', datetime(start_datetime, 'unixepoch')) AS event_year,
    strftime('%m', datetime(start_datetime, 'unixepoch')) AS event_month,
    count(*) as num_events
FROM events
GROUP BY event_year, event_month;
```

Ordering the results by the total number of events in descending order:

```sql
SELECT
    strftime('%Y', datetime(start_datetime, 'unixepoch')) AS event_year,
    strftime('%m', datetime(start_datetime, 'unixepoch')) AS event_month,
    count(*) as num_events
FROM events
GROUP BY event_year, event_month
ORDER BY num_events DESC;
```

We discovered that August 2024 was the month with the highest number of events (10)! Compared with August 2023 (only 2 events) we had a huge increase. However, because we don't have enough data (less than 3 years), we can't say that August will always be the month with the higher number of events.

## Is there any day of the week when an event is most likely to be happening in space?

Maybe we can have more insights if we look at the data grouped by day of week. This can be valuable information for someone that would like to visit us unannounced, but that would like to have a better chance of meeting someone in space.

The option `%w` returns an integer from 0-7 each one representing one day of week (starting on Sunday).

```sql
SELECT
    COUNT(*) AS num_events,
    strftime('%w', datetime(start_datetime, 'unixepoch')) AS day_of_week
FROM
    events
GROUP BY
    day_of_week
ORDER BY
    num_events
DESC;
```

```
43|6
29|4
26|1
18|3
14|5
14|2
1|0
```

We have 6 (Saturday) with **43** events, and then 4 (Thursday) with **29** events. It appears that if someone wants to visit us, if they appears on Saturday or Thursday, there is a better chance to have an event happening there.

Having numbers to reference weekday is not user-friendly, so using [`CASE` expression](https://www.sqlite.org/lang_expr.html) (that serves a role similar to IF-THEN-ELSE in other programming languages), we can have a nicer result.

```sql
SELECT COUNT(*) AS num_events,
    CASE
        strftime('%w', datetime(start_datetime, 'unixepoch'))
        WHEN '0' THEN 'Sunday'
        WHEN '1' THEN 'Monday'
        WHEN '2' THEN 'Tuesday'
        WHEN '3' THEN 'Wednesday'
        WHEN '4' THEN 'Thursday'
        WHEN '5' THEN 'Friday'
        WHEN '6' THEN 'Saturday'
    END AS day_of_week
FROM events
GROUP BY day_of_week
ORDER BY num_events desc;
```

```
43|Saturday
29|Thursday
26|Monday
18|Wednesday
14|Tuesday
14|Friday
1|Sunday
```

But as mentioned before, sometimes we have online events (or events in a different venue). So we should consider in my query only the events that are in-person in the hackerspace.

We need to combine data from `events` and `places`. First let's see how places are returned.

```sql
SELECT id, name from places;
```

```
1|Laboratório Hacker de Campinas
2|online
3|ARCA
4|Anhembi
6|UNISAL - Campus São José
7|SIRIUS
8|Candreva Proença
```

In `events` table we have `placeId` field that relates both tables. Using a [`JOIN` operator](https://www.sqlite.org/lang_select.html) together with a `WHERE` clause for filtering, it is possible to return only the events that happened in our place.

```sql
SELECT COUNT(*) AS num_events,
    CASE
        strftime('%w', datetime(start_datetime, 'unixepoch'))
        WHEN '0' THEN 'Sunday'
        WHEN '1' THEN 'Monday'
        WHEN '2' THEN 'Tuesday'
        WHEN '3' THEN 'Wednesday'
        WHEN '4' THEN 'Thursday'
        WHEN '5' THEN 'Friday'
        WHEN '6' THEN 'Saturday'
    END AS day_of_week
FROM events
JOIN places ON places.id = events.placeId
WHERE places.name = 'Laboratório Hacker de Campinas'
GROUP BY day_of_week
ORDER BY num_events desc;
```

```
39|Saturday
27|Thursday
18|Wednesday
13|Tuesday
13|Friday
6|Monday
```

The results are quite different, but Saturday and Thursday are still the busiest days of the week. Monday, on the contrary, which appeared with 26 events before, had only 6 events at our house!

## What now?

What we can do with this information? If I want to visit LHC at random, probably I would try to go there on Saturday or Thursday!

But if I want to improve the use of space, so that we have more activities on other days of the week, maybe organizing more events on Tuesday or Friday is an option.

Understanding why these other days are not the favorites is also important. Sunday not having any events makes sense to me, since it's a day that people prefer to stay at home with family.

Interviewing this dataset gave me some insights, but opened more questions in my mind. Some probably I will be able "to ask" to my database. Others I will need to talk with our hackerspace members to figure out.