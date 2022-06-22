- Start Date: 2022-06-21
- PartiQL Issue [#524](https://github.com/partiql/partiql-lang-kotlin/issues/524)
- RFC PR [#13](https://github.com/partiql/partiql-docs/pull/13)

# Summary
[summary]: #summary

The purpose of this RFC is to establish the rules of coercion for the PartiQL IN Predicate. This document proposes that
the PartiQL Specification be updated to reflect the following rules:
- When the Right-Hand Side (RHS) of an IN Predicate is a Select-From-Where (SFW) sub-query with a projection list of
size *n* where *n* > 1, the results (tuples) of the SFW sub-query are coerced to a sorted list of the tuple's values.
- When the RHS of an IN Predicate is an SFW sub-query with a projection list of size *n* where *n* = 1, the results
  (tuples) of the SFW sub-query are coerced to the singlular value that each respective tuple holds.
- When the RHS of an IN Predicate is an SFW sub-query with a projection of `*`, the resulting tuples will be coerced
  contingent on the presence of schema.
- When the RHS of an IN Predicate is not an SFW sub-query, the RHS will not experience any coercion.

With these four rules of coercion, PartiQL is able to provide an experience similar to that of other SQL DBMS's.

# Motivation
[motivation]: #motivation

This document proposes this change to comply with the requirements of the SQL-92 Specification and to provide customers
with an experience similar to that of other SQL DBMS's. According to the
[PartiQL Open-Source Charter](https://partiql.org/charter.html), "existing SQL queries will continue to work in SQL
query processors extended to provide PartiQL".

The initial motivation came from [GitHub Issue #524](https://github.com/partiql/partiql-lang-kotlin/issues/524), which
outlined the current difference in results between PartiQL and PostgreSQL. Please reference the issue to see an example
of the inconsistency between PartiQL and PostgreSQL.

# Guide-Level Explanation
[guide-level-explanation]: #guide-level-explanation

### General Explanation

The IN Predicate is a PartiQL predicate that acts as a function that determines membership within sequences. Before
giving examples of how this works within PartiQL, users of PartiQL can view the IN Predicate to act similarly to the
Kotlin pseudocode below:
```kotlin
// Code Snippet #1
fun inPredicate(lhs: T, rhs: Sequence<T>): Boolean = rhs.contains(lhs)
```

#### Single Projection Item

Let us consider the simplest case, where we are comparing using a single value. Consider the following example:

```sql
-- Code Snippet #2
SELECT id, full_name, ssn FROM users WHERE id IN (SELECT id FROM blocked_users);
```

The above valid SQL/PartiQL query, in simple terms, shows the `id`, `full_name`, and `ssn` of all users in the `users` table that are
also found in the `blocked_users` table. The example SFW sub-query, in a typical SQL engine, results in a single-column table
with all IDs present in `blocked_users`. Therefore, the IN Predicate, in this scenario, will determine whether the
scalar `id` is a member of the resulting column (sequence of scalars).

PartiQL, on the other hand, does not represent the result of the above SFW sub-query as a single-column table. It
actually represents the column by returning a sequence (bag or list) of tuples (structs). This struct
holds the resulting scalar `id` by placing it in a key-value pair. In order to make the above query work in PartiQL,
PartiQL shall coerce each tuple into the value of the singular key-value pair that it holds. Therefore, in this case, the SFW
sub-query shall return a sequence (bag or list) of whatever `id` represents (possibly an `UNSIGNED INT`).

Therefore, in the above example, PartiQL will coerce the RHS sub-query's results from
```partiql
<< { 'id': 0 }, { 'id': 1 }, { 'id': 2 } >>
```

to this:

```partiql
<< 0, 1, 2 >>
```

#### Multiple Projection Items

To expand on this, PartiQL shall also allow multiple projection items within the RHS sub-query. Take the following valid
SQL/PartiQL query that gathers the product information (`id`, `pname`, `country`) from `grocery_store_0` of products that
are also sold in some other grocery store (`grocery_store_1`):

```sql
-- Code Snippet #3
SELECT id, pname, country FROM grocery_store_0 WHERE (pname, country) IN (SELECT product_name, origin_country FROM grocery_store_1)
```

To relate the example above to the Kotlin example (Code Snippet #1), the LHS is a list containing the values of `pname`
and `country`, and the RHS is a sequence (bag or list) containing 0 or more lists of size 2. Typical SQL engines, in response
to the SFW sub-query on `grocery_store_1`, would return a table of 2 columns and determine whether the 2-column tuple
on the LHS is a member of the RHS table.

PartiQL, as we know, represents the results of an SFW sub-query as a sequence (bag/list) of structs containing key-value
pairs. Therefore, in order to accommodate the above SQL query, PartiQL shall coerce the two, explicitly specified, 
ordered projection items into a list. Therefore, the above RHS SFW sub-query would be coerced into a sequence of lists
of size 2.

Therefore, in the above example, PartiQL will coerce the RHS sub-query's results from

```partiql
<<
  { 'product_name': 'apple', 'origin_country': 'US' },
  { 'product_name': 'avocado', 'origin_country': 'MX' }
>>
```

```partiql
<<
  ['apple', 'US'],
  ['avocado', 'MX']
>>
```

#### Unspecified Number of Projection Items

This document has discussed how PartiQL represents the results of SFW sub-queries due to its
schema-less nature. An implication of this representation is the use of structs, which are defined to be an unordered
collection of key-value pairs. As such, the order of the contained elements cannot be guaranteed. With this in mind, 
in the absence of schema, PartiQL shall *not* perform coercion on any SFW sub-query on the RHS of an IN Predicate that projects
all columns using the `*` symbol.

However, in the presence of schema, the functionality will act identically to the above scenarios. See the following example:

```sql
-- Code Snippet #4
SELECT id FROM ids WHERE id IN (SELECT * FROM users);
```

In the presence of schema, PartiQL is capable of determining the number of ordered columns
in the `users` table. If the table has a single column, this query is valid -- otherwise, it is not. However, without a
defined schema, PartiQL has no way of representing order within the columns of a data source, and, therefore, it cannot support
coercion of the RHS SFW sub-query. Therefore, the above query (Code Snippet #4), would evaluate as-is -- the possibly
scalar `id` would be compared against the sequence of structs.

#### Literal Sequences

Another important scenario of the IN Predicate is the case where a literal sequence is on the RHS of an IN Predicate. In
this scenario, PartiQL shall *not* perform coersion on the RHS, regardless of its data type. See the following simple, 
yet valid, example:

```sql
-- Code Snippet #5
SELECT disease_count FROM cdc_stats_2022 WHERE 1 IN (1, 2, 3);
```

The above valid SQL query is a simple example of evaluating the LHS and RHS as-is. For a more complex PartiQL query, see
below:

```partiql
-- Code Snippet #6
SELECT disease_count FROM cdc_stats_2022 WHERE disease_id IN [ { 'disease_id': 45 } ];
```

Code Snippet #6 gives an example where the RHS is a literal sequence containing a single struct. According to this
proposal, PartiQL will *not* perform coercion on the RHS of the IN Predicate. The above IN Predicate would **always**
return `false` as the disease_id (presumably an UNSIGNED INT) would be compared against a sequence of structs (not a sequence
of UNSIGNED INTs).

#### Current Specification vs Proposal

Current users of PartiQL will *only* gain new semantics to further allow SQL queries to work without modification using
PartiQL. In the past, PartiQL users were required to use the **VALUE** clause to get SQL-like functionality. Taking our
first example (Code Snippet #2), it needed to be written as:

```partiql
-- Code Snippet #2 (Copied for Reference)
SELECT id, full_name, ssn FROM users WHERE id IN (SELECT id FROM blocked_users);

-- Code Snippet #7 (Modified from Code Snippet #2)
SELECT id, full_name, ssn FROM users WHERE id IN (SELECT VALUE id FROM blocked_users);
-- OR
SELECT id, full_name, ssn FROM users WHERE { 'id': id } IN (SELECT id FROM blocked_users);
```

With multiple projection items, Code Snippet #3 needed to be rewritten as:
```partiql
-- Code Snippet #3 (Copied for Reference)
SELECT id, pname, country FROM grocery_store_0 WHERE (pname, country) IN (SELECT product_name, origin_country FROM grocery_store_1)

-- Code Snippet #8 (Modified from Code Snippet #3)
SELECT id, pname, country FROM grocery_store_0 WHERE (pname, country) IN (SELECT VALUE [product_name, origin_country] FROM grocery_store_1)
-- OR
SELECT id, pname, country FROM grocery_store_0 WHERE { 'product_name': pname, 'origin_country': country } IN (SELECT product_name, origin_country FROM grocery_store_1)
```

# Reference-Level Explanation
[reference-level-explanation]: #reference-level-explanation

#### Single Projection Item

In order to make this change in the current implementation, the implementation needs to coerce the SFW sub-query to
return a sequence of values. The implementation can do this by rewriting Code Snippet #2 into the following:

```partiql
-- Code Snippet #2 (Copied for Reference)
SELECT id, full_name, ssn FROM users WHERE id IN (SELECT id FROM blocked_users);

-- Code Snippet #9 (Modified from Code Snippet #2)
SELECT id, full_name, ssn FROM users WHERE id IN (SELECT VALUE id FROM blocked_users);
```

#### Multiple Projection Items

In order to support an RHS SFW sub-query, the implementation would coerce Code Snippet #3 by wrapping the SFW sub-query's
projection items into a list of values. See below:

```partiql
-- Code Snippet #3 (Copied for Reference)
SELECT id, pname, country FROM grocery_store_0 WHERE (pname, country) IN (SELECT product_name, origin_country FROM grocery_store_1)

-- Code Snippet #10 (Modified from Code Snippet #3)
SELECT id, pname, country FROM grocery_store_0 WHERE (pname, country) IN (SELECT VALUE [product_name, origin_country] FROM grocery_store_1)
```

#### Unspecified Number of Projection Items

In the absence of schema, the query processor will not perform any re-writing of the query.

In the presence of schema, the query processor will re-write the sub-query based on the number of columns in the schema.
If the schema contains a single column, please refer to the section on [Single Projection Items](#reference-level-explanation).
If the schema contains more than a single column, please refer to the section on [Multiple Projection Items](#reference-level-explanation)

#### Literal Sequences

As coercion will not take place for literal sequences, this document does not propose any modification to the current
implementation for these two cases.

# Drawbacks
[drawbacks]: #drawbacks

With these rules of coercion, there is 1 specified drawback:
1. Perceived inconsistency

#### Perceived Inconsistency

Developers have largely been taught to incrementally test their code as they build large systems. With the concept of
incremental testing, developers might find themselves questioning why PartiQL acts in certain ways (if they have not
fully read and understood the specification). This document will present the following scenario:

> A developer wants to write a query with an IN Predicate where the RHS is an SFW sub-query. The developer might first 
> write the SFW sub-query and run it. Then, since PartiQL supports bags as first-class citizens, the developer can copy and paste 
> the SFW sub-query's results into the RHS of the query's IN Predicate. Then, they might shape their query around this 
> literal (which does not perform coercion). Once their query works, they could replace the literal with their original
> SFW sub-query -- but, then, as coercion takes place, their query would not return the same results.

# Rationale and Alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

The proposed design is the best option to maintain conformance with SQL. While it is limited by how PartiQL
represents schema (through unordered structs), it allows additional semantics that are currently present in other SQL
engines and the SQL-92 Specification.

Other designs considered included:
- Always coercing the RHS SFW sub-query elements into the list of values it holds at evaluation time. This, however,
wouldn't work as the underlying struct (prior to coercion) cannot guarantee ordering.
- For the scenario of a single projection item, coercing both the LHS and RHS of the IN Predicate to be a single-element
list to make the implementation slightly easier, although this would add additional overhead during evaluation.
- Deviating from SQL conformance and asking users to use the VALUE clause as appropriate. This would remove the drawback
  of perceived inconsistency, but it would break PartiQL's tenet of SQL backwards compatibility.

The impact of not implementing this proposal is that SQL users transitioning to PartiQL will not be able to run their SQL queries
on PartiQL without needing to modify their existing queries. PartiQL would not be abiding by its tenet of SQL conformance.

# Prior Art
[prior-art]: #prior-art

This document will examine the current PartiQL Specification and the SQL-92 Specification.

#### PartiQL Specification

While the PartiQL Specification does not specify rules of coercion for the elements within the result of an SFW sub-query,
this document will reference *somewhat* relevant sections of the PartiQL Specification for the reader's knowledge.

Section 9.1 of the PartiQL Specification gives insight into, specifically, how an SFW sub-query undergoes coercion into a scalar:

> A SFW subquery coerces into a scalar ... if it is an SFW subquery expression that (a) is not the collection expression
> of a *FROM* clause item and (b) is not the rhs of an *IN*. (If it is the rhs of an *IN* then it should not be coerced;
> see note on semantics of *IN*, Section *??*).

> Or it may be a subexpression of the *WHERE* clause expression, as long as it is not the rhs of an *IN*. In any of
> these cases the result of the subquery s is cast into a scalar. Technically, the subquery s (which uses *SELECT*) is
> rewritten into an equivalent subquery s′ that utilizes *SELECT VALUE*, by following the steps of Section 6.3. Then the
> result of s′ is cast into a scalar by applying the function *COLL_TO_SCALAR(*s′ *)*.

Section 9.2 provides more information on coercion into arrays:

> An *SELECT* SFW subquery coerces into an array when it is the rhs (respectively, lhs) of a comparison operator
> whose other argument is an array.

The PartiQL Specification also sheds some light on the role of schema in type checking (Section 4.1.1):
> In the presence of schema, PartiQL may return a compile-time error when the query processor can prove that the path
> expression is guaranteed to always produce MISSING. The extent of error detection is implementation-specific.

The above excerpt provides insight into how implementations of PartiQL may accomplish appropriate coercion of the RHS SFW
sub-query when the projection includes the `*` symbol.

#### SQL-92 Specification

Now, as we have shown that the PartiQL Specification does not explicitly provide direction on how to coerce elements of
the result of a sub-query on the RHS of an IN operation, it is important to further investigate any constraints provide
by the SQL-92 Specification — as it serves as a guide for PartiQL.

Likely most important from the SQL-92 Specification is contained within Section 17.6. See the following excerpt:

> Q) In an \<in predicate\> that specifies a \<table subquery\>, the data types of \<value expression\>s E1 in the \<row value
> constructor\> are assumed to be the same as the data types of the respective columns of the \<table subquery\>.  
> ...  
> R) In an \<in predicate> that specifies an \<in value list\>, if the \<row value constructor\> is not E1, then let D be its
> data type. Otherwise, let D be the data type of the first \<value specification\> of the \<in value list\>. The data type
> of any E1 in the \<in predicate\> is assumed to be D.

The definition of an `<in predicate>` can be gathered from Section 8.4 of the SQL-92 Specification:

> **Format**  
> \<in predicate> ::=  
> &nbsp; &nbsp; \<row value constructor>  
> &nbsp; &nbsp; &nbsp; &nbsp;\[ NOT ] IN \<in predicate value>  
> \<in predicate value> ::=  
> &nbsp; &nbsp; \<table subquery>  
> &nbsp; &nbsp; | \<left paren> \<in value list> \<right paren>  
> \<in value list> ::=  
> &nbsp; &nbsp; \<value expression> { \<comma> \<value expression> }...
> 
> **Leveling Rules**
> 1) The following restrictions apply for Intermediate SQL:  
> &nbsp; &nbsp; a) Conforming Intermediate SQL language shall not contain a \<value expression> in an \<in value list> that is not a <value specification>.
> 2) The following restrictions apply for Entry SQL in addition to any Intermediate SQL restrictions.  
> &nbsp; &nbsp; None.

With the above excerpts in mind, this document asserts that the current implementation of PartiQL does *not* abide by
the SQL-92 Specification. Examining Sentence R above, it becomes apparent that the data types of the "\<row value constructor>"
(AKA, the LHS) are *not* currently assumed to be the same as the data types of "the \<in value list>" (RHS).

This document makes this assertion based on how PartiQL wraps the RHS elements in a struct -- therefore performing a
comparison with the LHS values and the sequence's structs on the RHS. By opting to avoid coercion on literals and to accept
coercion with sub-queries, PartiQL is maintaining SQL-92 conformance.

# Unresolved Questions
[unresolved-questions]: #unresolved-questions

Please reference the [PR's correspondence](https://github.com/partiql/partiql-docs/pull/13). 

# Future Possibilities
[future-possibilities]: #future-possibilities

This document does not put forward any future possibilities.