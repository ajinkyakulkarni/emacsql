# Emacsql

Emacsql is a high-level Emacs Lisp front-end for SQLite. It is
currently a work-in-progress as the s-expression query language is
still being hammered out.

It works by keeping a `sqlite3` inferior process running (a
"connection") for interacting with the back-end database. Connections
are automatically cleaned up if they are garbage collected. All
requests are synchronous.

Any [readable lisp value][readable] can be stored as a value in
Emacsql, including numbers, strings, symbols, lists, vectors, and
closures. Emacsql has no concept of "TEXT" values; it's all just lisp
objects. The lisp object `nil` corresponds 1:1 with `NULL` in the
database.

Requires Emacs 24 or later and SQLite 3.7.15 or later.

Due to [bad behavior from SQLite on Windows][stderr] Emacsql will
*not* signal error messages for invalid statements on this platform.
Fortunately this would only be an issue during package development and
shouldn't impact normal use of the database.

## Example Usage

```el
(defvar db (emacsql-connect "/var/lib/company.db"))

;; Create a table. A table identifier can be any kind of lisp value.
(emacsql db [:create-table people [name id salary]])

;; Or optionally provide column constraints.
(emacsql db [:create-table people [name (id integer :unique) (salary float)]])

;; Insert some data:
(emacsql db [:insert :into people
             :values (["Jeff"  1000 60000.0] ["Susan" 1001 64000.0])])

;; Query the database for results:
(emacsql db [:select [name id]
             :from people
             :where (> salary 62000)])
;; => (("Susan" 1001))

;; Queries can be templates, using $1, $2, etc.:
(emacsql db [:select [name id]
             :from people
             :where (> salary $1)]
         50000)
;; => (("Jeff" 1000) ("Susan" 1001))
```

## Schema

A table schema is a vector of column specifications, or a list
containing a vector of column specifications followed by table
specifications. A column identifier is a symbol and a specification
can either be just this symbol or it can include constraints. Because
Emacsql stores entire lisp objects as values, the only relevant (and
allowed) types are `integer`, `float`, and `object` (default).

Columns constraints include `:primary` (aka `PRIMARY KEY`), `:unique`,
`:non-nil` (aka `NOT NULL`), `:default`, and `:check`.

Table constraints can be `:primary`, `:unique`, `:check`, and `:foreign`.

```el
;; No constraints schema with four columns:
[name id building room]

;; Add some column constraints:
[(name :unique) (id integer :primary) building room]

;; Add some table constraints:
([(name :unique) (id integer :primary) building room]
 :unique [building room] :check ())
```

Foreign keys are the most complex. Action triggers are `:on-delete`
or `:on-update` and possible actions are `:set-nil`, `:set-default`,
`:restrict`, `:cascade`. See [the SQLite documentation][foreign] for
the details on what each of these means.

`:foreign (<child-keys> <parent-table> <parent-keys> [<actions>])`.

```el
;; "subject" table
[(id integer :primary) subject]

;; "tag" table references subjects
([(subjectid integer) tag]
 :foreign (subjectid subject id :on-delete :cascade))
```

Put the keys in a vector if the reference is composite. Also remember
that foreign key checks are currently disabled by default in SQLite,
so you'll need to enable it for each connection.

```el
(emacsql db [:pragma (= foreign_keys on)])
```

## Operators

Emacsql currently supports the following expression operators, named
exactly like so in a structured Emacsql statement.

    *     /     %     +     -     <<    >>    &
    |     <     <=    >     >=    =     !=
    is    like  glob  and   or    in

The `<=` and `>=` operators accept 2 or 3 operands, transforming into
a SQL `_ BETWEEN _ AND _` operator as appropriate.

With `glob` and `like` keep in mind that they're matching the
*printed* representations of these values, even if the value is a
string.

The `||` concatenation operator is unsupported because concatenating
printed representations breaks an important constraint: all values must
remain readable within SQLite.

## Structured Statements

The database is interacted with via structured s-expression
statements. You won't be concatenating strings on your own. (And it
leaves out any possibility of a SQL injection!) See the "Usage"
section above for examples. A statement is a vector of keywords and
other lisp object.

