+++
title = "Writing ON CONFLICT Clauses on Partial Indexes using GORM"
date = "2025-08-21T02:35:04+03:00"
keywords = ['go', 'gorm', 'postgres', 'sql']
description = """A short tale of caution on keeping your dependencies up-to-date."""
showFullContent = false
readingTime = false
hideComments = true
toc = true
+++

## Introduction

PostgreSQL's `ON CONFLICT` clause is a useful construct that allows you to 
handle cases where an insertion may violate a uniqueness constraint. It is 
not standard SQL syntax, but an extension, like SQLite's `INSERT OR REPLACE`.

Let's say we have a table named `tasks` like this:

```sql
create table if not exists tasks
(
    account_id integer not null,
    task_name  text    not null,
    task_desc  text    not null
);
-- Yes, I prefer lowercase SQL. Sue me
```

Say we want tasks to have unique names for each account. Let's define a unique index for it:

```sql
create unique index if not exists unq_tasks_index
    on tasks (account_id, task_name);
```

Let's say we need to perform an upsert operation on this table.
Maybe we have a wacky endpoint that either rewrites the description of 
an existing task, or creates one anew. I don't know; I just want to get to the point.

To **upsert**; we can write the following query:

```sql
insert into tasks (account_id, task_name, task_desc)
values (1, 'test task', 'This is a test task')
on conflict (account_id, task_name)
do update set task_desc=excluded.task_desc;
```

This does one of two things, atomically:
- If an entry with the same `(account_id, task_name)` pair exists, 
  it simply _updates_ the `task_desc` field.
- If such an entry does not exist, it _inserts_ it.

`excluded` is a keyword here; referring to the row that is newly being inserted.
(Perhaps `proposed` or `candidate` would be better names, but what do I know?)

> We can also insert multiple rows this way:
> ```sql
> insert into tasks (account_id, task_name, task_desc)
> values (1, 'test task', 'This is a test task'),
>        (1, 'test task 2', 'Another task')
> on conflict (account_id, task_name)
> do update set task_desc=excluded.task_desc;
> ```
> But there is an important edge case here! If the proposed rows conflict ***with each other***, then you get the following error:
> ```
> [21000] ERROR: ON CONFLICT DO UPDATE command cannot affect row a second time
> ```
> This behavior is documented [here](https://www.postgresql.org/docs/current/sql-insert.html#SQL-ON-CONFLICT):
> ```
> INSERT with an ON CONFLICT DO UPDATE clause is a "deterministic" statement.
> This means that the command will not be allowed to affect any single existing
> row more than once; a cardinality violation error will be raised when this 
> situation arises.
> ```
> Be careful if you're using this variant!

Raw query is fine and all, but if we wanted to achieve the same behavior using GORM,
we can use `clauses.OnConflict` [like this](https://gorm.io/docs/create.html#Upsert-On-Conflict):

```go
type Task struct {
    AccountID uint
    TaskName, TaskDesc string
}

task := Task{1, "test task", "This is a test task"}

db.Clauses(clause.OnConflict{
    Columns: []clause.Column{{Name: "account_id"}, {Name: "task_name"}},
    DoUpdates: clause.AssignmentColumns([]string{"task_desc"}),
}).Create(&task)
```

## Problem

What if we wanted soft-delete functionality for this table? We'd need a column like `is_deleted`, or `deleted_at`:

```sql
create table tasks
(
    account_id integer not null,
    task_name  text    not null,
    task_desc  text    not null,
    -- added
    deleted_at timestamp with time zone
);
```

That's easy, but now our unique index does not make sense.
Now if we create a task named `abc`, delete it, then create a new one;
the newly created entry will conflict with the deleted one!
We should keep that from happening using a _partial index_:

```sql
create unique index if not exists unq_tasks_index
    on tasks (account_id, task_name)
    -- added
    where deleted_at is null;
```

Now uniqueness checks are performed only among non-deleted rows. Cool.

We should also modify our query. We add the `deleted_at` column to 
the column list, corresponding values, and the update set:
```sql
insert into tasks (account_id, task_name, task_desc, deleted_at)
values (1, 'test task', 'This is a test task', null)
on conflict (account_id, task_name)
do update set
    task_desc=excluded.task_desc,
    deleted_at=excluded.deleted_at;
```
After this, converting it to GORM is trivi-
```
[42P10] ERROR: there is no unique or exclusion constraint matching the ON CONFLICT specification
```
Oh. We have an error. Of course.

Now that we have a _partial_ index; our conflict condition does not match an existing index.

Fortunately, we can pass a `where` clause as an
[index predicate](https://www.postgresql.org/docs/current/sql-insert.html#SQL-ON-CONFLICT):

```sql
insert into tasks (account_id, task_name, task_desc, deleted_at)
values (1, 'test task', 'This is a test task', null)
on conflict (account_id, task_name)
    -- added
    where deleted_at is null
do update set
    task_desc=excluded.task_desc,
    deleted_at=excluded.deleted_at;
```
Then the partial index is properly inferred, and the query works as intended.

Converting this to GORM _is_ trivial:

```go
type Task struct {
    AccountID uint
    TaskName, TaskDesc string
    DeletedAt time.Time
}

task := Task{AccountID: 1, TaskName: "test task", TaskDesc: "This is a test task"}

db.Clauses(clause.OnConflict{
    Columns: []clause.Column{{Name: "account_id"}, {Name: "task_name"}},
    DoUpdates: clause.AssignmentColumns([]string{"task_desc"}),
    // added
    TargetWhere: clause.Where{Exprs: []clause.Expression{
      clause.Eq{Column: "deleted_at", Value: nil},
    }},
}).Create(&task)
```

The `TargetWhere` field of `clauses.OnConflict` lets us provide the index predicate here.

> Do not confuse it with the `Where` field, which is a regular 
> `WHERE` that goes at the _end_ of the clause.

We can perform atomic upsert operations on a table with a partial index using GORM like this.

## Why did I write a blog post on this?

Because the `TargetWhere` field does not exist in GORM version v1.21.3, which is what I had to use in a 
project due to, uhh, reasons. Mind that the latest version is > v1.30; and the problem has long been 
addressed with [this commit](https://github.com/go-gorm/gorm/commit/dd8bf88eb9abdac71a290222ee2f70cf293c662b),
**four years ago** (The joys of enterprise software).

I ended up writing a temporary duplicate for the `OnConflict` clause like this,
along with a TODO message strongly suggesting we update our dependencies. 

Note that I used the same field names as `clauses.OnConflict` in v1.30.1, 
so I can simply replace this with the original clause once our dependency is upgraded.
This version is also missing other unused fields, and some value checks etc. for brevity:

```go
type PartialOnConflict struct {
    Columns     []clause.Column
    TargetWhere clause.Where
    DoUpdates   clause.Set
}

func (c PartialOnConflict) Build(b clause.Builder) {
    b.WriteByte('(')
    for i, e := range c.Columns {
        if i > 0 {
            b.WriteByte(',')
        }
        b.WriteQuoted(e)
    }
    b.WriteString(") WHERE ")
    c.TargetWhere.Build(b)
    b.WriteString(" DO UPDATE SET ")
    c.DoUpdates.Build(b)
}

func (PartialOnConflict) Name() string { return "ON CONFLICT" }

func (c PartialOnConflict) MergeClause(m *clause.Clause) { m.Expression = c }
```

As far as I could see, the issue and the corresponding fix in GORM was not explicitly documented
in the changelog, but I might have missed it too. If you found yourself in a similar situation,
I hope this blog post explains the situation, and encourages you to keep your dependencies up-to-date.

The rest of you, I wish you productive days. Go now.
