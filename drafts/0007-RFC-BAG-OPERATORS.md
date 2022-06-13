- Feature Name: Bag Operators: UNION, INTERSECT, EXCEPT
- Start Date: 2022-06-07

# Summary
[summary]: #summary

This RFC defines the PartiQL specification for the operators: UNION, INTERSECT, and EXCEPT. The behavior of each operator conforms to the SQL standard in strict typing mode. In the absence of schemas, each operator is equivalent to the mathematical definitions of the respective multi-set operator where member equality is consistent with the equality in the *GROUP BY* clause.

# Motivation
[motivation]: #motivation

The operators UNION, INTERSECT, and EXCEPT have been defined since SQL-92 specifications, yet these are missing from the PartiQL spec. In SQL, these operators are the typical bag (set if DISTINCT) union, intersect, and difference respectively for *union-compatible* relations. In the presence of schemas, the PartiQL set operators are compatible with the ISO SQL specification for set operators; however, PartiQL extends these operators to allow for combining arbitrary values such as bags of heterogenous tuples. SQL compatbility is discussed in the following section, and a detailed look at SQL set operators can be found in Appendix A.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## Definition

Each bag operator has the form *q S q'* where *q* and *q'* are of type *\<query\>* and *S* is the operator. Additionally, the operator may be suffixed with *ALL* which indicates the output may have duplicate elements. In its absence, duplicates are eliminated from the final result. Arguments of the operators are coerced into bags with the following behavior.  

- *NULL* and *MISSING* become the empty bag *<< >>*
- A scalar value *v* becomes the singleton bag *<< v >>*
- An array is coerced to a bag by discarding ordering

**Figure 1**

Let *A* and *B* be bags and *x* be an element in a bag. Define *M(x, A)* as the multiplicity of *x* in some bag *A*.
```
UNION_ALL(A, B) = A MULTISET UNION B

INTERSECT_ALL(A, B):
  - If A or B is << >>, then << >>
  - Else << x >> with multiplicity m, for all x in A with x in B and m = min(M(x, A), M(x, B))

EXCEPT_ALL(A, B):
  - If A is << >>, then << >>
  - If B is << >>, then A
  - Else << x >> with multiplicity m, for all x in A with m = max(M(x, A) - M(x, B), 0)

UNION(A, B) = DISTINCT(UNION_ALL(A, B))

INTERSECT(A, B) = DISTINCT(INTERSECT_ALL(A, B))

EXCEPT(A, B) = DISTINCT(EXCEPT_ALL(A, B))
```

In strict-typing mode, the set operators behave just like the set operators defined in the SQL specification as discussed in the next section. Outside of strict-typing mode, these operators are the pure mathematical multi-set union, intersect, and except which enables the creation and combination of heterogneous bags (TODO polish this sentence).

### SQL Compatibility

In the presence of schema, the column descriptors for a table are known. These column descriptors are an ordered list of metadata objects which describe a column's: type, name, nullability, and more (see Appendix A). The column descriptors play an important role for set operators. Two relations *T1* and *T2* are said to be *union-compatible* if they have the same number of columns (\*) and the data type for the i-th column of *T1* and the data type for the i-th column of *T2* are comparable. Comparability and the resultant data type follow the same rules as the comparability rules and data types of results of aggregations defined in SQL-99 section 9.3.  

Each set operator produces a new relation *TR* which has its own column descriptors. The exact details of the these column descriptors is discussed in Appendix A, as well as SQL-99 section 7.12. The specification states that, for SQL conformance:

**Column Naming Rule**
- If the i-th columns of *T1* and *T2* share the name *C*, then the i-th column of *TR* will be named *C*.
- Otherwise, the name of the i-th column of *TR* is implementation-dependent and must be unique amongst all other column names.

**Nullability Rule**
- The i-th column of *TR* is nullable only if both the i-th columns of *T1* and *T2* are nullable.
- If the set operator is *EXCEPT*, then the i-ith column of *TR* is nullable only if the i-th column of *T1* is nullable.

