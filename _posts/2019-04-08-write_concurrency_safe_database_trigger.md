---
layout: post
title: Write concurrency-safe database triggers to validate inter-record constraints
---

> **TL;DR**
> 
> Database triggers usually are not concurrency-safe. The programmers should be aware of this, and be responsible to apply necessary locks to make it safe.

Constraints within a row are easy to validate. Both application-level validation and database constraints can do the job perfectly. SQL\'s `ADD CONSTRAINT CHECK` can describe arbitrary constraints against the current row (it explicitly does not work for other rows). However, constraints across multiple rows are much more trickier:

1. It is hard to check the constraints at the application level when your application is distributed and atomicity cannot be easily guaranteed across application instances.
2. SQL only provides syntax to describe a very limited set of constraints, such as `UNIQUE`, `EXCLUDE`, `FOREIGN KEY`, etc. It is inadequate to describe arbitrary constraints for application semantics.

Many think database triggers straightly solve the problem: execute your SQL function to validate the constraints as a trigger. As the trigger is executed in the same transaction with the insertion or update, atomicity and consistency is \'automatically guaranteed\'. I had thought so, until I found incorrect data passed though my validation trigger and saved in the database. In the course of resolving the issue, I came across [incorrect Stackoverflow answers](https://gis.stackexchange.com/questions/270769/enforce-postgis-constraint-that-geometries-do-not-overlap), [unattended questions](https://www.postgresql.org/message-id/22881336.post@talk.nabble.com), and [text book sections](http://iips.icci.edu.iq/images/exam/databases-ramaz.pdf) that discuss triggers without mentioning concurrency and isolation level. I realised that this might be something that many people misunderstood about, and it is worthy to put my findings together as a thorough discussion. (Most of this post is based on PostgreSQL. )

### Are database trigger concurrency-safe?

A short answer: No. 
A more precise answer: it depends on your transaction isolation level.

In the SQL standard, the most restrict isolation level is `SERIALIZABLE`, at which level all transactions run as if there is only one transaction at a time. As database triggers run in the same transaction as the event, when a database trigger is being executed, there is no other active transaction to violate atomicity. Thus, database triggers are concurrency-safe with the `SERIALIZABLE` isolation level.