Structured Emacsql statements are compiled into SQL statements. The
statement compiler is memoized so that using the same statement
multiple times is fast. To assist in this, the statement can act as a
template -- using `$1`, `$2`, etc. -- working like the Elisp `format`
function.

### Keywords

Rather than the typical uppercase SQL keywords, keywords in a
structured Emacsql statement are literally just that: lisp keywords.
When multiple keywords appear in sequence, Emacsql will generally
concatenate them with a dash, e.g. `CREATE TABLE` becomes
`:create-table`.

#### :create-table `<table>` `<schema|select>`

Provides `CREATE TABLE`. A selection can be used in place of a schema,
which will create a `CREATE TABLE ... AS` statement.

```el
[:create-table employees [name (id integer :primary) (salary float)]]
[:create-table (:temporary :if-not-exists employees) ...]
[:create-table names [:select name :from employees]]
```

#### :drop-table `<table>`

Provides `DROP TABLE`.

```el
[:drop-table employees]
```

#### :select `<column-spec>`

Provides `SELECT`. `column-spec` can be a `*` symbol or a vector of
column identifiers, optionally as expressions.

```el
[:select [name (/ salary 52)] ...]
```

#### :from `<table>`

Provides `FROM`.

```el
[... :from employees]
[... :from [employees accounts]]
[... :from [employees (accounts a)]]
[... :from (:select ...)]
[... :from [((:select ...) s1) ((:select ...) s2)]]
```

#### :where `<expr>`

Provides `WHERE`.

```el
[... :where (< count 10)]
```

#### :group-by `<expr>`

Provides `GROUP BY`.

```el
[... :group-by name]
```

#### :order-by `<expr>|(<expr> <:asc|:desc>)|[<expr> ...]`

Provides `ORDER BY`.

```el
[... :order-by date]
[... :order-by [(width :asc) (height :desc)]]
[... :order-by [(width :asc) (- height)]]
```

#### :limit `<limit>|[<offset> <limit>]`

Provides `LIMIT` and `OFFSET`.

```el
[... :limit 50]
[... :limit [150 50]]
```

#### :insert, :replace

Provides `INSERT`, `REPLACE`.

```el
[:insert :into ...]
[:replace :into ...]
```

#### :into `<table>`

Provides `INTO`.

```el
[:into employees ...]
[:into (employees [id name]) ...]
```

#### :delete

Provides `DELETE`.

```el
[:delete :from employees :where ...]
```

#### :values `<vector>|(<vector> ...)`

```el
[:insert :into employees :values ["Jeff" 0]]
[:insert :into employees :values (["Jeff" 0] ["Susan" 0])]
```

#### :update `<table>`

Provides `UPDATE`.

```el
[:update people :set ...]
```

#### :set `<assignment>|[<assignment> ...]`

Provides `SET`.

```el
[:update people :set (= name "Richy") :where ...]
[:update people :set [(= name "Richy") (= salary 300000)] :where ...]
```

#### :union, :union-all, :difference, :except

Provides `UNION`, `UNION ALL`, `DIFFERENCE`, and `EXCEPT`.

```el
[:select * :from sales :union :select * :from accounting]
```

#### :pragma `<expr>`

Provides `PRAGMA`.

```el
(emacsql db [:pragma (= foreign_keys on)])
```

### Templates

To make statement compilation faster, and to avoid making you build up
statements dynamically, you can insert `$n` "variables" in place of
identifiers and values. These refer to argument positions after the
statement in the `emacsql` function, 1-indexed.

    (emacsql db [:select * :from $1 :where (> salary $2)] 'employees 50000)

## Limitations

Emacsql is *not* intended to play well with other programs accessing
the SQLite database. Non-numeric values are are stored encoded as
s-expressions TEXT values. This avoids ambiguities in parsing output
from the command line and allows for storage of Emacs richer data
types. This is a high-performance database specifically for Emacs.

## See Also

 * [SQLite Documentation](https://www.sqlite.org/docs.html)


[readable]: http://nullprogram.com/blog/2013/12/30/#almost_everything_prints_readably
[stderr]: http://thread.gmane.org/gmane.comp.db.sqlite.general/85824
[foreign]: http://www.sqlite.org/foreignkeys.html