These rules tell us, in the presence of schema, the set operators are valid when the argument relations are union-compatible; and the resultant column descriptors inherit their names and nullability from the column descriptors of the arguments.

- \* — If the statement contains a *CORRESPONDING* clause, then the column descriptors of *T1* and *T2* will be compared by the delcared order of column names in the *CORRESPONDING* list.

## Examples

### Guide

#### Union
1. Homogenous Bags Permissive Typing
2. Homogenous Bags Strict Typing
3. Heterogenous Bags Strict Typing
4. Heterogenous Bags Permissive Typing

#### Intersect, Except, and Special Cases
1. Basic Intersect
2. Basic Except
3. Singleton bag coercion
4. NULL and MISSING coercion

### Environment

```
-- Each bag of employees is *almost* the same! Let's see how we can compose them.
!add_to_global_env { 
  'hr': { 
    'employees': <<
      { 'name': 'Bob Smith',   'title': 'Consultant' }, 
      { 'name': 'Susan Smith', 'title': 'Manager' },
      { 'name': 'Jane Smith',  'title': 'DEI'}
    >>
  },
  'accounting': { 
    'employees': <<
      { 'level': 3, 'name': 'Quincy Jones',   'title': 'Director' }, 
      { 'level': 2, 'name': 'James Earl Jones', 'title': 'Manager' },
      { 'level': 1, 'name': 'Indiana Jones',  'title': 'Intern'}
    >>
  },
  'engineering': { 
    'employees': <<
      { 'title': 'Manager', 'name': 'Paul McCharmly' }, 
      { 'title': 'SDE', 'name': 'George Parasol' },
      { 'title': 'Principal SDE', 'name': 'Ringo Stone' },
      { 'title': 'Intern', 'name': 'Eric Lennon' }
    >>
  }
} 
```

### Strict Mode Example Column Descriptors

In strict-typing mode, we can define the *hr.employees, accounting.employees, engineering.employees* relations to have the following schemas.

```
              hr.employees

| Ordinal | Name  | Type   | Nullable |
|---------|-------|--------|----------|
| 0       | NAME  | String | NO       |
| 1       | TITLE | String | NO       |

        accounting.employees

| Ordinal | Name  | Type   | Nullable |
|---------|-------|--------|----------|
| 0       | LEVEL | Int    | NO       |
| 1       | NAME  | String | NO       |
| 2       | TITLE | String | NO       |

        engineering.employees

| Ordinal | Name  | Type   | Nullable |
|---------|-------|--------|----------|
| 0       | TITLE | String | NO       |
| 1       | NAME  | String | NO       |
```

### Union Examples

**Union 1** — Homogenous Bags — Both strict and permissive typing would produce this result.
```sql
SELECT name, title FROM hr.employees
UNION ALL
SELECT name, title FROM engineering.employees

<<
  {
    'name': 'Bob Smith',
    'title': 'Consultant'
  },
  {
    'name': 'Susan Smith',
    'title': 'Manager'
  },
  {
    'name': 'Jane Smith',
    'title': 'DEI'
  },
  {
    'name': 'Paul McCharmly',
    'title': 'Manager'
  },
  {
    'name': 'George Parasol',
    'title': 'SDE'
  },
  {
    'name': 'Ringo Stone',
    'title': 'Principal SDE'
  },
  {
    'name': 'Eric Lennon',
    'title': 'Intern'
  }
>>
-- OK!
```

**Union 2** — Homogenous Bags with strict typing

Here is **union 1** with strict typing, but the **names** of the column descriptors do not match. Notice the lack of selection list here. These two relations are union-compatible because the i-th column of *hr.employees* is comparable to the i-th column of *engineering.employees*, but notice the column names. This behavior conforms to SQL specification whereby, if the column names do not match, then the column name is implementation-dependent i.e. it is not derived from any particular value.

