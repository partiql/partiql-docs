- Start Date: (fill me in with today's date, YYYY-MM-DD)
- PartiQL Issue (fill me in with the related PartiQL issue)
- RFC PR: (fill me in with the PR link once PR is created)

# Summary
[summary]: #summary

We defined PartiQL `WINDOWED` clause and PartiQL window function to explain SQL’s window function semantics. The PartiQL `WINDOWED` clause is responsible for “window construction”, i.e., it carries the created partition, and the index of current row in the partition ( for now frame is out of the scope) in the binding tuple. And a PartiQL window function is a general function that takes in the partition and index to deliver compatible result as SQL’s window function.
The defined WINDOWED clause and partiQL window function are conceptual and used to explain the semantic. It is not required for an implementation.

# Motivation
[motivation]: #motivation

Window functions were first introduced as part of the SQL:2003. Window functions allows user to write analytical queries, such as time series analysis, moving averages, cumulative sums, and ranking. Formulating such queries in plain SQL-92 can be cumbersome if not impossible.

Since introduced, window functions are supported by major database systems, such as PostgreSQL, MySQL, Orcale, SAP HANA, etc. The motivation is to provide semantic guidance through the specification for window function implementation.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## Out of scope

* We focus on the LEAD/LAG function in this doc. Other window functions, along with the usage of aggregation functions as window function, is out of the scope for now.
* Since LEAD/LAG operates on PARTITION only, the `FRAME` concept is out of the scope for now. (To the best of our knowledge, the purposed semantics can be extended to support `FRAME` clause)  (See [POC on FRAME clause support](#poc-on-frame-clause-support)).
* SQL spec allows for the usage of additional `IGNORE NULLS` or `RESPECT NULLS` with `LAG` / `LEAD` functions along with some other window functions. For now we do not support it as only some mainstream SQL vendor support it. (See: [NULL TREATMENT Semantics](#null-treatment-semantics)).

## Proposed grammar

```
<window function> ::= <window function type> OVER <window specification> 
<window function type> ::= <lead or lag function>
<lead or lag function> ::= <lead or lag> <left paren> <lead or lag extent> 
    [ <comma> <offset> 
        [ <comma> < default expression] ] <right paren> 
<lead or lag> :: LEAD | LAG
<lead or lag extend> ::= <expr query>
<offset> :: <extract numeric literal> (* SQL spec*)
<default expresion> ::= <value expr> (* Defined in RFC-0011 *)
<window specification> ::= 
    <left paren> <window specification detail> <right paren>
<window specificaition detail> ::= 
    [ <window partition clause> ]
    [ <window order by clause> ]
    
<window partition clause> ::= PARTITION BY <window partition reference list>
<window partition reference list> ::= <expr query> [ <expr query> ... ]
<window order by clause> ::= ORDER BY <window sort specification list> 
<window sort specification list> ::= <sort specification> [ { <comma> <sort specification> } ... ]
<sort specification> ::= <sort key> [ <ordering specification> ] [ <null ordering> ]
<sort key> ::= <expr query> 
<ordering specification> ::= ASC | DESC
<null ordering> ::= NULL FIRST | NULL LAST
```

## Purposed Semantics

### WINDOWED clause

PartiQL introduces `WINDOWED` clause to extends SQL's window function semantics. The PartiQL `WINDOWED` clause can be thought of a standalone operator that inputs a collection of binding tuples and outputs a collection of binding tuples.


```
WINDOWED (
    PARTITION BY e1, e2, ..., em
    ORDER BY (o1 [ASC|DESC]? [NULLS FIRST| NULLS LAST]?,
              ...,
              on [ASC|DESC]? [NULLS FIRST| NULLS LAST]?
    ) PARTITION AS p AT pos
```

Where `e1,...,en` is a list of **partition expressions**, `o1,....,on` is a list of **ordering expressions**, `p` is the **partition variable**, and `pos` is the **position variable** which indicates the position of the current row in the partition.

The `WINDOWED clause` can be viewed as a modular function, denote the bag of input binding tuples `BinWindow`

1. PARTITION BY:

* If PARTITION BY is presented: `BinWindow` is partitioned into the minimal number of equivalence partition `B1,...,Bn`. The equivalence rule is the same as the one used in `GROUP BY` clause.
* If there is no `PARTITION BY`, the entire input binding collection is considered as one partition.
* Each of the `Bi` produced by `PARTITION BY` SHALL be an array instead of a bag, even though the order is non-deterministic at the moment.

2. ORDER BY:

* If ORDER BY is presented, each equivalence partition is ordered based on the rules defined in Spec Section 12.
* Notice that the ORDER BY in WINDOWED clause applies to partition only, it does not necessarily affect the final order of the result.

3. Windowed output:

* For each binding tuple bi in Bin_window, output b = bi || <p : Bi, pos: x> where bi in Bi and p[x] = bi.
* Notice that in case of duplication, the pos is nondeterministic but MUST be guaranteed to be unique.

Logically, windowed clause is evaluated after GROUP BY and HAVING and before ORDER BY, the evaluation order looks like:

```
FROM ...
WHERE ...
GROUP BY ...
HAVING ...
WINDOWED ...
ORDER BY ...
LIMIT ...
SELECT ...
```


Examples:

Suppose  `B_inWindow` is:

```
<<
    <l : {'sensor' : 1, 'co' : 0.4, 'time': 200}>,
    <l : {'sensor' : 1, 'co' : 0.3, 'time': 100}>,
    <l : {'sensor' : 2, 'co' : 0.1, 'time': 300}> 
>>
```



1. Window specification without PARTITION BY AND ORDER BY

```sql
WINDOWED () PARTITION AS p AT pos
```

The output binding collection is:

```
<<
   <
    l: {'sensor' : 1, 'co' : 0.4, 'time': 200},
    p : [
         <l : {'sensor' : 1, 'co' : 0.4, 'time': 200}>,
         <l : {'sensor' : 1, 'co' : 0.3, 'time': 100}>,
         <l : {'sensor' : 2, 'co' : 0.1, 'time': 300}>
        ],
    pos: 0
   >,
   <
    l : {'sensor' : 1, 'co' : 0.3, 'time': 100},
    p : [
         <l : {'sensor' : 1, 'co' : 0.4, 'time': 200}>,
         <l : {'sensor' : 1, 'co' : 0.3, 'time': 100}>,
         <l : {'sensor' : 2, 'co' : 0.1, 'time': 300}>
        ],
    pos: 1
   >
   <
    l : {'sensor' : 2, 'co' : 0.1, 'time': 300},
    p : [
         <l : {'sensor' : 1, 'co' : 0.4, 'time': 200}>,
         <l : {'sensor' : 1, 'co' : 0.3, 'time': 100}>,
         <l : {'sensor' : 2, 'co' : 0.1, 'time': 300}>
        ],
    pos: 2
   >      
>>
```

Remarks: If ORDER BY is not present, the order of partition variable p is arbitrary.


2. Window specification with PARTITION BY ONLY

```sql
WINDOWED (PARTITION BY l.sensor) PARTITION AS p AT pos
```

The output binding collection is:

```
<<
   <
    l: {'sensor' : 1, 'co' : 0.4, 'time': 200},
    p : [
         <l : {'sensor' : 1, 'co' : 0.4, 'time': 200}>,
         <l : {'sensor' : 1, 'co' : 0.3, 'time': 100}>
        ],
    pos: 0
   >,
   <
    l : {'sensor' : 1, 'co' : 0.3, 'time': 100},
    p : [
         <l : {'sensor' : 1, 'co' : 0.4, 'time': 200}>,
         <l : {'sensor' : 1, 'co' : 0.3, 'time': 100}>
        ],
    pos: 1
   >
   <
    l : {'sensor' : 2, 'co' : 0.1, 'time': 300},
    p : [
         <l : {'sensor' : 2, 'co' : 0.1, 'time': 300}>
        ]
    pos: 0
   >      
>>
```

Remark: Notice that the second equivalence partition contains only one binding tuple. Nevertheless, it is still a list.


3. Window specification with ORDER BY ONLY

```sql
WINDOWED (ORDER BY l.time) PARTITION AS p AT pos
```

The output binding collection is:

```
<<
   <
    l: {'sensor' : 1, 'co' : 0.4, 'time': 200},
    p : [
         <l : {'sensor' : 1, 'co' : 0.3, 'time': 100}>,
         <l : {'sensor' : 1, 'co' : 0.4, 'time': 200}>,
         <l : {'sensor' : 2, 'co' : 0.1, 'time': 300}>
        ],
    pos: 1
   >,
   <
    l : {'sensor' : 1, 'co' : 0.3, 'time': 100},
    p : [
         <l : {'sensor' : 1, 'co' : 0.3, 'time': 100}>,
         <l : {'sensor' : 1, 'co' : 0.4, 'time': 200}>,
         <l : {'sensor' : 2, 'co' : 0.1, 'time': 300}>
        ],
    pos: 0
   >
   <
    l : {'sensor' : 2, 'co' : 0.1, 'time': 300},
    p : [
         <l : {'sensor' : 1, 'co' : 0.3, 'time': 100}>,
         <l : {'sensor' : 1, 'co' : 0.4, 'time': 200}>,
         <l : {'sensor' : 2, 'co' : 0.1, 'time': 300}>
        ],
    pos: 2
   >      
>>
```

4. WINDOW specification with PARTITION BY AND ORDER BY

```sql
WINDOWED (PARTITION BY l.sensor ORDER BY l.time) PARTITION AS p AT pos
```

The output binding collection is:

```
<<
   <
    l: {'sensor' : 1, 'co' : 0.4, 'time': 200},
    p : [
         <l : {'sensor' : 1, 'co' : 0.3, 'time': 100}>,
         <l : {'sensor' : 1, 'co' : 0.4, 'time': 200}>
        ],
    pos: 1
   >,
   <
    l : {'sensor' : 1, 'co' : 0.3, 'time': 100},
    p : [
         <l : {'sensor' : 1, 'co' : 0.3, 'time': 100}>,
         <l : {'sensor' : 1, 'co' : 0.4, 'time': 200}>
        ],
    pos: 0
   >
   <
    l : {'sensor' : 2, 'co' : 0.1, 'time': 300},
    p : [
         <l : {'sensor' : 2, 'co' : 0.1, 'time': 300}>
        ]
    pos: 0
   >      
>>
```

5. Interaction with GROUP BY

Suppose we have:

```sql
GROUP BY l.sensor as sensor GROUP AS g
WINDOWED (PARTITION BY sensor) PARTITION AS p AT pos
```

The output binding collection is:

```
<<
   <
    sensor : 1, 
    g: <<
         <l : {'sensor' : 1, 'co' : 0.3, 'time': 100}>,
         <l : {'sensor' : 1, 'co' : 0.4, 'time': 200}>
       >>
    p: [
         <
            sensor : 1, 
            g: <<
                <l : {'sensor' : 1, 'co' : 0.3, 'time': 100}>,
                <l : {'sensor' : 1, 'co' : 0.4, 'time': 200}>
               >> 
         >, 
        ],
    pos: 0
   >,
   <
    sensor : 2,
    g: <<
         <l : {'sensor' : 2, 'co' : 0.1, 'time': 300}>
        >>
    p: [
         <
            'sensor' : 2,
             g : <<
                   <l : {'sensor' : 2, 'co' : 0.1, 'time': 300}>
                 >> 
         >
        ],    
    pos: 1
   >      
>>
```

6. Duplicated binding:

Suppose  `B_inWindow` is:

```
<<
    <l : {'sensor' : 1, 'co' : 0.4, 'time': 200}>,  -- b1
    <l : {'sensor' : 1, 'co' : 0.4, 'time': 200}>   -- b2
>>
```

With

```sql
WINDOWED () PARTITION AS p AT pos
```

The output binding collection is:

```
<<
   <
    l: {'sensor' : 1, 'co' : 0.4, 'time': 200}, -- can be either b1 or b2 
    p : [
         <l : {'sensor' : 1, 'co' : 0.4, 'time': 200}>,
         <l : {'sensor' : 1, 'co' : 0.4, 'time': 200}>
        ],
    pos: 0
   >,
   <
    l: {'sensor' : 1, 'co' : 0.4, 'time': 200},
    p : [
         <l : {'sensor' : 1, 'co' : 0.4, 'time': 200}>,
         <l : {'sensor' : 1, 'co' : 0.4, 'time': 200}>
        ],
    pos: 1
   >
>>
```

Remark: PartiQL does not specify the relative order of binding tuples with equal sort keys. However, the pos variable for each duplicated binding tuples has to be unique.

### Window function

SQL's window functions are similar to general functions in that it returns a value for every item in the input dataset. But a crucial difference is that a window function can potentially access different rows in the input data set.

PartiQL's windowed clause computed the partition each window function can access and carry the partition in the input binding tuples. Therefore, we can map the SQL's window functions to pure general functions.

For example: For each sensor, show the current co read and the previous co read:
In PartiQL:

```
SELECT VALUE
    {'sensor' : l.sensor,
     'current_read': l.co,
     'previous_read' : coll_lag(
                        SELECT VALUE p.l.co FROM p AT idx where idx = pos - offset,
                        NULL
                       )
    }
FROM logs as l
WINDOWED (
        PARTITION BY l.sensor
        ORDER BY l.time
    ) PARTITION AS p AT pos
```

where `coll_lag` is a core PartiQL window function corresponding to the SQL lag function.
Notice that `coll_lag` is a general function, if the first parameter is `NULL` or `MISSING` the `coll_lag` function will return the second parameter, otherwise it returns the first parameter as is.

Suppose logs is :

```
logs : <<
        {'sensor' : 1, 'co' : 0.4, 'time': 200},
        {'sensor' : 1, 'co' : 0.3, 'time': 100},
        {'sensor' : 2, 'co' : 0.1, 'time': 300}
       >>
```

The result of the above query is

```
<<
   {'sensor': 1, 'current_read': 0.4, 'previous_read': 0.3},
   {'sensor': 1, 'current_read': 0.3, 'previous_read': NULL},
   {'sensor': 2, 'current_read': 0.1, 'previous_read': NULL}
>>
```

### SQL Compatibility

SQL allows three ways to specify window:
1) inline window specification:

```sql
SELECT 
    rank() OVER (PARTITION BY sensor), -- inline window specification
    lag(co) OVER (PARTITION BY sensor ORDER BY time)
FROM logs as l
```

2) window clause

```sql
SELECT 
    rank() OVER w1,
    lag(co) OVER w2
FROM logs as l 
WINDOW w1 AS (PARTITION BY sensor) -- window clause
       w2 AS (PARTITION BY sensor ORDER BY time) -- window clause
```

3) reuse window definition

```sql
SELECT
    rank() OVER w1,
    lag(co) OVER w2
FROM logs as l
WINDOW w1 AS (PARTITION BY sensor)
       w2 AS (w1 ORDER BY time)
```

The above three queries are functionally the same. Notice that the two variants of SQL's WINDOW clause associate a window specification with a window alias. To reduce SQL's WINDOW clause to inline window specification is to replace the window alias with the specification associated with it.

Therefore, we demonstrate how to explain SQL's WINDOW function in PartiQL:
Suppose that a query:

1. Is a SELECT query
2. the SELECT clause contains one or more SQL window function (identified by OVER)

Then the query is rewritten using the following process:

* If the query contains variants of SQL's WINDOW clause, rewrite the query in form of inline window specification.
* Denote each window function as wf_i, its window specification ws_i
* Added a WINDOWED keyword(if not already exist)
* For each window function wf_i:
* Add the inline window specification ws_i
* Add PARTITION AS ws_i_partition AT ws_i_pos
* Rewrite `wf_i` into `wf'_i` where `wf'_i` is a PartiQL corresponding question.

The mapping relationship between `wf` (SQL’s window function) and `wf’` (PartiQL’s window function) is a little more complicated. For example, the mapping relationship for SQL’s `LAG` function is

An example mapping for LAG function is

```
 LAG(${expr}, ${offset}, ${default}) OVER (...) 
    -> coll_lag(SELECT VALUE {expr} 
                        FROM p AT index
                        WHERE index = pos - ${offset},
                   ${defualt})
```

For each SQL window function `wf`, an implementation MAY offer a corresponding core PartiQL window function `coll_f`.
The core PartiQL window function partiQL_f and the core PartiQL WINDOWED Clause, together explains the semantics of window function. Nevertheless, it is possible that an implementation offer only the SQL style window function without implementing the WINDOWED clause and PartiQL window functions.

Also notice that such rewriting creates a one-to-one mapping of window function and the window specification. For example:

```sql
SELECT 
    rank() OVER (PARTITION BY l.sensor ORDER BY l.time),
    lag(co) OVER (PARTITION BY l.sensor ORDER BY l.time)
FROM logs as l
```

will be rewritten to:

```sql
SELECT 
    coll_rank(ws_1_partition, ws_1_pos)
    coll_lag(ws_2_partition, ws_2_pos, l.co, 1, null)
FROM logs as l
WINDOWED
    (PARTITION BY l.sensor ORDER BY l.time) 
        PARTITION AS ws_1_partition AT ws_1_pos
    (PARTITION BY l.sensor ORDER BY l.time)
        PARTITION AS ws_2_partition AT ws_2_pos
```

# Drawbacks
[drawbacks]: #drawbacks

TBD

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives
The following approach has been considered:
```sql
WINDOWED (
PARTITION BY e1, ...., en
ORDER BY a1, ...., an
) AS w
```

Where for each `bi` in `B_inWindow`, the WINDOWED clause outputs a `b’` = `bi || w`, where `w` is a struct contains `p` and `pos`. The nested structure introduces additional complexity and is therefore discarded.


# Prior art
[prior-art]: #prior-art

SQL Spec: 2003 and later. 

# Unresolved questions
[unresolved-questions]: #unresolved-questions

TBD

# Future possibilities
[future-possibilities]: #future-possibilities

As a road map to be feature compatible with SQL’s window function, in the future we can:

1. Support other window function that operates on partition. i.e. rank
2. Extends our semantics and window operator for Frame clause. i.e., RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW.
3. Supports other window functions that operates on frame. i.e. first_value
4. Support usage of aggregation functions as window function. 

# Appendix
## POC on FRAME clause support
We can modify the windowed clause to
```sql
WINDOWED (
PARTITION BY ...
ORDER BY ...
RANGE | ROW ... // the SQL syntax here
) PARTITION AS p AT pos FRAME as f
```
Notice that for function that uses frame, for example first_value, the rewriting rule is in charge of “directing” the function to frame.
For example:
```
first_value(${expr}, ${offset}, ${default}) OVER (...)
-> coll_first_value(SELECT ${expr} FROM f)
```

## NULL TREATMENT Semantics
The semantics is to find the nth non-null value of x starting from current row if IGNORE NULLS is specified.
IGNORE NULLS [example](http://sqlfiddle.com/#!4/09d79/8) in Oracle

