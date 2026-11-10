# Database steps (beta)

A step can check what the database holds, or seed the data a scenario
starts from, with no query written by hand. While recording, the agent
reads the schema, writes one parameterized statement, runs it, and records
it with the step's values as binds; every later run replays it at zero
tokens. They work in browser scenarios and in `@api` scenarios alike.
PostgreSQL, MySQL and SQL Server, named by the URL's scheme
(`postgres://`, `mysql://`, `mssql://`).

## Setup

```json
{
  "db": { "url": "{env:DATABASE_URL}", "writes": false, "allow": ["localhost", "shop_test"], "timeout": 3000, "cleanup": "all" }
}
```

- `npm install node-sql-parser` and the engine's driver (`pg`, `mysql2` or
  `mssql`) in the project. Both load only when `db` is set.
- Every engine takes `$1`, `$2` binds; write the engine's own SQL
  otherwise. On MySQL and SQL Server an `INSERT` with an auto-numbered key
  writes one row (several rows give the key as binds), and an `UPDATE`
  never sets the key: Saffron reads the rows back by it. A recording is
  refused on another engine.
- `url` must be an `{env:...}` reference: the connection string and its
  password never sit in a file, and never reach a recording, a proposal,
  a report, a trace or the agent. Write the host and the database in the
  URL itself; a `?host=` parameter, or no host, is refused for writes.
- `writes` (default `false`) lets `Given` and `When` steps seed data.
- `allow` lists the hosts or database names a write may go to. A write
  anywhere else is refused before it runs.
- `timeout` (milliseconds, default 3000) is how long a check reads again.
- `cleanup` (default `"all"`) is what is undone when a scenario ends.
- Give Saffron its own database user on a test database.

## Wording the steps

```gherkin
Scenario: A new customer is saved
  Given a customer "ann-{unique:c}@example.test" on the "basic" tier
  When I upgrade "ann-{unique:c}@example.test" to "gold" on the account page
  Then the customer "ann-{unique:c}@example.test" should be on the "gold" tier in the database
  And no refund should exist in the database for "ann-{unique:c}@example.test"
```

- Name the database in a check (`in the database`), and name the record
  by a value the scenario wrote: the step's own text, an Examples column,
  a `{unique:...}` value, or a value an earlier step kept. A check that
  names nothing ("a customer exists") would pass on rows any earlier run
  left behind, and is refused while recording. A value spelled in the
  step is the same every run: where the check must prove this run saved
  the row, use a `{unique:...}` value or an id an earlier step kept.
- Seed with `{unique:...}` values where a column is unique: parallel
  workers and proof replays each write their own.
- A seed `Given` states the data, not how to insert it. Seed only what the
  app gives no way to set; a `When` that the app performs goes through the
  app.
- A `Fixture:` block does not write to the database: seed in each
  scenario's `Given`.

## What is recorded

| Action | Step | Holds |
|---|---|---|
| `dbWrite` | `Given`, `When` | One `INSERT`, or one `UPDATE` with a `WHERE`; values as `$1`, `$2` binds (`body`); `extract` keeps a field of the first row written as `{captured:name}` |
| `expectRows` | `Then` | One `SELECT`; values as binds; `json` checks on its rows read as a JSON list (`count`, `/0/column` equals, `item`); `rowCount`, the rows it found when recorded |

- Values are binds, never written into the SQL: a literal is refused, and
  so is a comment. Each bind is compared with a column (`email = $1`); a
  query that returns a bind, compares values with no column, or adds
  `OR true` is refused. The `WHERE` must tie its rows to a bind with `=`,
  `IN`, `LIKE` (or `ILIKE` on PostgreSQL). Each side of an `OR`, and
  every value of an `IN`, needs such a bind (`email = $1 OR role =
  'admin'` is refused); one side of an `AND` is enough. `<>`, `NOT (...)`, a
  range alone, a bind in `LIMIT`, and `UNION` are refused, and a subquery
  must tie its own rows. A `LIKE` value made only of `%` or `_` (an
  Examples cell too) matches every row, so it is refused unless another
  condition ties the rows. A JSON field can tie a check
  (`data->>'orderRef' = $1`). Cast with `CAST($1 AS type)`, not `::`. A
  `WITH` that selects a bind as a column is refused like a `SELECT $1`.
  On MySQL join texts with `CONCAT()`: `||` runs as `OR` there and is
  refused.
  Binds are written as their tokens, so each Examples row checks its own
  value. Bind a row's whole value (`<email>`), never a part of it ("ann"
  of ann@example.test): a part is kept as written for every row and is
  refused, unless every row holds it alike. An aggregate with no `GROUP BY` is always one row: check its
  value (`/0/n equals 1`, or a count on a list it holds), never a count
  or an exists on the row itself, also when the aggregate is in the
  query's only `FROM` or `WITH` source. A count above 0, or a check that
  there is no second row, under `LIMIT`, `TOP`, `FETCH`, `DISTINCT` or a
  `GROUP BY` with no aggregate is refused, also when they sit in a query
  read `FROM` or `WITH`, and so is any check under `OFFSET`, and `LIMIT
  0`, `TOP 0` and `FETCH ... 0`. A field that must not be in a row
  (`/0/data/discount`) holds only when that row is there.
