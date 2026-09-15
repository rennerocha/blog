---
title: "Avoiding N+1 query problems using new Django's model fetch modes"
date: 2026-09-15
tags: ["python", "django"]
slug: avoiding-n-plus-1-problems-with-django-model-fetch-modes
---

[Django 6.1](https://docs.djangoproject.com/en/6.1/releases/6.1/#django-6-1-release-notes) was released in August 5, 2026 and introduced the [model fetch modes](https://docs.djangoproject.com/en/6.1/releases/6.1/#model-field-fetch-modes), a new feature that
have the potential to protect us from the **N+1 queries problem**.

This problem occurs when retrieving a collection of objects, and then accessing related
objects for each item in that collection. Instead of using a single query to fetch all data,
Django will issues one query to fetch the main object and then N additional queries
(N is the number of items in the collection) for the related objects not fetched
initially. In large dataset, this can create a severe impact in the application's performance.

With the model fetch modes, we can avoid, or at least reduce the impact of scenarios like that.

As an example, consider an event application that have the following models: `Event` for the event
details, and `Place` with information of the location where the event will happen:

```python
class Event(models.Model):
    name = models.CharField()
    date = models.DateField()
    place = models.ForeignKey("event.Place", on_delete=models.CASCADE)

class Place(models.Model):
    city = models.CharField()
    state = models.CharField(max_length=2)
```

The following code is just an example, but in a real application, it could be a template
where we iterate over a list of events to show them in the HTML response, or some background
task that perform some processing in a set of events. But to test and understand how the fetch
modes works it should be good enough.

```python
events = Event.objects.all()
for event in events:
    print(event.name)
```

Accessing only attributes of `Event` model, one single query is executed by Django:

```sql
SELECT
   "events_event"."id",
   "events_event"."name",
   "events_event"."date",
   "events_event"."place_id"
FROM
   "events_event"
```

The problem starts when we try to access attributes of related objects. If
we need to use one property of the `Place` that relates for each event, depending 
how we create the QuerySet and which field fetch mode configured for the model we 
can have just one or a very large number of distinct SQL queries which causes serious
performance issues, specially when dealing with a large dataset.

Given the following code, where we get the `name` of the `Event`, and the `city`
of the related `Place`, in the next section we can compare the behavior and the
SQL queries generated:

```python
events = Event.objects.all()
for event in events:
    print(event.name, event.place.city)
```

#### FETCH_ONE

```python
events = Event.objects.fetch_mode(models.FETCH_ONE)
for event in events:
    print(event.name, event.place.city)
```

This is the default behavior, like if we are using `.all()`. For each row, the
missing field will be fetched. So if we have for example 100 events,
the previous code will execute 101 SQL queries: one to get all events (showed above) and
for each returned event, another query to retrieve the information of the related place.

```sql
SELECT
    "events_place"."id",
    "events_place"."city",
    "events_place"."state"
FROM
    "events_place"
WHERE
    "events_place"."id" = 6
```

The query above will be execute 100 times, with the `id` on "events_place"."id" = 6` varying according to
the place ID for each event. Even if multiple events share the same place ID, a new query will be executed multiple times.

#### `.select_related()`

The default behavior is the root cause of the well know **N+1 queries problem**. Django
provides a solution if you use [`.select_related()`](https://docs.djangoproject.com/en/6.1/ref/models/querysets/#select-related) method. Instead of multiple 
queries, the ORM will join both tables:

```python
events = Event.objects.select_related("place")
for event in events:
    print(event.name, event.place.city)
```

The query will be:

```sql
SELECT
    "events_event"."id",
    "events_event"."name",
    "events_event"."date",
    "events_event"."place_id",
    "events_place"."id",
    "events_place"."city",
    "events_place"."state"
FROM
    "events_event"
INNER JOIN "events_place"
ON ("events_event"."place_id" = "events_place"."id")
```

This is the proper way to avoid the problem, but we can't forget
to call it when creating the QuerySet. Usually this is a hidden bug: we start the project
accessing only attributes of the main model, and eventually we start using information
from related models and forget to update the QuerySet.

#### FETCH_PEERS

In addition to the default mode, there is the `FETCH_PEERS` mode. If our QuerySet
uses it, we will have the initial query to get all events without joining with the places
table, but in the moment we access one of place's attribute, a new query is executed
to retrieve the missing field for all instances that originally came from the QuerySet.

```python
events = Event.objects.fetch_mode(models.FETCH_PEERS)
for event in events:
    print(event.name, event.place.city)
```

Running the code above will execute the following SQL queries:

```sql
SELECT
    "events_event"."id",
    "events_event"."name",
    "events_event"."date",
    "events_event"."place_id"
FROM
    "events_event";

SELECT
    "events_place"."id",
    "events_place"."city",
    "events_place"."state"
FROM
    "events_place"
WHERE
    ("events_place"."id") IN ((1), (2), (3), (4), (6), (5))
```

It works like a on-demand select related. So instead a "N+1 queries problem", in the worst case
we will have only two queries and we don't need to worry about which field to prefetch.

#### FETCH_RAISE

We also have the `FETCH_RAISE`. In that mode, every time we try to access an attribute not prefetched,
a `FieldFetchBlocked` exception will be raised. In a performance-critical code where any unexpected
query may be a problem, this fetch mode can prevent unintentional queries.

```python
events = Event.objects.fetch_mode(models.FETCH_RAISE)
for event in events:
    print(event.name, event.place.city)
```

Only the first query is executed:

```sql
SELECT
    "events_event"."id",
    "events_event"."name",
    "events_event"."date",
    "events_event"."place_id"
FROM
    "events_event"
```

And when `event.place.city` is needed, the exception is raised:

```python
File .../fetch_modes.py:55, in FetchRaise.fetch(self, fetcher, instance)
     53 klass = instance.__class__.__qualname__
     54 field_name = fetcher.field.name
---> 55 raise FieldFetchBlocked(
        f"Fetching of {klass}.{field_name} blocked.") from None
```

### Changing the default fetch mode

You can change the default fetch for a model class using a custom manager that overrides
`get_queryset()`:

```python
from django.db import models

class EventManager(models.Manager):
    def get_queryset(self):
        return super().get_queryset().fetch_mode(models.FETCH_PEERS)

class Event(models.Model):
    name = models.CharField()
    date = models.DateField()
    place = models.ForeignKey("event.Place", on_delete=models.CASCADE)

    objects = BookManager()
```

### Limitations

**Not all "N+1 queries problems"** can be solved by chosing a proper fetch mode. Fetch modes [only apply](https://docs.djangoproject.com/en/6.1/topics/db/fetch-modes/#fetch-modes) to:

- `ForeignKey` fields

- `OneToOneField` fields and their reverse accessors

- Fields deferred with `QuerySet.defer()` or `QuerySet.only()`

- [Generic relations](https://docs.djangoproject.com/en/6.1/ref/contrib/contenttypes/#generic-relations)

So if you have a `ManyToManyField` like the example:

```python
class Event(models.Model):
    name = models.CharField()
    date = models.DateField()
    place = models.ForeignKey("events.Place", on_delete=models.CASCADE)
    tags = models.ManyToManyField("events.Tag")

class Tag(models.Model):
    value = models.CharField()
```

And we try to access attributes of the related tags:

```python
events = Event.objects.fetch_mode(models.FETCH_PEERS)
for event in events:
    event_tags = [tag for tag in event.tags.all()]
    print(event.name, event_tags)
```

The result is 101 queries (considering that we have 100 events in our database):

```sql
SELECT
    "events_event"."id",
    "events_event"."name",
    "events_event"."date",
    "events_event"."place_id"
FROM
    "events_event";
```

For each event we will have one query as follows:

```sql
SELECT
    "events_tag"."id",
    "events_tag"."value"
FROM
    "events_tag"
INNER JOIN "events_event_tags"
ON ("events_tag"."id" = "events_event_tags"."tag_id")
WHERE "events_event_tags"."event_id" = 1;
```

In that scenario, the solution is to use [`.prefetch_related()`](https://docs.djangoproject.com/en/6.1/ref/models/querysets/#prefetch-related) method, that is similar to [`.select_related()`](https://docs.djangoproject.com/en/6.1/ref/models/querysets/#select-related), but generate 
and extra query for each of the related objects that we access:

```python
events = Event.objects.prefetch_related("tags")
for event in events:
event_tags = [tag for tag in event.tags.all()]
    print(event.name, event_tags)
```

And the queries are reduced to only two:

```sql
SELECT
    "events_event"."id",
    "events_event"."name",
    "events_event"."date",
    "events_event"."place_id"
FROM
    "events_event";

SELECT
    ("events_event_tags"."event_id") AS "_prefetch_related_val_event_id",
    "events_tag"."id",
    "events_tag"."value"
FROM "events_tag"
INNER JOIN "events_event_tags"
ON ("events_tag"."id" = "events_event_tags"."tag_id")
WHERE "events_event_tags"."event_id" IN (...);
```