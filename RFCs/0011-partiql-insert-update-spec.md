- Start Date: 2022-06-17
- PartiQL Issue [partiql/partiql-docs/#4](https://github.com/partiql/partiql-docs/issues/4)
- RFC PR: [partiql/partiql-docs/#11](https://github.com/partiql/partiql-docs/pull/11)

# Summary

[summary]: #summary

Summary

This RFC specifies PartiQL's syntax and semantics for insert-or-update—also known as UPSERT—and insert-or-replace as part 
of PartiQL Data Manipulation Language (DML). UPSERT is an informal term for a statement that denotes inserting or updating 
a set of data to a target database’s table based on existing availability condition of that data in the same table.

# Motivation

[motivation]: #motivation

The PartiQL Specification as it stands today does not specify the syntax and semantics of Data Manipulation Language (DML).
The `INSERT` is one of the statements of the DML that requires specification; the motivation is to provide guidance
through the specification for the DML insert-or-update implementations based on PartiQL.

# Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

The keywords “MUST”, “MUST NOT”, “REQUIRED”, “SHALL”, “SHALL NOT”, “SHOULD”, “SHOULD NOT”, “RECOMMENDED”, “MAY”, and
“OPTIONAL” in this format specification are to be interpreted as described
in [Key Words RFC2119](https://datatracker.ietf.org/doc/html/rfc2119).

## Terminology

* **Arbiter Constraint**: A constraint that is used by an algorithm in a database system for inferring if data conflict
  exists during data insertion.
* **Arbiter Index**: A unique index that is used by an algorithm in a database system for inferring if conflict exists
  on a unique index during data insertion.
* **Attribute**: The column of a table (or schema) are named by attributes in Relational model. In Non-relational model
  they’re equivalent to column headers; for example in NoSQL model they are names in name/value pairs.
* **Closed Schema**: A schema where all attributes are declared, and undeclared attributes are not permitted.
* **DDL**: A data definition language defines SQL statements used for defining or altering various database entities
  like tables and views.
* **DML**: A data manipulation language defines SQL statements used for adding (inserting), deleting, and modifying (updating)
  data in a database.
* **Declared Attributes**: Any attribute that is defined in the Schema with a name, type, whether the attribute is
  required or optional, and any additional properties; all declared attributes are either required or optional.
* **Item**: A collection of attributes in a Non-relational (E.g. NoSQL) data model.
* **NoSQL Data Model**: A non-relational data model that represents the data in a non-tabular way, for example in
  document or key-value form.
* **Open Schema**: A Schema where undeclared attributes are permitted.
* **Optional Attributes**: A set, comprising the declared attributes that are not necessarily required for a successful 
  data insertion or update to occur.
* **PartiQL value**: A PartiQL value is a value defined by PartiQL specification that can be one of absent, missing,
  scalar, tuple, array, or bag. (see [PartiQL Specification](https://partiql.org/assets/PartiQL-Specification.pdf)
  Figure 1 for more details).
* **Partial Unique Index**: A unique index with predicate. (E.g. a `WHERE a > 10;`)
* **Relation**: A two-dimensional representation of data in rows and columns.
* **Relational Model:** A data model that represents data as relations. With this model data is organized into relations
  where each relation is an unordered collection of tuples.
* **Required Attributes** A set, comprising the declared attributes that must exist for a data item to conform to the
  Schema.
* **Schema:** Schema is an abstract entity that is used as a blueprint to describe data along with its associated
  constraints.
* **Tuple:** The rows of relations which are collection of column values. In a Non-relational data model (E.g.
  NoSQL/Semi-structured model) denotes a collection of name/value pairs also known as an Item. Tuples can be ordered or
  unordered depending on the database system.

*Note: Some examples and parts of the text are based on [PostgresSQL INSERT documentation.](https://www.postgresql.org/docs/current/sql-insert.html)*

### Out of scope

* Returning output from INSERT statement.
* Constraints and unique index syntax semantics.
* Type coercion rules.
* Security privileges for execution of DML statements.

## 2. Proposed Grammar and Semantics

Figure 1. illustrate the BNF grammar corresponding to PartiQL’s `INSERT INTO ... ON CONFLICT` statements. 
See Appendix 1. for the Grammar conventions.

Note:

* All the definitions that are missing from the Grammar refer to the definitions that are already in SQL-92 standard.
* <index expr> and <index predicate> will be defined in a separate RFC.

```EBNF
<insert statement> ::= INSERT INTO <table name> [ AS <alias> ] 
    [  ( <attr name> [, <attr name> ]... ) ]
        <values>
    [ ON CONFLICT [ <conflict target> ] <conflict action> ]

<values> ::= DEFAULT VALUES | <values clause>
      | <bag value> | <sfw query>

<values clause> ::= VALUES <value> [, <value>]...

<value> ::= ( { <value expr> | DEFAULT } [, { <value expr> | DEFAULT } ]...)

<conflict target> ::=
    ( <index target> [, <index target>]... )
      [ WHERE <index predicate> ]
    | ( { <primary key> | <composite primary key> } )
    | ON CONSTRAINT <constraint name>

<index target> ::= <index attr name> | <index expr>

<conflict action> ::= DO NOTHING 
    | DO UPDATE <do update>
    | DO REPLACE <do replace>

<do update> ::= EXCLUDED
        | SET {
          <attar values> [, <attr values>]...
        | <tuple value>
      }
   [ WHERE <condition> ]

<do replace> ::= SET {
        <attar values> [, <attr values>]...
            | <tuple value>
      }
   [ WHERE <condition> ]

<attr values> ::=  {
            <attr name> = { <value expr> | DEFAULT }
            | ( <attr name> [, <attr name>]... ) = <value>
            | ( <attr name> [, <attr name>]... ) = ( <sfw query> )
        }

<primary key> ::= <attr name>

<composite primary key> ::= <attr name>, <attr name> [, <attr name> ]...

<value expr> ::= <partiql value>
     | <sql value expr>

<partiql value> ::= NULL
    | <ion value>
    | <tuple value>
    | <collection value>

<sql value expr> ::= <sql expression>

<ion value> ::= <backtick> <ion literal> <backtick>

<tuple value> ::= <l curly brace> <r curly brace>
    | <l curly brace> <string value> : <partiql value> [, <string value> : <partiql value>]... <r curly brace>

<collection value> ::= <array value>
    | <bag value>

<array value> ::= <left bracket> { <partiql value> [, <partiql value>]... } <right bracket>

<bag value> ::= <opening bag> { <partiql value> [, <partiql value>]... } <closing bag>

(* See PartiQL spec. figure 3: BNF Grammar for PartiQL Queries:
    https://partiql.org/assets/PartiQL-Specification.pdf *)
<sfw query>

<index attr name> ::= <attr name>
<attr name> ::= <identifier>
<constraint name> ::= <identifier>

(* Literals *)
<l curly brace> ::= 
  “{”
<r curly brace> ::=
  “}”
<left bracket> ::=
  “[”
<right bracket> ::=
  “]”
<backtick> ::=
  `
<opening bag> ::=
  <<
<closing bag> ::=
  >>
<left paren> ::=
  “(”
 <right paren> ::=
  “)”
```

## 3. Description

Implementation based on this specification MUST lead to atomic insertions of PartiQL values into a Schema (E.g. a database table). 
Atomic insertion means, when inserting multiple values—in case of an error in insertion of one, or more
data items—the existing data items MUST remain intact.

### 3. 1 Attribute names, <value expr>

When attribute names are omitted and values are provided by `VALUES` clause or a `<sub select>`:

(a) If the number of values in a data item is equal to the number of declared attributes, the values are assumed to
refer to the declared attributes, in order of their definition in the table schema, and

(b) If the number of values in data item (M) is fewer than the number of declared attributes (N) the values are assumed
to refer to the first M declared attributes, in order, and

(c) If the number of values in a data item (M) is greater than the number of declared attributes (N) it is an error, and

(d) If a value is not specified for every required attribute, it is an error.

It’s worth emphasizing that, the implementing database MUST provide an ordering for the declared attributes that appear
in the Schema.

When attribute names are omitted and `VALUES <bag value>` is provided, the values in `<bag value>` MUST include required
attribute–value pairs and MAY include optional or undeclared attribute-value pairs.

When attribute names are provided, they can be listed in any order. Each required attribute that is omitted in the 
provided attributes, SHOULD be filled with a default value, either its declared default value or an implementation of null.

If the expression for any attribute is not of the correct data type, automatic type conversion i.e. type coercion MAY
be attempted.

#### 3.1  Examples

In the following examples, `films` table has a closed schema.

Insert a single row:

```SQL
INSERT INTO films
VALUES ('UA502', 'Bananas', 105, '1971-07-13', 'Comedy', '82 minutes');
```

In this example, the `len` required attribute is omitted in the explicit list, and therefore it will have the default
value:

```SQL
INSERT INTO films (code, title, did, date_prod, kind)
VALUES ('T_601', 'Yojimbo', 106, '1961-06-16', 'Drama');
```

The following examples use the `DEFAULT` clause for the date attributes rather than specifying a value:

```SQL
INSERT INTO films
VALUES ('UA502', 'Bananas', 105, DEFAULT, 'Comedy', '82 minutes');
INSERT INTO films (code, title, did, date_prod, kind)
VALUES ('T_601', 'Yojimbo', 106, DEFAULT, 'Drama');
```

The following examples statement inserts a row or item consisting entirely of default values for required attributes:

```SQL
INSERT INTO films DEFAULT
VALUES;
```

The following example statement inserts multiple rows using the multirow VALUES syntax:

```SQL
INSERT INTO films (code, title, did, date_prod, kind)
VALUES ('B6717', 'Tampopo', 110, '1985-02-10', 'Comedy'),
       ('HG120', 'The Dinner Game', 140, DEFAULT, 'Comedy');
```

In the following statement, we insert two values to `Music` table, `Music` table has an open schema with `Artist`
and `SongTitle` as required attributes:

```SQL
INSERT INTO Music
VALUES
    <<{'Artist' : 'Acme Band', 'SongTitle' : 'PartiQL Rocks'},
    {'Artist' : 'Emca Band', 'TitleSong' : 'PartiQL Rocks'};
>>
```

In the following statement we insert multiple person items as a bag value to `Person` table, `Person` table has an open
schema with `LastName` and `DOB` as required attributes:

```SQL
INSERT INTO Person <<{'FirstName' : 'Raul',
    'LastName' : 'Lewis',
    'DOB' : 1963-08-19T,
    'GovId' : 'LEWISR261LL',
    'GovIdType' : 'Driver License',
    },{'LastName' : 'Logan',
    'DOB' : 1967-07-03T,
    'Address' : '43 Stockert Hollow Road, Everett, WA, 98203'
    },{'LastName' : 'Pena',
    'DOB' : 1974-02-10T,
    'GovId' : '744 849 301',
    'GovIdType' : 'SSN',
    'Address' : '4058 Melrose Street, Spokane Valley, WA, 99206'
    }>>;
```

### 3.2 Query

`SELECT` query expression can be used for inserting the resulting values from the query to the target table. The
resulting tuples (E.g. rows) MUST have the same attribute name and type as the target schema’s required attributes.

Optional attributes in target schema that have the same name in the resulting tuples from the source schema,  
SHOULD have the same type. If the implementing database provides type coercion rule for the types involved in source and
destination, the `SELECT` MAY lead to a successful outcome, otherwise it's a `SemanticError`. See examples for more clarification.

#### 3.4 Examples

The following example statement inserts items from `Vehicles` table to `HeavyVehicles` table. Both tables have the same
layout for required and optional attributes, therefore the `INSERT` statement can insert items using a `SELECT` query.

```SQL
-- The following `CREATE TABLE` is arbitrary since the PartiQL DDL is yet to be
-- defined.
CREATE TABLE Music
(
    Artist     VARCHAR(20) NOT NULL,
    SongTitle  VARCHAR(30) NOT NULL,
    AlbumTitle VARCHAR(25) NOT NULL,
    Year       INT,
    Price      FLOAT,
    Genre      VARCHAR(10),
    PRIMARY KEY (Artist, SongTitle)
);

CREATE TABLE RockAlbums
(
    Artist     VARCHAR(20) NOT NULL,
    SongTitle  VARCHAR(30) NOT NULL,
    AlbumTitle VARCHAR(25) NOT NULL,
    Year       INT,
    RockGenre  VARCHAR(10),
    PRIMARY KEY (Artist, SongTitle)
);

INSERT INTO Music
SELECT *
FROM RockAlbums
WHERE RockGenre IN ('Alternative', 'SpaceRock');
```

The following statement shows that the attributes can be specified with a sub-select.

```SQL
INSERT INTO Music (Artist, SongTitle)
SELECT Artist, SongTitle RockAlbums WHERE RockGenre IN ('Alternative', 'SpaceRock');
```

The following example statement attempts to insert items from `Vehicles` table to `HeavyVehicles` table. Both tables
have the same layout for required attributes but have different types for `Year` optional attribute, therefore
the `INSERT` statement using a `SELECT` query leads to a `SemanticError` if the implementing database system does not
implement a type coercion rule from `DATE` to `INT`.

```SQL
-- The following `CREATE TABLE` is arbitrary since the PartiQL DDL is yet to be
-- defined.
CREATE TABLE Music
(
    Artist     VARCHAR(20) NOT NULL,
    SongTitle  VARCHAR(30) NOT NULL,
    AlbumTitle VARCHAR(25) NOT NULL,
    Year       INT,
    Price      FLOAT,
    Genre      VARCHAR(10),
    PRIMARY KEY (Artist, SongTitle)
);

CREATE TABLE RockAlbums
(
    Artist     VARCHAR(20) NOT NULL,
    SongTitle  VARCHAR(30) NOT NULL,
    AlbumTitle VARCHAR(25) NOT NULL,
    Year       DATE,
    RockGenre  VARCHAR(10),
    PRIMARY KEY (Artist, SongTitle)
);

INSERT INTO Music
SELECT *
FROM RockAlbums;
```

The following example statement, inserts some rows into table `films` from a table `tmp_films`. Both tables must have
the same layout for required attributes and optional attributes that are in common.

```SQL
INSERT INTO films
SELECT *
FROM tmp_films
WHERE date_prod < '2004-05-07';
```

### ON CONFLICT

`ON CONFLICT` can be used to specify an alternative action to raising a constraint violation error (E.g. unique
constraint violation). In case target schema does not specify at least one constraint and ON CONFLICT is used, 
a `SemanticError` SHOULD be thrown by the implementing database system.

When present `ON CONFLICT` clause specifies an alternative action to raising a constraint or unique index violation
error (E.g. primary key violation). For each individual PartiQL value proposed for insertion, either the insertion
proceeds, or, if an arbiter constraint or index is violated, the alternative `<conflict action>` is
taken. `ON CONFLICT DO NOTHING` avoids inserting a PartiQL value as its alternative
action. `ON CONFLICT DO UPDATE` updates the conflicting PartiQL value with its alternative action,
whereas  `ON CONFLICT DO REPLACE` replaces it.

`ON CONFLICT` implementation based on this specification MUST guarantee atomic outcomes—assuming no syntactic or semantic 
errors. In case of an error the existing data MUST remain intact.

See `ON CONFLICT` clause in the Parameters section for more details and examples.

## 4. Parameters

### 4.1 Insert Parameters

This section covers parameters that may be used when only inserting new PartiQL values. Parameters exclusively used with
the `ON CONFLICT` clause are described separately.

`<table name>`
The name (optionally schema-qualified) of an existing table.

`<alias>`
A substitute name for `<table name>`. When an alias is provided, it SHOULD hide the actual name of the table. This is
particularly for the instance when `ON CONFLICT` targets a database table named `EXCLUDED`. In such cases
since `EXCLUDED` is also the name of the special table representing rows proposed for insertion, using the alias
disambiguates the name conflict.

`<attr name>`
The name of an attribute in the table named by `<table name>`. When referencing an attribute with `ON CONFLICT`, the
table name MUST get omitted from the specification of a target attribute. For
example, `INSERT INTO table_name ... ON CONFLICT DO UPDATE SET table_name.attr = 1` is invalid; this is
because, as `ON CONFLICT` operates on `<table name>` only, having the option of using the table name alongside of table
alias (if defined) is superfluous and can lead to potentially more user errors while making the statements less readable.

`DEFAULT VALUES`
All required attributes MUST be filled with their default values, as if `DEFAULT` were explicitly specified for each
attribute. If the target database system has no concept or implementation of default values, this should return
a `SemanticError` exception.

`<expr>`
An expression or value to assign to the corresponding attribute.

`DEFAULT`
The corresponding attribute gets filled with its default value. For a generated attribute (E.g. auto-increments),
specifying this is permitted which SHOULD result in computing the attribute value from its
generation expression. Examples for `DEFAULT`:

```SQL
-- The following `CREATE TABLE` is arbitrary since the PartiQL DDL is yet to be
-- defined.
CREATE TABLE Foo
(
    foo_key  serial PRIMARY KEY,
    bar      VARCHAR(40) NOT NULL,
    baz_date DATE        NOT NULL DEFAULT CURRENT_DATE
);

INSERT INTO FOO
VALUES (DEFAULT, 'foo for today');

INSERT INTO FOO(bar)
VALUES ('bar for today');

SELECT *
FROM Foo;

-- Result
-- foo_key  bar                   bar_date
-- 1        foo for today         2022-05-04T00:00:00.000Z
-- 2        bar for today         2022-05-04T00:00:00.000Z
```

`<sub-select>`
A query (`SELECT` statement) that supplies the rows or items to be inserted.

This example inserts some rows into table `films` from a table `tmp_films` with the same required attribute layout
as `files` and no common optional attributes between `films` and `tmp_films` tables.

```SQL
INSERT INTO films
SELECT *
FROM tmp_films
WHERE date_prod < '2004-05-07';
```

### 4.2 ON CONFLICT Clause

`<conflict target>`
`<conflict target>` is an optional parameter that specifies which conflicts `ON CONFLICT` takes the alternative action
on. This SHOULD be implemented by choosing primary key, arbiter index(es), or specifying constraint using `ON CONSTRAINT`
clause. When omitted, there MUST be an inference logic on the implementing database system to unambiguously determine if
a constraint is being violated by the values specified in the `INSERT` statement. If the implementing database can’t
guarantee this inference a `SemanticError` MUST be returned in case `<conflict target>` is omitted.

When provided, `<conflict target>` MUST either be used to infer primary key, unique indexes, or specify a constraint.
For unique indexes it should be used to perform unique index inference using the provided index attribute names
and/or `<index expression>` and an optional `<index predicate>`. All `<table name>` unique indexes that,
without regard to order, contain exactly the `conflict target`–specified attributes or expressions should be inferred (chosen)
as arbiter indexes. If an `<index predicate>` is specified, it must be able to be inferred as arbiter indexes.
Note that this means a non-partial unique index (a unique index without a predicate) should be inferred (and thus used
by `ON CONFLICT`) if such an index satisfying every other criteria is available. If an attempt at inference is
unsuccessful, a `SemanticError` MUST be raised.

For both `ON CONFLICT DO UPDATE` and `ON CONFLICT DO REPLACE`, when conflict occurs on a primary key or a constraint and
the proposed value for `UPDATE` or `REPLACE` is a `<tuple value>`—primary_key, or the constraint MUST exist in the
proposed `<tuple value>`, otherwise a `SemanticError` MUST be raised. See *ON CONFLICT DO UPDATE Examples`* section for
more clarification.

#### Excluded keyword

In addition to the existing rows or items access, `ON CONFLICT` and `WHERE` clauses have readonly access to rows/items
proposed for insertion using the special `EXCLUDED` keyword. The `EXCLUDED` is chosen to reduce the learning overhead as
it’s a common term across PostgresSQL and SQLLite.

`<conflict action>`
`<conflict action>` specifies an alternative `ON CONFLICT` action. It can be either `DO NOTHING`, `DO UPDATE`,
or `DO REPLACE`. The `ON CONFLICT` and `WHERE` clauses have readonly access to the existing rows or items using the
table's name (or an alias). In case of existence of SQL triggers or equal event-driven procedures, the effects of those
per-row/per-item triggers/operations MUST be reflected in `EXCLUDED` values, since those effects may have contributed to
the row being excluded from insertion.

`<primary key>`
Explicitly specifies the `<table name>`'s primary key.

`<composite primary key>`
Explicitly specifies `<table name>`'s composite primary key. E.g. a combination for `partition_key_sort_key` for an
implementation of a NoSQL database system.

`<index attr name>`
The name of a `<table name>` attribute used to infer arbiter indexes.

`<index expression>`
Similar to `<index attr name>`, but must be used to infer expressions on `<table name>` attributes appearing within index
definitions.

`<index predicate>`
Used to allow inference of partial unique indexes. Any indexe that satisfy the predicate (which need not actually be
defined as partial indexes) can be inferred.

`<constraint name>`
Explicitly specifies an arbiter constraint by name, rather than inferring a constraint or index.

`<condition>`
An expression that MUST return boolean values `TRUE` or `FALSE`. Only rows or items for which this expression returns `TRUE`
MUST be updated. Note that `<condition>` MUST be evaluated last; after a conflict has been identified as a candidate to
update.

`INSERT` with an `ON CONFLICT DO UPDATE` or `ON CONFLICT DO REPLACE` clause MUST be a “deterministic” statement. This
means that the command MUST not be allowed to affect any single existing row or item more than once; a cardinality
violation error MUST be raised when this situation arises. Rows resulting from insertion after conflict, should not 
duplicate each other in terms of attributes constrained by an arbiter index or constraint. 
See *ON CONFLICT DO REPLACE Examples section* for more clarification.

#### ON CONFLICT DO UPDATE Examples

##### Example 4.2.1

Inserts or updates new distributors in a relational database. Assumes a unique index has been defined that constrains
values appearing in the `did` column. Note that the special `excluded` table is used to reference values originally
proposed for insertion.

```SQL
INSERT INTO distributors
VALUES (5, 'Gizmo Transglobal'),
       (6, 'Associated Computing, Inc') ON CONFLICT (did) DO
UPDATE SET dname = EXCLUDED.dname;
```

```SQL
INSERT INTO distributors AS e
VALUES
    (5, 'Gizmo Transglobal'),
    (6, 'Associated Computing, Inc')
ON CONFLICT (did) DO
UPDATE SET e.dname = e.dname;
```

##### Example 4.2.2

Inserts or updates an item into a NoSQL database. Assumes a unique constraint like primary key has been violated for an
existing item. In this case `DO UPDATE SET` updates the item with an additional attribute.

```SQL
-- Existing Item with HK as primary key:
--  {HK: 1, RK: 1}
-- Item after the update:
--  {HK: 1, RK: 1, myAttr: 1}
INSERT into Customers
VALUES
    <<{HK: 1, RK: 1}>>
ON CONFLICT DO
UPDATE SET {HK: 1, RK: 1, myAttr: 1}
```

##### Example 4.2.3

Inserts or updates an item into NoSQL database. Assumes a unique constraint like primary key has been violated for an
existing item. In this case `DO UPDATE SET` updates the item with the value specified in `INSERT`.

```SQL
-- Existing Item with HK as primary key:
--  {HK: 1, RK: 1, myAttr: 10}

-- Item after the update:
--  {HK: 1, RK: 1, myAttr: 12, anotherAttr: "Hello"}

INSERT into Customers
VALUES
    <<{HK: 1, RK: 1, myAttr: 12, anotherAttr: "Hello"}>>
ON CONFLICT DO
UPDATE EXCLUDED
```

##### Example 4.2.4

Insert or update two items into a NoSQL database. Assumes a unique constraint like primary key has been violated for an
existing item. In this case the upsert fails as `EXCLUDED` references an attribute which is non-existent for the item
that is conflicting.

```SQL
-- Existing Item with HK as primary key:
--  {HK: 1, RK: 1, myAttr: 10}
-- Item after the update*:
--  {HK: 1, RK: 1, myAttr: 10}
INSERT into Customers
VALUES
    <<{HK: 4, RK: 1, someAttr: "Foo"},
    {HK: 1, RK: 1, myAttr: 12, anotherAttr: "Hello"}>>
ON CONFLICT DO
UPDATE SET myAttr = EXCLUDED.someAttr, newAttr = 'World';
```

_In the above example before and after items are the same because an error is raised._

##### Example 4.2.5

Insert or update an item into a NoSQL database. Assumes a unique constraint like primary key has been violated for an
existing item. In this case `DO UPDATE SET` updates the item with an additional attribute and keeps the previous
attributes.

```SQL
-- Existing Item with HK and RK form a composite primary key:
--  {HK: 1, RK: 1, myAttr: 10}
-- Item after the update:
--  {HK: 1, RK: 1, myAttr: 12, newAttr = 'World'}
INSERT into Customers
VALUES
    <<{HK: 1, RK: 1, myAttr: 12, anotherAttr: "Hello"}>>
ON CONFLICT DO
UPDATE SET myAttr = EXCLUDED.myAttr, newAttr = 'World';
```

##### Example 4.2.6

Insert or update an item into a NoSQL database. Assumes a unique constraint like primary key has been violated for an
existing item. In this case `DO UPDATE SET` does not update the existing item because the condition in `WHERE` clause is
not met:

```SQL
-- Existing Item with HK as primary key:
--  {HK: 1, RK: 1, myAttr: 10}
-- Item after the execution:
--  {HK: 1, RK: 1, myAttr: 10}
INSERT into Customers AS CX
VALUES
    <<{HK: 1, RK: 1, myAttr: 12, anotherAttr: "Hello"}>>
ON CONFLICT DO
UPDATE SET myAttr = EXCLUDED.myAttr, newAttr = 'World'
WHERE CX.myAttr > 10;
```

#### ON CONFLICT DO REPLACE Examples

For the following examples we assume the existence of composite key `partition_key_sort_key` comprising `HK` and `RK`
attributes i.e `HK_RK`.

##### Example 4.2.7

The following example leads to a `SemanticError` exception because `sort_key` is absent from the value specified by
the `REPLACE`.

```SQL
-- Existing Item is {HK: 1, RK: 1, myAttr: 12}
-- Outcome is a `SemanticError` with the existing item being intact.
INSERT into Customers
VALUES
    <<{HK: 1, RK: 1, myAttr: 12, anotherAttr: "Hello"}>>
ON CONFLICT DO REPLACE {HK: 1, thirdAttr: "World"};
```

##### Example 4.2.8

The following example leads to a `SemanticError` exception because partition key is absent from the value specified by
replace.

```SQL
-- Existing Item is {HK: 1, RK: 1, myAttr: 12}
-- Outcome is a `SemanticError` with the existing item being intact.
INSERT into Customers
VALUES
    <<{HK: 1, RK: 1, myAttr: 12, anotherAttr: "Hello"}>>
ON CONFLICT DO REPLACE {RK: 1, thirdAttr: "World"};
```

##### Example 4.2.9

The following example leads to a `SemanticError` exception because both partition key and sort_key are absent in the
update statement:

```SQL
-- Existing Item is { HK: 1, RK: 1, myAttr: 12 }
-- Outcome is a SemanticError
INSERT into Customers
VALUES
    <<{HK: 1, RK: 1, myAttr: 12, anotherAttr: "Hello"}>>
ON CONFLICT DO REPLACE {thirdAttr: "World"};
```

##### Example 4.2.10

The following example leads to replacement of the item specified by replace with the existing one:

```SQL
-- Existing Item:
--  {HK: 1, RK: 1, myAttr: 12 }
-- Outcome:
--  {HK: 1, RK: 1, thirdAttr: "World"}
INSERT into Customers
VALUES
    <<{HK: 1, RK: 1, myAttr: 12, anotherAttr: "Hello"}>>
ON CONFLICT DO REPLACE {HK: 1, RK: 1, thirdAttr: "World"};
```

##### Example 4.2.11

The following example SHOULD* lead to replacing the existing item with the item specified by `REPLACE` as a new item:

```SQL
-- Existing items:
--  {HK: 1, RK: 1, myAttr: 12 }
--  {HK: 1, RK: 2, myAttr: 12 }
-- Outcome:
--  {HK: 1, RK: 3, thirdAttr: “World”}
--  {HK: 1, RK: 2, myAttr: 12 }
INSERT into Customers
VALUES
    <<{HK: 1, RK: 1, myAttr: 12, anotherAttr: "Hello" }>>
ON CONFLICT DO REPLACE {HK: 1, RK: 3, thirdAttr: "World"};
```

**  SHOULD, because for a distributed database system, implementing this logic may lead to data inconsistency. This is
because there is no guarantee that the INSERT procedure that generates the conflict executes on the same database host
where the conflict is going to get resolved.

##### Example 4.2.12

The following example MUST lead to a `SemanticError` if there is an existing item that conflicts with the item specified
by replace:

```SQL
-- Existing items:
--  {HK: 1, RK: 1, myAttr: 12 }
--  {HK: 1, RK: 2, myAttr: 12 }
-- Outcome is Error
INSERT into Customers
VALUES
    <<{ HK: 1, RK: 1, myAttr: 12, anotherAttr: "Hello" }>>
ON CONFLICT DO REPLACE {HK: 1, RK: 2, thirdAttr: "World"};
```

## 5. Other Examples

To insert multiple rows as bag value using the multi–row `VALUES` syntax:

```SQL
INSERT INTO films
VALUES
    <<{code: 'B6717', title: 'Tampopo', did: 110},
    {code: 'HG120', title: 'The Dinner Game', did: 140}>>;
```

Insert or update new distributors as appropriate. Assumes a unique index has been defined that constrains values
appearing in the `did` attribute. Note that the special `excluded` table is used to reference values originally proposed
for insertion:

```SQL
INSERT INTO distributors (did, dname)
VALUES (5, 'Gizmo Transglobal'),
       (6, 'Associated Computing, Inc') ON CONFLICT (did) DO
UPDATE SET dname = EXCLUDED.dname;
```

Insert or update new distributors values as bag value. Assumes the database engine implementation can infer the
primary key for `<conflict target>`. Note that the special `excluded` table is used to reference values originally proposed
for insertion:

```SQL
INSERT INTO distributors
VALUES
    <<{did: 5, dname: 'Gizmo Transglobal'},
    {did: 6, dname: 'Associated Computing, Inc'}>>
ON CONFLICT DO
UPDATE SET dname = EXCLUDED.dname;
```

Insert distributors values as bag value. In case of a conflict on unique constraint updates the values as stated in
the insert. Assumes the database engine implementation can infer the unique index as <conflict target>. Note that the
special `excluded` table is used to reference values originally proposed for insertion:

```SQL
INSERT INTO distributors
VALUES
    <<{did: 5, dname: 'Gizmo Transglobal'},
    {did: 6, dname: 'Associated Computing, Inc'}>>
ON CONFLICT DO
UPDATE SET EXCLUDED;
```

## References

* [Database Systems: The Complete Book, second Edition](https://www.goodreads.com/book/show/6608398-database-systems), Hector Garcia-Molina Jeffrey D. Ullman Jennifer Widom
  *Department of Computer Science Stanford University.*
* [Designing Data-Intensive Applications](https://www.goodreads.com/book/show/23463279-designing-data-intensive-applications), by Martin Kleppmann
* [Postgres INSERT ref](https://www.postgresql.org/docs/current/sql-insert.html)
* [Postgres constraints](https://www.tutorialspoint.com/postgresql/postgresql_constraints.htm)
* [BNF-EBNF Grammars](https://matt.might.net/articles/grammars-bnf-ebnf/)
* [SQL 1992](https://www.contrib.andrew.cmu.edu/~shadow/sql/sql1992.txt)

## Appendices

#### 1. Grammar Definition Syntax:

```
   < >   Angle brackets delimit character strings that are the names
         of syntactic elements, the non-terminal symbols of the SQL
         language.

   ::=   The definition operator. This is used in a production rule to
         separate the element defined by the rule from its definition.
         The element being defined appears to the left of the opera-
         tor and the formula that defines the element appears to the
         right.

   [ ]   Square brackets indicate optional elements in a formula. The
         portion of the formula within the brackets may be explicitly
         specified or may be omitted.

   { }   Braces group elements in a formula. The portion of the for-
         mula within the braces shall be explicitly specified.

   |     The alternative operator. The vertical bar indicates that
         the portion of the formula following the bar is an alterna-
         tive to the portion preceding the bar. If the vertical bar
         appears at a position where it is not enclosed in braces
         or square brackets, it specifies a complete alternative for
         the element defined by the production rule. If the vertical
         bar appears in a portion of a formula enclosed in braces or
         square brackets, it specifies alternatives for the contents
         of the innermost pair of such braces or brackets.

    ...  The ellipsis indicates that the element to which it applies
         in a formula may be repeated any number of times. If the el-
         lipsis appears immediately after a closing brace "}", then it
         applies to the portion of the formula enclosed between that
         closing brace and the corresponding opening brace "{". If
         an ellipsis appears after any other element, then it applies
         only to that element.

[, ...]  Is a short hand for ellipsis applied to a previous item enclosed by
         []. E.g. { <query> | <attr name> } [, { <query> | <attr name> } ]...
         is equal to { <query> | <attr name> } [, ...]
```

#### 2. PostgresQL query examples

1. ON CONFLICT using ON CONSTRAINT:
    1. https://www.db-fiddle.com/f/pdzztVpZqAreALu5Wquwrw/0
    2. https://www.db-fiddle.com/f/pdzztVpZqAreALu5Wquwrw/1 (with same primary key)
2. UPSERT with auto-increment postgres:
    1. https://www.db-fiddle.com/f/gWBLQ4NXhhxu7KTg9M6JwB/0
    2. https://www.db-fiddle.com/f/gWBLQ4NXhhxu7KTg9M6JwB/2 (with existing key in UPDATE)
    3. https://www.db-fiddle.com/f/gWBLQ4NXhhxu7KTg9M6JwB/3 (replacing an existing item)
3. UPSERT with EXCLUDED in postgres:
    1. https://www.db-fiddle.com/f/nP56Wo66Qj1Bswsx3a8GxU/0
4. UPSERT with multiple values in postgres:
    1. https://www.db-fiddle.com/f/nP56Wo66Qj1Bswsx3a8GxU/1
    2. https://www.db-fiddle.com/f/nP56Wo66Qj1Bswsx3a8GxU/2
5. ON CONFLICT Error for a table with no constraints:
    1. https://www.db-fiddle.com/f/gWBLQ4NXhhxu7KTg9M6JwB/6

# Drawbacks

[drawbacks]: #drawbacks

Why should we *not* do this?

One reason for not taking this approach is the statement complexity for `UPSERT` and `REPLACE` scenarios. Considering
this, having the `INSERT` statement incorporating both `UPSERT` and `REPLACE` leaves the room open for rewrites from
other statements designated for `UPSERT` and `REPLACE` separately.

# Rationale and alternatives

[rationale-and-alternatives]: #rationale-and-alternatives

- Why is this design/proposal the best in the space of possible designs? The `INSERT INTO ... ON CONFLICT` statement is
  chosen based on the assessment of both relational and non-relational use-cases and the need for being able to upsert
  in the absence of a source table. We believe `INSERT INTO ... ON CONFLICT` as adopted by PostgresSQL and SQLite allows
  for incorporating attribute level and row-level atomic operations with the option of re-using the values used for
  insertion (by using EXCLUDED keyword). On the other
  hand, [Merge](https://en.wikipedia.org/wiki/Merge_%28SQL%29#Other_non-standard_implementations) as defined by SQL
  standard require a table as a source which might not be required in all cases. In addition, arguably, among other
  alternatives there isn't necessarily a "best" option available. Finally, this is not a one-way door decision because
  so long as the semantics remain the same, other statements can be added to the language with possible rewrites to this
  base syntax.

- Which other designs/proposals have been considered, and what is the rationale for not choosing them?
  SQL [Merge](https://en.wikipedia.org/wiki/Merge_%28SQL%29#Other_non-standard_implementations) statement has been
  considered but considering that it requires a source table it has been discarded.

- What is the impact of not doing this? No service level or production impact as this is an addition to the PartiQL
  specification.

# Prior art

[prior-art]: #prior-art

Discuss prior art, both the good and the bad, in relation to this proposal. A few examples of what this can include are:

- For specification proposals: Does this feature exist in any ISO SQL standard or other SQL dialects? As mentioned in
  prior section, the `INSERT INTO ... ON CONFLICT` has already been used in PostgresSQL and SQLite. Generally
  the `UPSERT` functionality which is formally known
  as [Merge](https://en.wikipedia.org/wiki/Merge_%28SQL%29#Other_non-standard_implementations)
  already exists in SQL Standard and has been implemented in various databases systems—both relational as NoSQL systems.

# Unresolved questions

[unresolved-questions]: #unresolved-questions

# Future possibilities

[future-possibilities]: #future-possibilities

In the future, we may be able to accommodate `DELETE` conflict action which specifies the delete operation if a conflict 
is raised. In addition, we see addition of specification for other syntactic variation of the semantics specified by this
RFC using designated syntax for `UPSERT` and `REPLACE` operations separately. This is to provide more ergonomic choices 
for implementations that require to adopt the corresponding semantics as separate statements.