```sql
SELECT * FROM hr.employees
UNION ALL
SELECT * FROM engineering.employees

<<
  {
    '_0': 'Bob Smith',
    '_1': 'Consultant'
  },
  {
    '_0': 'Susan Smith',
    '_1': 'Manager'
  },
  {
    '_0': 'Jane Smith',
    '_1': 'DEI'
  },
  {
    '_0': 'Manager',
    '_1': 'Paul McCharmly'
  },
  {
    '_0': 'SDE',
    '_1': 'George Parasol'
  },
  {
    '_0': 'Principal SDE',
    '_1': 'Ringo Stone'
  },
  {
    '_0': 'Intern',
    '_1': 'Eric Lennon'
  }
>>
-- OK!

-- Result Column Descriptors
-- | Ordinal | Name  | Type   | Nullable |
-- |---------|-------|--------|----------|
-- | 0       | _0    | String | NO       |
-- | 1       | _1    | String | NO       |
```

Notice that we have some issues with the meaning of our keys/columns. For the first three structs/tuples, key *_0* contains a name and *_1* contains a title; yet it is the opposite for the last four structs/tuples. We can fix this by using the SQL *CORRESPONDING* clause.

```sql
SELECT * FROM hr.employees
UNION ALL CORRESPONDING BY ( name, title )
SELECT * FROM engineering.employees

-- Equivalent to *Union 1*

SELECT name, title FROM hr.employees
UNION ALL
SELECT name, title FROM engineering.employees
```

**Union 3** — Heterogenous Bags with Strict Typing

Here we attempt to union the job titles in HR and Engineering, but there is a mistake. In the absence of schema, the bags produced by *T1* and *T2* are unioned as two bags of structs, but in strict-typing mode, we need to look at the column descriptors. This behavior conforms to the SQL specification.

```sql
SELECT * FROM hr.employees               -- T1
UNION
SELECT title FROM engineering.employees  -- T2
-- ERROR!
-- T1 and T2 are not union-compatible in strict-typing mode
```

```
       Column Descriptors of T1                          Column Descriptors of T2 

| Ordinal | Name  | Type   | Nullable |           | Ordinal | Name  | Type   | Nullable |       
|---------|-------|--------|----------|           |---------|-------|--------|----------|
| 1       | NAME  | String | NO       |  >UNION<  | 0       | TITLE | String | NO       |
| 2       | TITLE | String | NO       |
```

**Union 4** — Heterogenous Bag with Permissive Typing

Let's do a simple union that would cause confusion in strict mode (**Union 2**), but is no problem for permissive mode. Without schemas, the union operators is a pure multi-set union. This shows how PartiQL's UNION operator's functionality is a superset of the SQL UNION operator.

```sql
TABLE hr.employees UNION TABLE engineering.employees -- same as Union 2 example, but permissive mode

<<
  {
    'name': 'Bob Smith',
    'title': 'Consultant'
  },
  {
    'name': 'Susan Smith',
    'title': 'Manager'
  },
  {
    'name': 'Jane Smith',
    'title': 'DEI'
  },
  {
    'title': 'Manager',
    'name': 'Paul McCharmly'
  },
  {
    'title': 'SDE',
    'name': 'George Parasol'
  },
  {
    'title': 'Principal SDE',
    'name': 'Ringo Stone'
  },
  {
    'title': 'Intern',
    'name': 'Eric Lennon'
  }
>>
-- OK!
```

Notice how the structs keep their keys! In permissive mode, there's no notion of "i-th column descriptor" so we do not compare them! The UNION is the simple multi-set union of the structs in the *hr.employees* and *engineering.employees* bags.

### Intersect, Except, and Special Case Examples

**Basic Intersect**
```sql
SELECT title FROM hr.employees INTERSECT SELECT title FROM engineering.employees

<<
  {
    'title': 'Manager'
  }
>>
-- OK!
```

**Basic Except** — Job titles unique to the engineering department.
```sql
SELECT title FROM engineering.employees EXCEPT (
  SELECT title FROM hr.employees
  UNION
  SELECT title FROM accounting.employees
)

<<
  {
    'title': 'SDE'
  },
  {
    'title': 'Principal SDE'
  }
>>
-- OK!
```

**Special Cases**
```sql
-- result is the same as `TABLE engineering.employees`
SELECT * FROM engineering.employees EXCEPT << >> 

-- result is the empty bag
SELECT * FROM engineering.employees INTERSECT << >>

-- Coercion of single value
SELECT * FROM << 1 >> UNION 'A'
<<
  {
    '_1': 1
  },
  'A'
>>
-- OK!
```

