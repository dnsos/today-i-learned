---
layout: post
title: "How to enforce exclusive time ranges in PostgreSQL"
subtitle: "Using an example of booking rooms for hotel rooms"
tags:
  - "PostgreSQL"
  - "Constraints"
---

Let's approach this issue with an example that is near to reality.

Say we are managing the database of a hotel. Each room is stored in the database as a record in the `rooms` table:

```sql
CREATE TABLE rooms (
  id SERIAL PRIMARY KEY
  -- ... and other columns that are not relevant for this example
);
```

Each room can have `occupancies` which reference a room.

```sql
CREATE TABLE occupancies (
  id SERIAL PRIMARY KEY,
  room_id INTEGER REFERENCES rooms (id)
  -- ...
);
```

Now our `room_id` references a record in our `rooms` table. But we still need to define the duration of the occupancy. We could do this with timestamp columns  something like `started_at` and `terminated_at`. But it would get quite tricky to enforce the exclusivity of time ranges on a per-room basis like this.

Before diving into the SQL, let's first describe the necessary constraint in English. We want to ensure that each room can only be occupied by one record in the `occupancies` table **at the same time**. That means the room with the id `1` could receive an occupancy starting at `2023-01-14 18:00:00` and ending at `2023-01-15 09:00:00` the next morning. We assume that these are the times for checkin and checkout. We want to make sure that no other occupancy for room `1` can be started during this time range, while other _available_ rooms can become occupied, obviously.

In addition, we want to make sure that a room can have a start time for an occupancy, but an undefined end time (let's say this is possible in our hotel.) We need to find a way to prevent occupancies from being added if a room has an occupancy with a start time but a `NULL` value for the end time. Here's where PostgreSQL's time ranges come into play.

## A table that holds exclusive time ranges for each room

First things first, we want to enable the `btree_gist` extension. This is necessary for specifying our exclusion constraint in a moment.

```sql
CREATE EXTENSION btree_gist;
```

Then we can define our table:

```sql
CREATE TABLE occupancies (
  id SERIAL PRIMARY KEY,
  room_id INTEGER REFERENCES rooms (id),
  -- The duration will be a range of timestamps (without time zone):
  duration TSRANGE,
  -- Below we constrain records so that no overlaps of ranges can
  -- occur (per room):
  EXCLUDE USING GIST (room_id WITH =, duration WITH &&)
);
```

> PostgreSQL's documentation describes [constraints on ranges](https://www.postgresql.org/docs/current/rangetypes.html#RANGETYPES-CONSTRAINT) wonderfully.

## Inserting occupancies

Let's insert one terminated and one ongoing occupancy for the same room.

```sql
INSERT INTO occupancies (room_id, duration)
VALUES (
  1,
  -- A terminated time range:
  '[2023-01-14 18:00:00, 2023-01-15 09:00:00)'
);

INSERT INTO occupancies (room_id, duration)
VALUES (
  1,
  -- The 'upper' part of the range is NULL, which means that currently
  -- no new record can be added for this room. Only when we terminate
  -- the time range will this become possible again:
  '[2023-01-15 10:00:00,)'
);
```

> Note the `[` and `)` at the beginning and the end of the time ranges. This is no typo. Instead it defines the [inclusivity and exclusivity of the time range's bounds](https://www.postgresql.org/docs/current/rangetypes.html#RANGETYPES-INCLUSIVITY). In our case we want to make sure that the start time of our occupancy is included (`[`) and the end time is excluded (`)`), making it possible to start off a new time range at the same time a previous range has been terminated.

## Terminating an occupancy

The last occupancy we inserted has no value for the end time. We can terminate the occupancy by using the following SQL statement based on [this example for partially updating a range](https://gist.github.com/karanlyons/3af10ad9a90dbbd02a6b):

```sql
UPDATE occupancies
SET duration = TSRANGE(lower(duration), NOW()::TIMESTAMP, CONCAT(
            CASE WHEN lower_inc(duration) THEN '[' ELSE '(' END,
            CASE WHEN upper_inc(duration) THEN ']' ELSE ')' END
        ))
WHERE id = 4; -- or whichever id the occupancy has.
```

Note how we grab the already existing start time from the `duration` (via `lower(duration)`) to construct a new _terminated_ time range.

## Filtering rooms that are available

Using this table setup, we can easily filter the rooms that are currently available i.e. where the upper bound of the time range `IS NOT NULL`:

```sql
-- We only select the distinct room id's in this case:
SELECT DISTINCT rooms.id
FROM rooms
JOIN occupancies
  ON occupancies.room_id = rooms.id
WHERE upper(occupancies.duration) IS NOT NULL;
```

This way we get a pretty robust system for managing the occupancies and bookings of our rooms.

> Note: These solutions were lasted tested with PostgreSQL version 14.
