# Database steps (beta)

A step can check what the database holds, or seed the data a scenario
starts from, with no query written by hand. While recording, the agent
reads the schema, writes one parameterized statement, runs it, and records
it with the step's values as binds; every later run replays it at zero
tokens. They work in browser scenarios and in `@api` scenarios alike.
PostgreSQL, MySQL, SQL Server, Oracle, SQLite, MariaDB and CockroachDB
with SQL, and MongoDB, DynamoDB and Firestore with one JSON command per
step, named by the URL's scheme (`postgres://`, `mysql://`, `mssql://`,
`oracle://`, `sqlite:path/to/app.db`, `mariadb://`, `cockroachdb://`,
`mongodb://`, `dynamodb://<region>`, `firestore://<project-id>`).

## Setup

```json
{
  "db": { "url": "{env:DATABASE_URL}", "writes": false, "allow": ["localhost", "shop_test"], "timeout": 3000, "cleanup": "all" }
}
```

- `npm install node-sql-parser` and the engine's driver (`pg`, `mysql2`,
  `mssql` or `oracledb@6`; none for SQLite, which Node has built in) in the
  project; for MongoDB, DynamoDB or Firestore, their driver alone
  (`mongodb`, `@aws-sdk/client-dynamodb`, `@google-cloud/firestore@7`). They
  load only when `db` is set.
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
- A step says what should be stored, never how to query it. The agent
  writes SQL on a SQL engine and one JSON command on MongoDB, DynamoDB
  and Firestore. On DynamoDB the value that names the record must be its
  key, or an index's key (there is no scan): "the order "A-{unique:o}" of
  customer "ann@example.test"". On Firestore it is the document's id or a
  field's value.
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
  condition ties the rows. In a "not in the database" check a `LIKE` value
  holding a backslash (a `[` on SQL Server) is refused: compare that text
  with `=`. Placeholders start at `$1`. Such a check is red when rows are
  found once a bind is sent as the other type (text as a number, `true` as
  1 or as a boolean, 1 as a boolean, `0x` hex as bytes, a JSON bind's
  values, a value read as JSON as a JSON string): give each bind the type
  the database keeps, as the refusal says. A bind only a negation compares
  (`<> $2`) is left as sent. A JSON field can tie a check
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
  query's only `FROM` or `WITH` source. A count above 0, an `every`
  check on the rows, or a check that there is no second row, under `LIMIT`, `TOP`, `FETCH`, `DISTINCT`, a
  WHERE that picks one of the table's own rows by a subquery (`id = (SELECT
  max(id) FROM orders WHERE ref = $1)`, `>= ALL (...)`), a WHERE that
  keeps one row of each group (`max(id) ... GROUP BY`, a `row_number()`
  column) or a
  `GROUP BY` with no aggregate is refused, also when they sit in a query
  read `FROM` or `WITH`, and so is any check under `OFFSET`, and `LIMIT
  0`, `TOP 0` and `FETCH ... 0`. A row holds every column it selects,
  NULL too: a column that must be empty is `discount IS NULL` in the
  `WHERE`, never `/0/discount` exists false, which is refused. A field
  that must not be in a JSON column (`/0/data/discount`) holds only when
  that row is there.
- A "differs" check reads text that spells a number as the number (a
  decimal, a bigint and a count come back as text), and a number the
  driver returns as one (SQLite's `REAL`, a document's number) the same
  way: `12.50` does not differ from `12.5`, `+12.5`, `012.50`, nor `0.5`
  from `.5`. Where the text itself matters (a version `1.10`), give the
  check `"type": "string"`, as its failure says; a code with a leading
  zero and no point (`01234`) is compared as text already. A `char(n)`
  value is compared without its padding blanks, and a dollar amount
  (PostgreSQL's `money`, `$1,234.50`, on any engine) by its amount. A
  boolean column takes `true` or `false`: a number against it (`1` for a
  MySQL `BOOLEAN`) is refused. An "at most" check with an empty or null
  bind is refused, as a "not in the database" check is: to count rows
  whose column is rightly empty, compare its length with the number 0
  (`length(note) = $1`); on Oracle an empty text is NULL, so write
  `note IS NULL` there, beside a column compared with a bind
  (`note IS NULL AND email = $1`).
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

## MongoDB

A MongoDB step holds one JSON command where SQL has a statement, every
value a placeholder bound from the step:

```json
{"find": "customers", "filter": {"email": "$1"}, "projection": {"name": 1}}
{"count": "orders", "filter": {"customer": "$1:objectId"}}
{"insert": "customers", "documents": [{"name": "$1", "email": "$2"}]}
{"update": "customers", "filter": {"email": "$1"}, "set": {"tier": "$2"}}
```

- A `count` returns one row, `{"n": <how many>}`: check `/0/n` equals `1`
  with type `number`.
- A placeholder names the BSON type it is sent as: `"$1:objectId"`,
  `"$1:uuid"`, `"$1:number"`, `"$1:long"`, `"$1:decimal"`, `"$1:bool"`,
  `"$1:date"`, `"$1:binary"` (bytes as `0x` hex). A bind given as null
  finds a field held as null (or missing); `{"$exists": false}` finds only
  a document without the field.
- Filters use `$eq`, `$ne`, `$gt`, `$gte`, `$lt`, `$lte`, `$in`, `$nin`,
  `$exists`, `$all`, `$elemMatch`, `$not`, `$and`, `$or` and `$nor`;
  `$where`, `$expr`, `$regex` and aggregation are refused. A field equal
  to a bind, or `$in` binds, ties a check, on each side of an `$or`.
  Compare embedded fields with dotted paths (`"addr.city": "$1"`), never a
  whole document or list as a value, plainly or under `$eq`, `$ne`, `$in`,
  `$nin` or `$all`. Placeholders start at `$1`.
- A "not in the database" check is red when its filter finds a document
  with a bind sent as another type (an ObjectId's hex as text), when no
  document has a field the filter finds documents by (compared with a
  bind, in a range, under `$elemMatch`, or `$exists` true) or a field a
  check says must not be there, when
  `$elemMatch` reads a field held as one value, or when the collection is
  not there, so type each bind as its field holds it, both ends of a range
  too.
- A `find` with a `projection` cannot check that a field is missing, at
  any depth, a projection names fields only (no `lines.$`), and `every` or
  a count above 0 under `limit` is refused.
- An `update` never sets `_id`; its `set` may name dotted paths
  (`"addr.city": "$2"`, a list's item by its number, `"items.0.qty"`),
  never `$`, `$[]` or `$[name]`, and an inserted document writes a nested
  field as a nested object.
- An ObjectId reads as its hex, a UUID as its dashed text, a date as UTC
  ISO text, an int64 or a decimal as text. `extract` keeps a new `_id` with `/_id`.
- Cleanup deletes inserted documents and puts back only the paths an
  update set, type for type. A MongoDB check is never repaired: a collection has no schema.

## Oracle and SQLite

- Oracle: names are in capitals unless created in quotes, so check a
  row's columns as `/0/NAME`; a name created in quotes cannot be used.
  Write plain SQL: `FETCH FIRST` and `ROWNUM` are a `LIMIT`, and `CONNECT
  BY`, `(+)`, `AS OF`, `q'...'` strings, `MERGE` and PL/SQL are refused. A
  `DATE` holds a time (`2026-11-07 00:00:00`), a `NUMBER` reads as a
  number, decimals too (only one past 15 digits reads as text), and an
  empty string is `NULL`. A check never calls `NEXTVAL`. An `INSERT` writes one row when Oracle
  numbers its key.
- SQLite: `db.url` is `sqlite:path/to/app.db`, a file that must be there,
  and `db.allow` lists its name or path. A boolean is `1` or `0`; a date
  or JSON is the text the app wrote. A bind compared with an expression
  (`json_extract(data, '$.id') = $1`) must be the type the expression
  gives, or cast both to one type: a "not in the database" check that
  only a type keeps empty is refused. `INSERT OR REPLACE`, `UPDATE OR ...`
  and `REPLACE` are refused.
- MariaDB runs as MySQL and CockroachDB as PostgreSQL, by the same rules.

## DynamoDB

One JSON command per step, every value a placeholder:

```json
{"get": "Orders", "key": {"customer": "$1", "orderId": "$2"}}
{"query": "Orders", "key": {"customer": "$1", "orderId": {"$beginsWith": "$2"}}, "filter": {"status": "$3"}}
{"put": "Orders", "item": {"customer": "$1", "orderId": "$2", "total": "$3:number"}}
{"update": "Orders", "key": {"customer": "$1", "orderId": "$2"}, "set": {"status": "$3"}}
```

- `db.url` is `dynamodb://<region>`, or `dynamodb://<host>:<port>` for
  DynamoDB Local; AWS credentials come from the environment, never the
  URL. `db.allow` lists the region or the local host.