# Drawbacks
[drawbacks]: #drawbacks

We do not recognized drawbacks to implementing these features as these are clearly defined in the specification yet missing from the implementation.

TODO: Stating no drawbacks might come off as naive.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

The proposal is based upon the bag operators of SQL++ [1]. These operators define the extension of SQL bag operators for heterogenous, semi-structured data. Implementing these operators in required for ISO SQL compliance so there are no alternatives.

[1] *The SQL++ Unifying Semi-structured Query Language*: Kian Win Ong, Yannis Papakonstantinou, Romain Vernoux.

# Prior art
[prior-art]: #prior-art

As previously covered, these operators are defined in both the ISO SQL spec as well as the SQL++ paper [1]. Few databases implement these, but their implementations for *UNION ALL* showcase the handling of three problems:

1. Collection Mismatch
2. Coercion Behavior
3. Equality

With respect to the above, PartiQL has chosen to produce heterogenous bags in permissive mode and homogenous bags or error in conventional typing mode. PartiQL has chosen to coerce NULL and MISSING to the empty bag; but allows for a singleton bag of NULL or MISSING. Finally, equality in bag operators is consistent with equality used in the *GROUP BY* clause.

The following feature matrix is Table 12 of from the SQL++ paper.

**Feature Matrix for Bag Operators**

| Database | UNION ALL | INTERSECT ALL | EXCEPT ALL | UNION | INTERSECT | EXCEPT |
| -------- | --------- | ------------- | ---------- | ----- | --------- | ------ |
| SQL      | [x]       | [x]           | [x]        | [x]   | [x]       | [x]    |
| Hive     | [x]       | [ ]           | [ ]        | [ ]   | [ ]       | [ ]    |
| Jaql     | [x]       | [ ]           | [ ]        | [ ]   | [ ]       | [ ]    |
| Pig      | [x]       | [ ]           | [ ]        | [ ]   | [ ]       | [ ]    |
| CQL      | [ ]       | [ ]           | [ ]        | [ ]   | [ ]       | [ ]    |
| JSONiq   | [x]       | [ ]           | [ ]        | [ ]   | [ ]       | [ ]    |
| MongoDB  | [ ]       | [ ]           | [ ]        | [ ]   | [ ]       | [ ]    |
| N1QL     | [ ]       | [ ]           | [ ]        | [ ]   | [ ]       | [ ]    |
| AQL      | [ ]       | [ ]           | [ ]        | [ ]   | [ ]       | [ ]    |
| BigQuery | [x]       | [ ]           | [ ]        | [ ]   | [ ]       | [ ]    |

TODO: I don't think the feature matrix is that helpful. Would a better use of this section for describing *how* the other implementations behave? This is covered in [1].

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- TODO FILL IN FROM PR REVIEWS


# Future possibilities
[future-possibilities]: #future-possibilities

We can consider introducing extended bag operators for the key-value pairs of Ion structs. These extended operators would allow for joining or combining two structs key-value pairs. Keys are unique in structs, so we must make a decision on how to behave when a key appears in both the left and right operands.

Let *S* and *S'* be two structs. Let *k* be a key in both *S* and *S'* with values *s* and *s'* respectively. If a key does not appear in one of *S* or *S'*, then it is treated as *MISSING*. The operators could be defined as below.

```
S UNION S'     -> { k: s UNION s' }
S INTERSECT S' -> { k: s INTERSECT s' }
S EXCEPT S'    -> { k: s EXCEPT s' }
```

For example,

Without this extension, UNION would behave like so. Each operand is coerrced into a singleton bag.
```
SELECT * FROM { 'A': 0, 'B': 1 }
UNION
SELECT * FROM { 'A': 2, 'B': 3, 'C': 4 }
   |
==='
<<
  {
    'A': 0,
    'B': 1
  },
  {
    'A': 2,
    'B': 3,
    'C': 4
  }
>>
---
OK!
```