- A check reads in a transaction that is rolled back (read-only on
  PostgreSQL and MySQL). A date, a timestamp, a bigint or a numeric reads
  as text in UTC (`2026-11-07`, `2026-11-07 09:00:00+00` on PostgreSQL),
  the same on every machine; SQL Server reads a DECIMAL as a number.
- A presence check passes as soon as its rows are there, within
  `db.timeout`. A "not in the database" check (a count of 0, a `count(*)` checked as 0, a field that
  must not exist) waits for a row a worker deletes late, then must hold to
  the end of the window, so a row a worker writes late fails it. A check holding both waits for the first part, then holds
  the second to the end.
- The run report shows each check's SQL, row count and first five rows,
  masked.

## Refused

`DELETE`, `TRUNCATE`, `DROP`, `ALTER`, `GRANT`, several statements in one,
a write inside a `WITH`, an `UPDATE` without a `WHERE`, `UPDATE ... FROM`,
`INSERT ... ON CONFLICT` and `ON DUPLICATE KEY UPDATE`, MySQL's
`INSERT IGNORE`, an `UPDATE` with a `JOIN`, a second table, `ORDER BY` or
`LIMIT`, a write to a table of another database (`otherdb.t`,
`db.schema.t`), table hints (`WITH (NOLOCK)`, `READPAST`),
`SELECT ... INTO`, MySQL's `/*! ... */` comments, a backslash before a
quote on SQL Server, a write in a `Then`, a write through a view or synonym, a MySQL read
that locks rows (`FOR UPDATE`, `FOR SHARE`, `LOCK IN SHARE MODE`), an `UPDATE` whose `WHERE`
binds nothing the scenario wrote, and any statement calling a function
whose effects reach past its transaction (`pg_terminate_backend`,
`dblink`, `set_config`, sequences, advisory locks, large objects, files, `SLEEP`, `BENCHMARK`, `LOAD_FILE`, `OPENQUERY`,
`OPENROWSET`).
Statements are judged on their parsed form, never their text.

## Cleanup

When a scenario ends, green or red (and at Ctrl-C or SIGTERM), Saffron undoes its
writes, the latest first, by primary key and nothing else:

| `cleanup` | Inserts | Updates |
|---|---|---|
| `"all"` (default) | deleted | changed columns put back as they were before the step |
| `"inserts"` | deleted | left |
| `"none"` | left | left, for debugging |

A write that must be undone needs a table with a primary key; an `UPDATE`
that changes a primary key is refused under `"all"`. What a recording
wrote is undone before its proof replays it, and what the proof wrote
before the stability and row proofs. A row cleanup cannot undo is named
on the console. A killed run (SIGKILL, out of memory) undoes nothing, and
rows a database trigger writes are never undone: sweep them in
`teardown`. `run --plan` lists a write that `cleanup` keeps as a side
effect.

## The one Then that may be repaired

A `Then` is never healed, with one exception. When a database check's
stored query fails because the schema changed (a table or column gone, a
type that no longer fits), the agent repairs where the query looks: its
table, columns and joins. Its binds, its expected rows and its row count
are locked: a repair that changes any of them, or adds a `LIMIT`,
`DISTINCT`, `GROUP BY` or aggregate, is refused, and the step stays red.
A check that expects no rows, a `count(*)` of 0, a count from an Outline
column, or a field that must not be there, is not repaired: record it again.
A repaired check, and a heal that adds or changes a database write, is
left out of `accept --all`: it is accepted by file after review. The repaired check must hold on the live data and on a proof
replay; the step is reported healed and a proposal names the old and new
SQL. A query that runs and finds other rows is plain red, and no agent is
started.

## Not yet

MongoDB; `saffron eject` writes these scenarios as `test.fixme`; the
editor plugins show database steps only as the runner reports them.
