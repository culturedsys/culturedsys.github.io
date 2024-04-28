---
layout: post
title: Avoiding the soft delete anti-pattern
---

Programmers hate deleting things; we've all felt that feeling in the pit our stomach when we realise
that thing we deleted was *really* deleted, and on the other hand the security of deleting some 
unused code, safe in the knowledge that it's still really there in version control. In the sphere of 
databases, this terror of deleting things leads people to advocate *soft deletion*: instead of 
really deleting a record, you add a field which marks the record as deleted, and you treat any
record marked in that way as if it were deleted. This is generally a bad idea, and there are a 
number of better ways of ensuring access to old data. <!-- more -->

The main problem with soft deletion is that you're systematically misleading the database. This is
most obvious in the case of foreign key constraints. If it's important to your data integrity that
some column in one table references a row that exists in another table, a foreign key constraint
instructs the database to enforce that by, among other things, preventing the deletion. A similar 
issue arises with unique constraints: conceptually, uniqueness probably shouldn't consider deleted
items (say a user deletes their account, and later wants to sign up again with the same email 
address), but the database doesn't know that. You also face difficulties with modifying the database
structure, as you have to consider, not just how these modification affect live data, but how it 
would affect historical data, where the change might not make sense. You can mitigate some of these 
problems by carefully crafting triggers and constraints to take into account the deletion marker, 
but that adds significant complexity.

Because you're misleading the database, you also encounter problems with querying the data. You can
no longer rely on `SELECT`s to only return live data; you have to remember to check the deletion
marker in each `WHERE` clause (and in each `JOIN`; I nearly forgot about that while writing this, a
reflection of the many times I've forgotten about it while implementing this). You can potentially
avoid this by creating views that filter out the deleted rows, or you can outsource the problem to
an ORM (with something like Hibernate's `@Where` annotation which can add the deletion marker to
its constructed queries). But these are fragile and, especially for the ORM case, leaky 
abstractions; it's easy to make a mistake and end up querying soft deleted data, or for the ORM to
make a mistake and try and query data which it then fails to find because it thinks its deleted.

Of course, the extra complexity introduced by soft deletion isn't necessarily a reason to avoid it;
perhaps it's a price worth paying in the noble quest to never lose data. But it's worth thinking 
about how much soft deletion actually helps us here. The thing is, in the common case of a database
that's the live storage for an application, we remove data all the time --- not by deleting it, but
by overwriting it. Here, the database represents the *current* state of the application, not an 
archive (for something like a data warehouse, which is frequently added to but less frequently 
mutated, different rules apply, and is one of the cases where my "generally" above might not apply). So
soft deletion, as well as being complex, might be insufficient. Soft deletion is often touted as a
"best practice" and implemented without much consideration; but we might do better to consider 
exactly what data permenance we want to achieve, and adopt an approach which more precisely meets
those needs.

## Alternative 1: YOLO

I think it's worth giving serious consideration to the simplest option: just hard delete. There
are two reasons this might be a better idea than it sounds. The first is that, with or without soft
deletion, deleting data needs to be taken seriously. Although soft deletion allows you to restore
deleted data, it's still likely to have significant consequences: delete a user and they can't log
in, and there may be knock-on effects which are not so easy to rectify (for instance, perhaps users
are charged a subscription fee on a regular basis; while the user is deleted, they won't be charged,
and just restoring the user record won't retroactively apply these charges). We have to take serious
care to avoid incorrect deletes, and if we are taking that care, we have less need for the safety 
provided by soft deletes. The second reason is that, if mistakes do happen, with modern databases, 
even hard deletion isn't really hard deletion --- the database stores a log of changes, which means 
you can replay the changes and restore a snapshot for disaster recovery. Now, this rollback ability 
isn't usually unlimited (and even if it is, it's not an entirely trivial process), so after some 
point the data really will be lost, so this approach won't work if you need to maintain a historical 
record for an extended period of time.

## Alternative 2: Lifecycle

For entities that have a lifecycle, one option is to consider end-of-life as part of that lifecycle.
So, as well as having states for the various points of active processing, you could have an
archived or out-of-use state. This might seem merely a variant of a deletion marker, but the 
important difference is that it's explicitly represented at the application level. Generally, any
operation on the entity will need to consider its state, and so will naturally consider the archived
state, as opposed to the deletion marker which is an extraneous detail. This only really makes sense
for entities which naturally have a lifecycle (introducing a state solely to allow an archived 
state really is just the same as a deletion marker), and it doesn't entirely do away with the 
problem of modifying the database structure (although, as there is at least some semantic meaning to
the archived state, it may be a little easier to provide a reason for how a change would affect
archived data).

## Alternative 3: Temporal tables

The previous approach involves moving deletion from an irritating persistence detail to a first-
class part of the application logic. The mirror image is to instead make it a first-class feature 
of the database. Some databases support [temporal tables](https://mariadb.com/resources/blog/temporal-tables-part-1/), 
where every change has a timestamp associated with it and, rather than modifying existing data, 
a new change with a new timestamp is recorded. By default, queries operate on the data as it is at
the current time, but an explicit time (or range) can be specified to access historical data. This
is a good choice where a complete historical record is required, but it does have the disadvantage
of, well, keeping a complete historical record (the space for, and the need to query over, the
whole history; some of this can be mitigated by partitioning the historical records separately from
the current ones).

## Alternative 4: Delete them all and let the data warehouse sort it out

As I said above, problems with soft deletion apply primarily to databases containing an 
application's current state. It's increasingly common for an application to have multiple databases
(one database per microservice, for instance), and multiple other data sources (3rd-party analytics
services, for example), raising the need for some further data store (a data lake and/or warehouse)
to integrate these multiple data sources. You typically want a data warehouse to surface historical
data anyway for reporting purposes, so it makes sense to outsource your historical record keeping to
this data infrastructure; in that case, you can delete from the application's own data store with
impunity. This does require the overhead of setting up this data infrastructure, and of ensuring 
that your application robustly sends its data to that infrastructure; but you're likely to want to
do this for other reasons, and if you do, you can take advantage of that to liberate yourself from
the complexity of soft deletion.