Extending UNION to Ion structs would behave like so.

```
SELECT * FROM { 'A': 0, 'B': 1 }
UNION
SELECT * FROM { 'A': 2, 'B': 3, 'C': 4 }
   |
==='
<<
  {
    'A': << 0, 2 >>,
    'B': << 1, 3 >>,
    'C': << 4 >>
  }
>>
---
OK!
```

For INTERSECT and EXCEPT, if the value result is `<< >>`, then the key is removed from the resultant struct. Consider the following

```
SELECT * FROM { 'A': 0, 'B': 1 }
INTERSECT
SELECT * FROM { 'A': 2, 'B': 3, 'C': 4 }
   |
==='
<<
  {
    'A': << >>,
    'B': << >>,
    'C': << >>
  }
>>
---
OK!

-- Which becomes
   |
==='
<<
  {}
>>
---
OK!

SELECT * FROM { 'first_names': ['jack', 'jill'], 'last_names': ['smith', 'jones', 'johnson'] }
INTERSECT
SELECT * FROM { 'first_names': ['jack', 'john'], 'last_names': ['smith', 'snow'] }
   |
==='
<<
  {
    'first_names': << 'jack' >>,
    'last_names': << 'smith' >>
  }
>>
---
OK!
```

EXCEPT examples,
```
SELECT * FROM { 'A': 0, 'B': 1 } EXCEPT { 'A': 2, 'B': 3, 'C': 4 }
   |
==='
<<
  {
    'A': << 0 >>,
    'B': << 1 >>
  }
>>
---
OK!

SELECT * FROM { 'A': 0, 'B': 1 }
EXCEPT
SELECT * FROM { 'A': 0, 'B': 3 }
   |
==='
<<
  {
    'B': << 1 >>
  }
>>
---
OK!

SELECT * FROM { 'first_names': ['jack', 'jill'], 'last_names': ['smith', 'jones', 'johnson'] }
EXCEPT
SELECT * FROM { 'first_names': ['jack'], 'last_names': ['smith'] }
   |
==='
<<
  {
    'first_names': << 'jill' >>,
    'last_names': << 'jones', 'johnson' >>
  }
>>
---
OK!
```


# Appendices

## Appendix A — SQL Set Operators

 http://web.cecs.pdx.edu/~len/sql1999.pdf

### Goal
The goal of this document is to clearly define the behavior of column descriptors for SQL-99 conformant set operators.

### Definitions

**Definition**: Query Expression (p265) — For the purposes of this document, *<with_clause>* clause is omitted.

```
<query_expression> ::= [<with_clause] <query_expression_body>

<query_expression_body> ::=
    <non-join_query_expression>
  | <joined_table>

<non-join-query_expression> ::=
    <non-join_query_term>
  | <query_expression_body> UNION  [ALL|DISTINCT] [corr_spec>] <query_term>
  | <query_expression_body> EXCEPT [ALL|DISTINCT] [corr_spec>] <query_term>

<query-term> ::=
    <non-join_query_term>
  | <joined_table>

<non-join_query_term> ::=
    <non-join_query_primary>
  | <query_term> INTERSECT [ALL|DISTINCT] [corr_spec] <query_primary>

<query_primary> ::=
    <simple_table>
  | <joined_table>

<non-join_query_primary> ::=
    <simple_table>
  | ( <non-join_query_expression> )

<simple_table> ::=
    <query_specification>      # SFW
  | <table_value_constructor>  # VALUES ( ... )
  | TABLE <table_name>

<corr_spec> ::= CORRESPONDING [ BY ( <column_name_list> )]
```

**Definition**: Set Operators (p269) — SQL-99 defines the set operators as
- UNION ALL
- UNION DISTINCT
- EXCEPT ALL
- EXCEPT DISTINCT
- INTERSECT ALL
- INTERSECT DISTINCT.

If the set quantifier (ALL, DISTINCT) is not specified, then DISTINCT is implicit. 