However, most DBMS do not use `SERIALIZABLE` as default, plus it is not recommended to use `SERIALIZABLE` in production. PostgreSQL is defaulted to `READ COMMITTED`. A weaker isolation level allows better performance, but it also allows certain data anomalies. For example, `READ COMMITTED` [allows non-repeatable reads](https://www.postgresql.org/docs/current/transaction-iso.html), that is saying, repeating reads of the same data in one transaction may have different results.

Let\'s use [the Stackoverflow question](https://gis.stackexchange.com/questions/270769/enforce-postgis-constraint-that-geometries-do-not-overlap) mentioned above as an example. We want to validate the constraint that geometries in a table do not overlap with each other. We add a `BEFORE INSERT OR UPDATE` trigger to execute a SQL function to check if any existing geometries overlap with the new row; abort if overlapping. The trigger code is shown as below:

```
<pre class="EnlighterJSRAW" data-enlighter-group="" data-enlighter-highlight="" data-enlighter-language="sql" data-enlighter-linenumbers="" data-enlighter-lineoffset="" data-enlighter-theme="" data-enlighter-title="">CREATE OR REPLACE FUNCTION overlap_check() RETURNS TRIGGER
    LANGUAGE plpgsql
    AS $_$
DECLARE
    overlap_ids int[];
BEGIN
    SELECT array_agg(id) INTO overlap_ids FROM my_table WHERE
        id <> NEW.id
        AND geom && NEW.geom;
    IF cardinality(overlap_ids) > 0 THEN
        RAISE EXCEPTION 'OVERLAP: Found % overlapping with new record', overlap_ids;
    END IF;
    RETURN NEW;
END;
$_$;

CREATE TRIGGER overlap_trigger
    BEFORE INSERT OR UPDATE ON my_table
    FOR EACH ROW
    EXECUTE PROCEDURE overlap_check();
```

The trigger is invoked within the same transaction before an insertion. Considering the transaction implication, the pseudo code looks like this:

```
<pre class="EnlighterJSRAW" data-enlighter-group="" data-enlighter-highlight="" data-enlighter-language="sql" data-enlighter-linenumbers="true" data-enlighter-lineoffset="" data-enlighter-theme="" data-enlighter-title="">START TRANSACTION
-- trigger function starts
-- S1:
SELECT array_agg(id) INTO overlap_ids FROM my_table WHERE
    id <> NEW.id
    AND geom && NEW.geom;
IF cardinality(overlap_ids) > 0 THEN
    RAISE EXCEPTION 'OVERLAP: Found % overlapping with new record', overlap_ids;
END IF;
-- trigger function ends, we can insert the row
-- S2:
INSERT INTO my_table () VALUES (new row);
END TRANSACTION
```

S1 reads data to check if there are overlapping geometries. If there is no overlap, S2 inserts the data. These all happen in the same transaction. However, assuming the database runs with `READ COMMITT` that allows \'non-repeatable reads\', at S2, we can no longer assume our check at S1 still holds true. The isolation level simply does not guarantee repeatable reads through a transaction: other processes may insert geometries between S1 and S2, and they may possibly overlap with the new row here. But the insertion at S2 will still insert the new row without knowing the constraint is already broken.

**Weaker isolation level puts the concurrency-safety back to the programmers\' hands.**

### How to make triggers safe?

Knowing there might be race conditions, our question is: how can we make triggers concurrency-safe? Here I will present four different approaches that work in terms of correctness, and discuss their pros and cons.

#### Option 0: Rephrase your trigger to `UNIQUE` and `EXCLUDE`

This technically is not an approach for implementing triggers. However, before diving into triggers and managing concurrency by ourselves, I think it is worthwhile to try match your constraints\' semantics into the SQL standard `UNIQUE` and `EXCLUDE`. For example, the non-overlapping geometries constraint can be expressed with `EXCLUDE`:

```
<pre class="EnlighterJSRAW" data-enlighter-group="" data-enlighter-highlight="" data-enlighter-language="sql" data-enlighter-linenumbers="" data-enlighter-lineoffset="" data-enlighter-theme="" data-enlighter-title="">ALTER TABLE my_table 
ADD CONSTRAINT my_table_no_overlapping_geometry 
EXCLUDE USING GIST (
  -- if we want to make sure that only geometries within a user do not overlap, include the following line. We will discuss this example later. 
  -- user_id WITH =,
  geom WITH &&
);
```

I would strongly recommend reading through [the manual](https://www.postgresql.org/docs/current/ddl-constraints.html) about `UNIQUE` and `EXCLUDE`. Though they cannot express arbitrary constraints, they can handle some constraints across multiple rows with certain conditions and may be more powerful than you thought. However, if they are not powerful enough for your case, let\'s talk about triggers.

#### Option 1:`SERIALIZABLE` transaction

As we have discussed, the concurrency problem is that the isolation level allows certain kinds of data anomalies, such as \'unrepeatable reads\'. The easy way to fix it is to change the isolation level to `SERIALIZABLE`. One can [change the default isolation level to `SERIALIZABLE`](https://dba.stackexchange.com/questions/62024/how-to-set-isolation-to-serializable-deferrable-for-a-whole-postgresql-database) for the database, or use `SET TRANSACTION` to set isolation level per transactions. The latter has a smaller performance impact, as it only treats transactions that you care with the `SERIALIZABLE` level.

In practice, one issue arising from this approach makes it less ideal. As real `SERIALIZABLE` is too restrict to handle high throughput, most databases make assumptions about the `SERIALIZABLE` level. On PostgreSQL, `SERIALIZABLE` works exactly the same as `REPEATABLE READS`, except that in the case where the properties of `SERIALIZABLE` will break, the transactions will be simply aborted to make sure all succeeding transactions are still compliant with `SERIALIZABLE`(read [here](https://www.postgresql.org/docs/current/transaction-iso.html)). So a `SERIALIZABLE` transaction may arbitrarily fail in concurrent cases and an application-level retry mechanism is required to handle the failed cases. The retry time can be unbounded as there is no guarantee that the transaction will succeed.

For this reason, `SERIALIZABLE` is generally not recommended to handle the concurrency-safety here. It is preferable to handle the concurrency within the database, instead of on the application level.

#### Option 2: Explicit table lock

As said earlier, it would be good if we can handle the concurrency safety within the database. PostgreSQL uses implicit locks to implement isolation levels, and it allows a `LOCK TABLE` statement to explicitly acquire locks. We could use explicit table locks to give us the concurrency guarantee we want. There is a little background we need to fill in before diving into the solution.

- When executing most SQL statements, the database will acquire corresponding locks to make sure the concurrent behaviours match what the isolation level requires. For example,`SELECT` will acquire a `ACCESS SHARE` lock, and `UPDATE` will acquire a `ROW EXCLUSIVE` lock. Do not worry too much about the lock names; there are some historical reasons for the names, and the names do not really tell what they actually mean.
- A lock mode conflicts with a certain set of lock modes. The exclusiveness definition can be found [here](https://www.postgresql.org/docs/current/explicit-locking.html). A lock mode is not necessarily self-conflicting. Generally some locks are \'stronger\' - it is exclusive with more lock modes.
- A lock needs to be granted before the statement can be executed. If other process holds the hold, the current process will be blocked and waiting for the lock release.
- Once a lock is granted, the process holds it until the current transaction finishes (either succeeds or fails). There is no explicit release.
- Lock contention only happens between different processes. If a process holds a lock, and it tries to acquire a new lock that is exclusive to the existing lock, the acquiring can still be successful, as it won\'t be self-conflicting. Thus two locks will be held by the process (though one is \'stronger\' than the other).

Knowing this background, let\'s get back to the SQL pseudo code earlier, and add the locking behaviour to the pseudo code.

```
<pre class="EnlighterJSRAW" data-enlighter-group="" data-enlighter-highlight="" data-enlighter-language="sql" data-enlighter-linenumbers="" data-enlighter-lineoffset="" data-enlighter-theme="" data-enlighter-title="">START TRANSACTION
-- L1:
IMPLICIT LOCK for INSERT - ROW EXCLUSIVE LOCK
-- trigger function starts
-- L2:
IMPLICIT LOCK for SELECT - SHARE LOCK
-- S1:
SELECT array_agg(id) INTO overlap_ids FROM my_table WHERE
    id <> NEW.id
    AND geom && NEW.geom;
IF cardinality(overlap_ids) > 0 THEN
    RAISE EXCEPTION 'OVERLAP: Found % overlapping with new record', overlap_ids;
END IF;
-- trigger function ends, we can insert the row
-- S2:
INSERT INTO my_table () VALUES (new row);
END TRANSACTION
-- L1 and L2 are released
```

As we are executing an insertion, at the beginning of the transaction, PostgreSQL needs to acquire lock L1 for insertion. Then the trigger function gets executed, and lock L2 is granted for the select statement. There are a few things we need to note:

- L1 happens at the beginning of the transaction instead of just before S2. This can be observed by opening two Postgres clients, doing the transactions manually and observing the `pg_locks` table (an example could be found [here](https://www.compose.com/articles/common-misconceptions-about-locking-in-postgresql/)).
- L2 is a `SHARE` lock, and it does not conflict with L1 or self. That means other processes can do insertion while this process holds L1, and this is where the \'unrepeatable reads\' behaviour comes from.
- L1 is a `ROW EXCLUSIVE` lock, and not self-conflicting. That means two processes can holds this lock, and do insertion at the same time. This is why the trigger attempt below will cause deadlocks.

Understanding these, we can again look at the [unattended mail list question](https://www.postgresql.org/message-id/22881336.post@talk.nabble.com): why using explicit lock in a trigger function will cause deadlocks. Let\'s further add the explicit lock L0 to the pseudo code as the question.

```
<pre class="EnlighterJSRAW" data-enlighter-group="" data-enlighter-highlight="" data-enlighter-language="sql" data-enlighter-linenumbers="" data-enlighter-lineoffset="" data-enlighter-theme="" data-enlighter-title="">START TRANSACTION
-- L1:
IMPLICIT LOCK for INSERT - ROW EXCLUSIVE LOCK
-- trigger function starts
-- L0:
EXPLICIT LOCK - X LOCK
-- L2:
IMPLICIT LOCK for SELECT - SHARE LOCK
-- S1:
SELECT * FROM my_table WHERE existing geoms overlap the new row;
IF FOUND THEN
  RAISE EXCEPTION;
END IF;
-- trigger function ends, we can insert the row
-- S2:
INSERT INTO my_table () values (new row);
END TRANSACTION
```

We wanted to introduce the explicit lock L0 so that the condition we verified at S1 stays true at S2, which means the lock mode X needs to conflict with the insertion lock L1 (\'stronger\' than L1). However, this will cause deadlocks: two processes may both have acquired L1 (as its not self-conflicting), and attempts to acquire L0. But L0 conflicts with the other process\' L1, so both processes cannot proceed.

Fixing the problem is straight-forward: in the application code, start the transaction manually, then immediately acquire L0 lock, and do the insertion which will execute the trigger after L0 lock. This works correctly and cleanly. The code is shown as below.

```
<pre class="EnlighterJSRAW" data-enlighter-group="" data-enlighter-highlight="" data-enlighter-language="sql" data-enlighter-linenumbers="" data-enlighter-lineoffset="" data-enlighter-theme="" data-enlighter-title="">BEGIN;
-- L0: EXPLICIT LOCK, the lock mode needs to be stronger than L1 ROW EXCLUSIVE, e.g. use SHARE ROW EXCLUSIVE
LOCK TABLE my_table IN SHARE ROW EXCLUSIVE LOCK;
-- L1: IMPLICIT LOCK for INSERT - ROW EXCLUSIVE LOCK
-- L2: IMPLICIT LOCK for SELECT (in the trigger) - SHARE LOCK
INSERT INTO my_table () VALUES (new row);
END TRANSACTION
```

#### Option 3: Finer grained advisory lock

The above approach works fine. But it has performance implication. L0 acquired by `LOCK TABLE` is a table-level lock. It locks the whole table so that only one process can do the insertion transaction. In this example, it works exactly as we expect - we want to lock the whole table to check geometries overlapping.

However, let\'s say, our requirement is a little bit different: instead of making sure that geometries in the whole table do not overlap, we put geometries for different users in the table, and we want to make sure that geometries for one user do not overlap (different users may have overlapping geometries). That is saying when inserting geometries for a user, we only need to check part of the table that are related with the user. If we still use the above approach, a table-level lock from `LOCK TABLE` locks the entire table and prevents possible concurrency of inserting geometries of different users at the same time, which could become a performance bottleneck.

PostgreSQL provides an application-defined lock called [advisory lock](https://www.postgresql.org/docs/current/explicit-locking.html#ADVISORY-LOCKS). `pg_advisory_lock()` can take any integer, and apply locking on the integer (we can also lock on strings by hashing strings into ints with `hashtext(text)`). Advisory locks are held until the end of the transaction as table locks, but they can be released earlier with `pg_advisory_unlock()` which allows more precise control from the programmer. With advisory locks, in our case, we can apply advisory lock with the user\'s id to prevent concurrent operations only within a user\'s data. With advisory lock, our trigger function would look like this:

```
<pre class="EnlighterJSRAW" data-enlighter-group="" data-enlighter-highlight="" data-enlighter-language="sql" data-enlighter-linenumbers="" data-enlighter-lineoffset="" data-enlighter-theme="" data-enlighter-title="">CREATE OR REPLACE FUNCTION overlap_check() RETURNS TRIGGER
    LANGUAGE plpgsql
    AS $_$
DECLARE
    overlap_ids int[];
BEGIN
    -- app-defined advisory lock only locking on user_id
    -- the lock will be released at the end of the transaction
    PERFORM pg_advisory_lock(NEW.user_id);
    SELECT array_agg(id) INTO overlap_ids FROM my_table WHERE
        id <> NEW.id
        AND geom && NEW.geom;
    IF cardinality(overlap_ids) > 0 THEN
        RAISE EXCEPTION 'OVERLAP: Found % overlapping with new record', overlap_ids;
    END IF;
    RETURN NEW;
END;
$_$;
```

#### Option 4: Finer grained implicit lock with UPSERT 

This is quite an interesting approach, as an answer [here](https://dba.stackexchange.com/questions/202775/how-to-write-validation-trigger-which-works-with-all-isolation-levels). For our example, we create an extra helper table `trigger_helper`,

```
<pre class="EnlighterJSRAW" data-enlighter-group="" data-enlighter-highlight="" data-enlighter-language="sql" data-enlighter-linenumbers="" data-enlighter-lineoffset="" data-enlighter-theme="" data-enlighter-title="">CREATE TABLE trigger_helper (val int PRIMARY KEY);
```

and in our trigger function, we perform an `UPSERT` about the user id to the `trigger_helper` table. The pseudo code is as below.

```
<pre class="EnlighterJSRAW" data-enlighter-group="" data-enlighter-highlight="" data-enlighter-language="sql" data-enlighter-linenumbers="" data-enlighter-lineoffset="" data-enlighter-theme="" data-enlighter-title="">CREATE OR REPLACE FUNCTION overlap_check() RETURNS TRIGGER
    LANGUAGE plpgsql
    AS $_$
DECLARE
    overlap_ids int[];
BEGIN
    -- UPSERT with implicit lock on the trigger_helper table
    INSERT INTO trigger_helper (val)
        VALUES (NEW.user_id)
        ON CONFLICT (val) DO UPDATE
        SET val = NEW.user_id
        WHERE FALSE;
    SELECT array_agg(id) INTO overlap_ids FROM my_table WHERE
        id <> NEW.id
        AND geom && NEW.geom;
    IF cardinality(overlap_ids) > 0 THEN
        RAISE EXCEPTION 'OVERLAP: Found % overlapping with new record', overlap_ids;
    END IF;
    RETURN NEW;
END;
$_$;
```

An `UPSERT` (`INSERT` with `ON CONFLICT DO UPDATE` clause) may insert a value into the table, or may update the value if it exists. The same as other SQL statements, `UPSERT` implicitly acquires a lock to make sure the correctness, and the lock holds until the transaction is done. This implicit lock is row-based, which means for different user ids, we will lock on different rows in the `trigger_helper` table so that their trigger functions and insertions can run in parallel.

The original Stackexchange answer stated that this implicit lock is \'much cheaper\' (than a table-level lock, such as Option 2), and \'less intrusive\'. However, it is unclear whether this approach is better than using advisory locks. AliCloud has a few [posts](https://yq.aliyun.com/articles/137052) (in Chinese) about using Postgres\' advisory lock for high concurrency and high throughput scenarios, and showed that proper use of advisory locks can achieve better performance than table-level lock as well.

### Summary

Triggers are a powerful tool to implement arbitrary constraints. However, they are not concurrency safe as many people may assume. We discussed why they are not concurrency safe, and how isolation level affects the concurrent execution. We discussed four different approaches to make trigger concurrency-safe. Using `SERIALIZABLE` transaction is the easiest fix, but it requires application-level retry and may incur unbounded waiting time before a transaction succeeds. We discussed about Postgres\' locks with regards to SQL statements, and how we can use explicit table lock to make triggers safe. However, table locks are coarse grained, and we further discussed two approaches with finer grained lock - the triggers are concurrency safe without the need to lock up the whole table. I reckon both advisory locks and `UPSERT` locks are versatile and performant to handle most validation triggers.