- A `get` names the whole key; a `query` names its partition key by
  equality, with a sort key condition (`$eq`, `$gt`, `$gte`, `$lt`,
  `$lte`, `$between`, `$beginsWith`), an `index`, a `filter` (those and
  `$ne`, `$in`, `$exists`, on top-level attributes) or a `limit`. No
  scan, no delete.
- A key attribute is sent as the type the table declares. Any other
  attribute that is not text names its type: `"$1:number"`, `"$1:bool"`,
  `"$1:binary"` (bytes as `0x` hex). A bind given as null finds an
  attribute held as NULL. A "not in the database" check is red when an
  item is found with a bind sent as another type (both ends of a range
  too), or when it compares a set or a list with a value, and never reads
  a global secondary index.
- A `put` never replaces an item; an `update` changes one item that is
  there, never a key attribute. Cleanup deletes the put item and puts set
  attributes back. A DynamoDB check is never repaired.

## Firestore

One JSON command per step, every value a placeholder:

```json
{"get": "users/$1"}
{"query": "users", "where": [["email", "==", "$1"], ["age", ">=", "$2:number"]]}
{"create": "users/$1", "data": {"name": "$2"}}
{"add": "users", "data": {"name": "$1"}}
{"update": "users/$1", "set": {"tier": "$2", "address.city": "$3"}}
```

- `db.url` is `firestore://<project-id>`, or
  `firestore://<host>:<port>/<project-id>` for the emulator; credentials
  come from the environment. `db.allow` lists the project id or the
  emulator's host.
- A collection is written in the path; a document's id may be a
  placeholder. `where` filters are joined by AND and use `==`, `!=`, `<`,
  `<=`, `>`, `>=`, `in`, `not-in`, `array-contains`,
  `array-contains-any`. No delete, `orderBy` or collection group.
- A placeholder names its type when the field is not text: `"$1:number"`,
  `"$1:double"`, `"$1:bool"`, `"$1:timestamp"`, `"$1:reference"`,
  `"$1:binary"` (bytes as `0x` hex). A "not in the database" check is red
  when a document is found with a bind sent as another type (both ends of
  a range too), when it compares a list by `==` or `in` (write
  `array-contains` or `array-contains-any`) or one value by
  `array-contains`, and refused on a collection that holds no document.
- A row holds the document's fields with `__id__` and `__path__`. A
  `create` never writes over a document; cleanup deletes created
  documents and puts set fields back. A Firestore check is never
  repaired.

## Not yet

`saffron eject` writes these scenarios as `test.fixme`; the
editor plugins show database steps only as the runner reports them.