**Definition**: Column Descriptor (p41) — A column C is described by a *column descriptor* which includes
- The name of the column
- Whether the name of the column is an implementation-dependent name
- The data type descriptor of the declared type of C
- The default value (if any) of C
- The nullability characteristic of C
- The ordinal position of C within the table that contains it
- An indication of whether the column is a self-referencing column of a base table or not

### Rules for Column Descriptors

The following rules describe what the column descriptors are for various types of *<query_expression_body>*. It is important to understand the column descriptors of the operands because these descriptors are used as the descriptors for the resultant relation of a set operator. 

**Rule 1**

If a *<simple_table>* is a *<query_spec>*, then the column descriptor of the i-th column of the *<simple_table>* is the same as the column descriptor of the i-th column of the *<query_spec>*.

**Rule 2**

If a *<simple_table>* is an *<explicit_table>* (`SELECT * FROM T`), then the column descriptor of the i-th column of the *<simple_table>* is the same as the column descriptor of the i-th column of the table identified by the *<explicit_table>*

**Rule 3**

If a *<simple_table>* is a *<table_value_constructor>*, then the column descriptor of the i-th column of the *<simple_table>* is the same as the column descriptor of the i-th column of the *<table_value_constructor>*, except that the column name of the column descriptor is implementation-dependent and not equivalent to the column name of any column, other than itself, of any table referenced by a *<table_reference>* contained in the outermost SQL-statement.

**Rule 4**
If a *<non-join_query_primary>* is a *<simple_table>*, then its column descriptors are the same as the *<simple_table>*. If a *<query_primary>* is a *<non-join_query_primary>*, the column descriptors are the same as the column descriptors of the *<non-join_query_primary>*. 

### Rules for Set Operators

Let *T1* and *T2* be the first and second operands of a set operator, and let *TR* be the result. 

**Rule 1**
If *CORRESPONDING* is sepcified, then
1. The column name list cannot contain duplicates
2. At least one column name of T1 must be in T2
3. If no names are specified in the list, then the list is all the column names of T1.
4. Every column name in the list must be a column name of T1 and T2.

**Rule 2**
With a corresponding clause, T1 and T2 must have the same number of columns.

**Rule 3**
If the i-th column of T1 and T2 have the same name *C*, then the i-th column of TR has the name *C*.

**Rule 4**
If the i-th column of T1 and T2 does not have the same name, then the name of i-th column of TR is implementation-dependent and unique among all other column names.

## Appendix B

Aligning PartiQL parser implementation with the PartiQL specification. Right now, the parser will parse the arguments of each set operator as an expression, and the parser does not support the corresponding clause. Here is what we should consider updating our specifcation grammar to. This uses the ANTLR syntax with rules in *\< \>* for clarity.

```
query
  : <sfw_query>
  | <expr_query>
  ;

sfw_query
  : (WITH <query> AS <variable>)>
    <select>
    <from>
    (WHERE <expr_query>)?
    (GROUP BY <expr_query> (AS <variable>)? (',' <expr_query> (AS <variable)?)*)?
    (HAVING <expr_query>)?
    ((UNION|INTERSECT|EXCEPT) (ALL|DISTINCT)? <corr_spec>? <query>)?
    (ORDER BY <sort_item> (',' <sort_item>)*)?
    (LIMIT <expr_query>)?
    (OFFSET <expr_query>)?
  ;

expr_query
  : ( <sfw_query> )
  | <path_expr>
  | <function> '(' (<expr_query> (',' <expr_query>)*)? ')'
  | '{' (<expr_query> ':' <expr_query> (',' <expr_query> ':' <expr_query>)*)? '}'
  | '[' (<expr_query> (',' <expr_query>)*)? ']'
  | '<<' (<expr_query> (',' <expr_query>)*)? '>>'
  | <sql_scalar_expr>
  | <literal>
  ;

path_expr
  : <variable>
  | '(' <expr_query> ')'
  | <path_expr> '.' <attr_name>
  | <path_expr> '.' '*'
  | <path_expr> '[' <expr_query> ']'
  | <path_expr> '[' '*' ']'
  ;

corr_spec
  : CORRESPONDING ( BY '(' <column_name> (',' <column_name>)* ')' )?
  ;
```
