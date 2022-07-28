# Summary on LET clause

* * *
<!-- TOC -->
* [Summary on LET clause](#summary-on-let-clause)
  * [Purpose](#purpose)
  * [Background](#background)
  * [Overview](#overview)
  * [Terminology](#terminology)
  * [Definition of `LET`](#definition-of-let)
    * [Use case of `LET`:](#use-case-of-let-)
    * [Proposed `LET` definition](#proposed-let-definition)
      * [Formal Definition - Option 1](#formal-definition---option-1)
      * [`FROM-LET` Example Based On the above Semantics](#from-let-example-based-on-the-above-semantics)
      * [Alternative options](#alternative-options)
        * [Definition - Option 2](#definition---option-2)
        * [Definition - Option 3](#definition---option-3)
    * [`LET` as an optional following clause to any other clause](#let-as-an-optional-following-clause-to-any-other-clause)
      * [`GROUP BY`/ `ORDER BY`](#group-by--order-by)
      * [`SELECT`](#select)
  * [Impact on extending the support of `LET`](#impact-on-extending-the-support-of-let)
    * [Case 1:  Supporting `LET` only after `FROM`](#case-1--supporting-let-only-after-from)
    * [Case 2: Extending supports to `GROUP BY`](#case-2--extending-supports-to-group-by)
    * [Case 3: Extending supports to all clauses excluding `SELECT`](#case-3--extending-supports-to-all-clauses-excluding-select)
    * [Case 4: Extending supports to all clauses including `SELECT`](#case-4--extending-supports-to-all-clauses-including-select)
  * [Interacting with `SELECT *`](#interacting-with-select-)
  * [Cross comparison](#cross-comparison)
  * [Appendix](#appendix)
<!-- TOC -->
* * *

## Purpose

Although we have implemented the `LET` clause in `Partiql-lang-kotlin`, we have not yet defined the semantics for `LET` in our spec. The purpose of this document is to keep notes on relative information we have on `LET` clause so far, and hopefully can lay some ground work for future RFC and spec work.
I used the word “proposed” for lack of a better alternative, please view it as content for discussion.
* * *

## Background

The original discussion around this issue was inspired by the GitHub issue [#653](https://github.com/partiql/partiql-lang-kotlin/issues/653) and [#549](https://github.com/partiql/partiql-lang-kotlin/issues/549), which can be summarized by the following table.

|Query	|CompilerPipeline Actual	|PlannerPipeline Actual	|
|---	|---	|---	|
|`SELECT f.* FROM << {'a':1} >> AS f LET 23 AS f`	|`<< { '_1' : 23 } >>`	|`<< { 'a' : 1 } >>`	|
|`SELECT * FROM << {'a':1} >> AS f LET 23 AS f`	|`<< { '_1' : 23 } >>`	|`<< { 'a' : 1 } >>`	|

To deicide the expected behavior, we need to figure out the definition for `LET` and `SELECT *`.
* * *

## Overview

In this doc, I first try to define the `LET` Statement based on the existing spec, then discussed two alternative approaches that may require small addition to the Spec. Next, I discussed the possibility of extending the support of  `LET` to other clauses and proposed how `SELECT *` could interact with variables defined by `LET`. Finally, I did a cross comparison on results of the above queries based on different design.
* * *

## Terminology

In this document, I tried to stay coherent with the terminology defined in our [spec](https://partiql.org/assets/PartiQL-Specification.pdf).

1. Binding tuple: `bi = ⟨ x1 : v1, ..... xn : vn⟩` where each `xi` is a **bind name** to the PartiQL value `vi`
2. The bag of binding tuples: `<<b1,......bk>>` denoted B<sup>in/out</sup><sub>Clause</sub>
3. `⍴0,⍴ ⊢ q -> v` denotes that the PartiQL query `q` evaluates to the value `v` when evaluated within the database environment `⍴0` and variable environment, i.e. when every variable of `q` is instantiated by its binding in `⍴` and each database name is instantiated to its value in `⍴0`.
4. Variables environment: denoted by `⍴` created by the defined query variables.

* * *

## Definition of `LET`

For now, consider the scope is limit to `FROM-LET`, i.e., `FROM` clause followed by `LET`.

### Use case of `LET`:

LET should be used as a way to reduce redundancy and complexity of the query. It is sometimes useful to store the result of a (sub) expression / (sub) query to use in the subsequent clause. `LET` should allow such “variables storage”, by creating new variable bindings based on the collection of bindings fed in through it as part of the `SELECT`-`FROM`-`WHERE`.

Example:

```SQL
SELECT * FROM s3object
WHERE (((((NOT (lower("payload") LIKE '%gurr%' ESCAPE '\'))
AND (NOT (lower("payload") LIKE '%[stream_shadow=true]%' ESCAPE '\'))
AND (((NOT (lower("payload") LIKE '%gurr%' ESCAPE '\'))
AND lower("_sourcecategory") = 'stream' AND lower("payload") LIKE '%handling%' ESCAPE '\'
AND lower("payload") LIKE '%failure%' ESCAPE '\' AND lower("payload") LIKE '%error%' ESCAPE '\'
......
```

With let, we can rewrite the query to something similar to:

```SQL
SELECT *
FROM s3object AS s3
LET LOWER("payload") AS x
WHERE NOT x LIKE ...
```

* * *

### Proposed `LET` definition

Informally:

`LET` should be operating a one-on-one mapping on the input bindings. i.e., for each **binding tuple** `bi` in B<sup>out</sup>, `LET` should extends `bi` with a new binding.

Proposed EBNF Grammar:
```EBNF
<let clause> ::= <expr query> AS <identifier> [, <expr query> AS <identifier>] ...
```

Syntax:
`LET e1 as x1, ... , en as xn`
* * *

#### Formal Definition - Option 1

Note that in this option, I tried to obey the current spec as much as I can without introducing additional operator/definition.

1. **Support for a single `LET source`.**

Syntax: `LET e as x`, where `e` is an **expression** and `x` is an **element variable**.

The `LET` clause inputs a bag of binding tuples B<sup>in</sup> and output a bag of binding tuples B<sup>out</sup>. For each input binding tuple `b` in B<sup>in</sup><sub>Let</sub>, the LET clause outputs `b || ⟨ x : v ⟩`, where `⍴0,⍴||b ⊢ e -> v`
* * *

2. **Support for multiple LET source.**

Syntax: `LET e1 as x1, ... , en as xn`, where `e` is an **expression** and each `xi` is an **element variable**.

The expression `ej` may use variables defined by the `ei`. for `1 ≤ i ≤ j ≤ n`.

i.e., `LET 1 as a, a + 1 as B` should be legal.

We can define this behavior inductively:

Consider `LET e1 as x1, ......., ei as xi (1)`, where `i < n`.

Assume `(1)` outputs **a bag of binding tuples** `Bi`, then `LET e1, ...., ei+1` outputs a **binding tuple** `bi || ⟨xi+1 : vi+1⟩`, for each `bi` in `Bi` ,   `⍴0,⍴||bi ⊢ ei+1 -> vi+1`

We can generalize the first two rules as the following pseudo-code:

```
Prev <- Bin_Let
Cur <- <<>>  // an empty bag 
for each binding tuple b in Prev:
    let_binding = ⟨⟩
    for each e in LET SOURCE:
        let_binding = let_binding || ⟨r_current:e_current⟩
    add b || let_binding to Cur
    Prev = Current
    Current = <<>>
return Prev
```

* * *

#### `FROM-LET` Example Based On the above Semantics

Consider that we are in a database called `mydb`, such database(schema) have two tables, `customers` and `orders`

```
customers: <<
    {'id': 5, 'name': 'Joe'}, 
    {'id': 7, 'name': 'Mary'}
>>
```

```
orders: <<
    {'custId' : 7, 'productId' : 101},
    {'custId' : 7, 'productId' : 523}
>>
```

The following query:

```SQL
FROM mydb.customers as c LET mydb.orders as o
```

Should output:

```
Bout_From = BinLet =
          << 
            ⟨c: {'id' : 5, 'name' : 'Joe'} ⟩
            ⟨c: {'id' : 7, 'name' : 'Mary'} ⟩
          >>
Bout_Let = << 
            ⟨c: {'id' : 5, 'name' : 'Joe'}, 
             o: <<
                   {'custId' : 7, 'productId' : 101},
                   {'custId' : 7, 'productId' : 523}
                >> 
             ⟩
            ⟨c: {'id' : 7, 'name' : 'Mary'},
             o: <<
                   {'custId' : 7, 'productId' : 101},
                   {'custId' : 7, 'productId' : 523}
                >>
             ⟩
          >>
```

* * *

#### Alternative options

The two options below re-use the work above with small changes in the pseudocode. For readability, I will name them option 2, and option 3 respectively.

Since we do not explicitly disallow duplicated **bind names** in the **binding tuples**, i.e., `⟨ x: ..., x: ...⟩` is so far allowed, at least in our implementation ( `GROUP BY ... as x GROUP as x`). Assuming this is indeed the behavior we desired, then:

We could define that an operation on **binding tuple**, say `+`,
Let a =  `⟨ a1: b1, ... , ai: bi⟩, b = ⟨ x1: y1, ... , xk, yk⟩`
Then a+b = `⟨ a1: b1, ... , ai: bi, x1: y1, ... , xk, yk⟩`
Notice that `aj` can be the same as `xk`

##### Definition - Option 2

This definition slightly modifies option 1 in that it uses the newly defined operator `+` to add **bind names** without modifying the existing **bind names** in case of shadowing.

```
Prev <- Bin_Let
Cur <- <<>>  // an empty bag
for each binding tuple b in Prev:
    for each e in LET SOURCE:
        let_binding = let_binding + ⟨r_current:e_current⟩
    add (b + let_binding) to Cur
    Prev = Current
    Current = <<>>
return Prev
```

Notice that

``` SQL
LET 1     AS x 
    2     AS y 
    x + y AS x
```

will result in:

``` 
x = 1
y = 2 
x = 3 
```

##### Definition - Option 3

This definition uses the concatenation operator `||` when dealing with `LET` source, and uses the newly defined `+` operator to add **bind names** to the current **variable environment**.

```
Prev <- Bin_Let
Cur <- <<>>  // an empty bag
for each binding tuple b in Prev:
    let_binding = ⟨⟩
    for each e in LET SOURCE:
        let_binding = let_binding || ⟨r_current:e_current⟩
    add (b + let_binding) to Cur
    Prev = Current
    Current = <<>>
return Prev
```

This way:  Notice that

```SQL
LET 1     AS x 
    2     AS y 
    x + y AS x
```

will remain as:

```
y = 2 
x = 3 
```
Let us use a brief example to demonstrate the difference between those three options. 
Consider the query:

```FROM << { 'a' : 1} >> as x LET 2 as x, x + 1 as x ```

```
Bout_let = << x : 3 >> // option 1
Bout_let = << x: {'a' : 1}, x: 2, x: 3>> // option 2
Bout_let = << x: {'a' : 1}, x: 3>> // option 3
```


The implication of each option will be further demonstrated in the next section. 
Also a complete comparison of the expected behavior of the queries listed in the Background section
based on different option can be found in [Cross Comparison section](#cross-comparison). 
To reduce the length, we will mostly compare Option 1 with Option 3.
* * *

### `LET` as an optional following clause to any other clause

With the proposed `LET` definition, let us explore the possibility of extends the support for `LET` to other clauses.

```
FROM ...
LET ...
WHERE ...
LET ...
GROUP BY ...
LET ...
HAVING ...
LET ...
ORDER BY ...
LET ...
OFFSET ...
LET ...
LIMIT ...
LET ...
SELECT ...
LET ...
```

Consider the following to example:

``` SQL
FROM a LET b AS x, c AS y WHERE x > 1
```

``` SQL
FROM a LET b AS x WHERE x > 1 LET c AS y
```

`TODO:` expand the output of both examples to be more explicit about the function of `WHERE` predicate in the output.

Now notice **the bag of binding tuples** produced by the two queries are exact the same. One could argue that a query optimizer should be able to decide where the `LET` should be executed.

Informally, for any `LET` after `WHERE`, it is always functionally equivalent to push variable declaration up to `LET` after `FROM`.
For any `LET` after `HAVING`, it is always functionally equivalent to push variable declaration up to `LET` after `GROUP BY`.
For any `LET` after `LIMIT` or `ORDER BY`, it is always functionally equivalent to push variable declaration up to `LET` after `ORDER BY`.


With this in mind, the question we should focus on first is, should we support `LET` after `GROUP BY`, `LET` after `ORDER BY` and `LET` after `SELECT`.
* * *

#### `GROUP BY`/ `ORDER BY`

Notice that since the proposed `LET` semantics follows the pattern of “**input bag of binding tuples, evaluate constituent expressions, output bag of binding tuples**”, it should have no problem following `GROUP BY` and `ORDER BY`.
For example:

``` SQL
From log AS l GROUP BY l.sensor AS sensor GROUP AS g LET 1 AS x
```

Suppose that:

```
Bout_GROUPBY = Bin_Let 
         = **<<**
            ⟨sensor:1, 
             g: << ⟨l:{’sensor’:1, ’co’:0.4}⟩,⟨l:{’sensor’:1, ’co’:0.2}>>
            ⟩,
            ⟨sensor:2, 
             g:<< ⟨l:{’sensor’:2, ’co’:0.3}⟩ >>**
            ⟩ 
           >> 
**
```

Then:

```
Bout_Let = <<
            ⟨sensor:1, 
             g: << ⟨l:{’sensor’:1, ’co’:0.4}⟩,⟨l:{’sensor’:1, ’co’:0.2}>>,
             x:1
            ⟩,
            ⟨sensor:2, 
             g:<< ⟨l:{’sensor’:2, ’co’:0.3}⟩ >>,
             x:1
            ⟩ 
           >> 
```

`TODO`: expand the application of `LET` in `ORDER BY`. 

* * *

#### `SELECT`

Consider `SELECT ... LET ...`, should the `LET` be executed after the SELECT?
If so, there is no clause can reference the variable defined in the `LET` clause that follows `SELECT` in a single SFW query.

Perhaps a more practical  interpretation is that `LET` following `SELECT` will be the first executed clause. That could potentially be useful as it empowers the user to do single execution (executed once per query).  However, it breaks the previous defined logical execution order of the query. i.e.

```
FROM ...                          LET
LET ...                           FROM   
...                               LET
...                               ...
```

Single execution is out of scope for this document, but if we were to implement this feature, using another keyword at the top of query makes more sense to me, i.e.,

```
KEYWORD_FOR_SINGLE_EXECUTION ....
SELECT .....
```

* * *

## Impact on extending the support of `LET`

Here I find that different interpretation of `LET` may lead to different expectation to shadowing behavior.
Here I use the defined three options to demonstrate different shadowing behaviors under each scenario.

### Case 1:  Supporting `LET` only after `FROM`

Problem: LET as a clause or LET as a keyword?
In this scenario, `LET` should be functionally the same as `CROSS JOIN` against a singleton.
Treating `LET` differently here offers ground for different shadowing behavior.

1. `LET` as a keyword in `FROM` clause:

* Since we do not allow shadowing in the `FROM` clause(not explicitly stated in the spec, but no one really allows it), shadowing should be prohibited in this scenario.
* `LET` could be defined similar to `JOIN` in regard to our grammar.
* If we defined it that way, we could not extends the support to other clauses.

   Example: 
   ```sql
   FROM << { 'a' : 1} >> as x LET 2 as x
   ```
   In this scenario, we should consider the above as `FROM` clause. An analogy(similar, but not exact equivalent) will be:
   ```sql
   FROM << { 'a' : 1} >> as x CROSS JOIN 2 as x
   ```
   Since we do not allow shadowing in `FROM` clause, the above will throw an error. 

2. `LET` as a clause:

* The advantage to do so is that we can extend the support to different clause in the future.
* Under option 1, we allow shadowing by saying that variables defined in `LET` will overwrite the variables defined in the `FROM`

```
FROM << { 'a' : 1} >> as x LET 2 as x
Bout_Let = << ⟨x : 2⟩ >>
```

* Under option 3, `LET` will just add an additional binding with the same name.

```
Bout_Let = << ⟨x : {'a':1}, x: 2⟩ >>
```

* Excluding `SELECT`, I could not find a case where the above definition can cause problem, the issue with `SELECT` will be further demonstrated below.

In my opinion, if we decide that we shall not support `LET` in other clauses in the future, then `LET` should be considered as a keyword.
* * *

### Case 2: Extending supports to `GROUP BY`

Consider the following query:

```
FROM log AS l LET 1 AS x GROUP BY l.sensor AS sensor GROUP AS g LET 2 AS y
```

```
Bout_Let = <<
            ⟨sensor:1, 
             g: << ⟨l:{’sensor’:1, ’co’:0.4, ’x’:1}⟩,
                   ⟨l:{’sensor’:1, ’co’:0.2, ’x’:1}    <-- First LET
                >>,
             y:2 <--- Second LET
            ⟩,
            ⟨sensor:2, 
             g:<< ⟨l:{’sensor’:2, ’co’:0.3, ’x’:1}⟩ >>,
             y:2
            ⟩ 
           >> 
```

Question 1: Different lexical Scope

* Notice that the two variables (`x`,`y`) defined respectively by the two `LET` statements have different lexical scope, which could causing confusion to the user.
* I personally don’t find this to be a huge problem, it could be just me overthinking, but this problem may be solved by having a different keyword, i.e. `LETTING` to follow the `GROUP BY` clause.
* If we only allow “LET”, after `FROM` and `GROUP BY`, it might be worth to consider using different keywords.
* Again may be worth to say that “LET”/“LETTING” after `GROUP BY` is a keyword instead of a clause in this case.


Question 2: Shadowing:
Without the complication of `LET`, if we have `GROUP BY ... AS x GROUP AS x`, which we currently permit, then:

```
Bout_GroupBy= <<
            ⟨x: ..., <----- GROUP BY
             x: ..., <----- GROUP; In case of SELECT, this will be outputed.
            ⟩,
            ......
            >>
```

For reasons that I don’t fully understand, `SELECT x FROM GROUP BY ... AS x GROUP AS x` will output the second `x`.

Going with option 1:
If we add `LET ... AS x`, which binding should the `x` defined in `LET` replaced?

Going with option 3:
Option 3 solve this problem by allowing x to be added into the **binding tuples** without modifying the existing content of two **bind name** `x`.
Then how should we define `SELECT` behavior? i.e. what should `SELECT x` outputs?
* * *

### Case 3: Extending supports to all clauses excluding `SELECT`

I argue this is more for user convenience. If we decide that we potentially can go down this route, then `LET` should be considered as a clause, not a keyword.
* * *

### Case 4: Extending supports to all clauses including `SELECT`

As stated before, defining what `LET` after `SELECT` means is tricky. I purpose that we not implement this for now.
That said, if we shall revisit this decision, it should be relatively easy to plug the feature in.
* * *

## Interacting with `SELECT *`

The practical question here is, should SELECT * outputs variable produced by LET.

The following forms the basis of the analysis:

>According to the spec section 3.3:
the `SELECT` clause is responsible for converting from binding tuples to collections of arbitrary PartiQL elements. The `SELECT` **** inputs a bag (or array, if `ORDER BY` is present) of binding tuples, and outputs the SFW query’s result, which is a bag (resp. array) with exactly one element for each input binding tuple. I


Unfortunately, the Spec was not too particular about unqualified asterisk. The gap should be filled with a mechanism to convert unqualified asterisk to qualified asterisk.   
I presume the following happens:

```
for each bind_name n in ⍴:
    add n.*
```

i.e. Assume the bond name in `⍴`  is `(n1, ... , nk)`,  then

```
SELECT * FROM ....
```

is equivalent to

```
SELECT n1.*, ... , nk.* FROM ....
```

It seems like `SELECT *` should output the variable defined by `LET`.

That said, if we decide otherwise, the implementation should not be a problem.
* * *

## Cross comparison

|Query	|
|---	|
|`SELECT * FROM << {'a':1} >> AS f LET 23 AS g`	|
|`SELECT * FROM << {'a':1} >> AS f LET 23 AS f`	|
|`SELECT f.* FROM << {'a':1} >> AS f LET 23 AS g`	|
|`SELECT f.* FROM << {'a':1} >> AS f LET 23 AS f`	|

For query 1:

```
If SELECT * outputs variables defined by LET:
    Both option 1 and option 3 shall output: 
        << {'a':1,'_1': 23} >>
else:
    Both option 1 and option 3 shall output: 
        << {'a':1} >>
```

For query 2:

```
If SELECT * outputs variables defined by LET:
    Option 1 shall output: 
        << {'_1': 23} >>
    Option 3 shall output: 
        << {'a':1,'_1': 23} >>
else:
    option 1 shall output: 
        <<>>
    option 3 shall output:
        << {'a':1} >>
```

For query 3:

```
If SELECT * outputs variables defined by LET:
    Both option 1 and option 3 shall output: 
        << {'a':1} >>
else:
    Both option 1 and option 3 shall output: 
        << {'a':1} >>
```

For query 4:

```
If SELECT * outputs variables defined by LET:
    Option 1 shall output: 
        << {'_1': 23} >>
    Output of Option 3 remains a little vague: 
        << {'_1': 23} >> or << {'a': 23} >> <--- undeterministic or deterministic
        or
        << {'a':1,'_1': 23} >> 
else:    
    option 1 shall output: 
        <<>>
    option 3 shall output:
        << {'a':1} >>
```

In my opinion, `SELECT *` should output the variables defined by `LET`, and choice between option 1 and option 3 depends on whether we formally allows duplicated name in binding tuples.
* * *

## Appendix

1. PartiQL spec: https://partiql.org/assets/PartiQL-Specification.pdf
2. Github RFC on LET: https://github.com/partiql/partiql-spec/issues/15
3. Paraphrasing Almann’s analogy to have a initial overview of `LET` (FROM-LET):
    1. `LET` is similar to `CROSS JOIN` with one critical difference, `LET` will never increase the cardinality of the result, whereas `CROSS JOIN` may increase the cardinality.
    2. `LET` should be treated exactly as `CROSS JOIN` against a singleton. i.e `(a1, b1, c1)x(a2, b2, c2) vs. (a1, b1, c1)x((a2, b2, c2))`
    3. `LET` is a one-to-one mapping,  it takes the environment as an input and amends that environment, whereas `CROSS JOIN` is a flatten map, it takes the environment as an input, amends that environment, and flattens the result.
4. SQL++ paper: https://dbucsd.github.io/paperpdfs/2014_1.pdf
5. Couchbase LET: https://docs.couchbase.com/server/current/n1ql/n1ql-language-reference/let.html
6. Oracle LET: https://docs.oracle.com/cd/E93962_01/bigData.Doc/eql_onPrem/toc.htm#LET%20clause


