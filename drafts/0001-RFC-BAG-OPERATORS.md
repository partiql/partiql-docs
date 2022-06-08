- Feature Name: Bag Operators: UNION, INTERSECT, EXCEPT
- Start Date: 2022-06-07

# Summary
[summary]: #summary

This RFC defines the PartiQL specification for the operators: UNION, INTERSECT, and EXCEPT.

# Motivation
[motivation]: #motivation

The operators UNION, INTERSECT, and EXCEPT have been defined since SQL-92 specifications, yet these are missing from the PartiQL spec.
In SQL, these operators are the typical bag (set if DISTINCT) union, intersect, and difference respectively for *union-compatible* relations. Unlike SQL, PartiQL is schemaless and therefore these set operators are valid for arbitrary expressions — including a SFW block.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Each bag operator has the form *q S q'* where *q* and *q'* are of type *<query>* and *S* is the operator. Additionally, the operator may be suffixed with *ALL* which indicates the output may have duplicate elements. In its absense, duplicates are eliminated from the final result. Arguments of the operators are coerrced into bags with the following behavior.  
- *NULL* and *MISSING* become the empty bag *<< >>*
- A scalar value *v* becomes the singleton bag *<< v >>*
- An array is coerced to a bag by discarding ordering

The behavior of bag operators is specified by permissive or conventional type checking mode. The typing mode determines if heterogenous arguments are allowed or not allowed. Conventional typing mode will produce an error if a bag operator is used on heterogenous types. All six operators are defined in Figure 1.

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

INTERSECT(A, B) = DISTINCT(INTERSECT(A, B))

EXCEPT(A, B) = DISTINCT(EXCEPT(A, B))
```

The following examples show how these operators can be used to construct new heterogenous bags. All examples will use permissive typing mode as conventional typing mode is equivalent to the SQL bag operators.

**TODO EXAMPLES**

# Drawbacks
[drawbacks]: #drawbacks

We do not recognized drawbacks to implementing these features as these are clearly defined in the specification yet missing from the implementation.

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

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- Do we want other configuration to allow for singleton bag of *<< NULL >>* and *<< MISSING >>* rather than the empty bag?
- Should we allow for combining heterogenous bags in conventional mode? How would we type check all arguments of the bag?
- Is the definition for `INTERSECT_ALL` to clunky? Is it correct and clear?
- Should the below grammar be included? The existing spec lists the bag op argument as *<sfw_query>* but the parser and SQL++ extended spec use *<query>*
  - For example, in the current spec grammar, `SELECT * FROM << 1, 2, 3 >> UNION << 4, 5, 6 >>` in invalid, but the current parser supports these.
- What about the operators for structs? Should we have separate functions (specifically not operators) for the union etc. of two Ion structs? See future possibilities.

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
    (GROUP BY <expr_query> (AS <variable>)? (, <expr_query> (AS <variable)?)*)?
    (HAVING <expr_query>)?
    ((UNION|INTERSECT|EXCEPT) ALL? <query>)?
    (ORDER BY <sort_item> (, <sort_item>)*)?
    (LIMIT <expr_query>)?
    (OFFSET <expr_query>)?
  ;

expr_query
  : ( <sfw_query> )
  | <path_expr>
  | <function> ( (<expr_query> (, <expr_query>)*)? )
  | { (<expr_query> : <expr_query> (, <expr_query> : <expr_query>)*)? }
  | [ (<expr_query> (, <expr_query>)*)? ]
  | << (<expr_query> (, <expr_query>)*)? >>
  | <sql_scalar_expr>
  | <literal>
  ;

path_expr
  : <variable>
  | ( <expr_query> )
  | <path_expr> . <attr_name>
  | <path_expr> . *
  | <path_expr> [ <expr_query> ]
  | <path_expr> [ * ]
  ;
```

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
SELECT * FROM { 'A': 0, 'B': 1 } UNION { 'A': 2, 'B': 3, 'C': 4 }
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
SELECT * FROM { 'A': 0, 'B': 1 } UNION { 'A': 2, 'B': 3, 'C': 4 }
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
SELECT * FROM { 'A': 0, 'B': 1 } INTERSECT { 'A': 2, 'B': 3, 'C': 4 }
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

SELECT * FROM { 'first_names': ['jack', 'jill'], 'last_names': ['smith', 'jones', 'johnson'] } INTERSECT { 'first_names': ['jack', 'john'], 'last_names': ['smith', 'snow'] }
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

SELECT * FROM { 'A': 0, 'B': 1 } EXCEPT { 'A': 0, 'B': 3 }
   |
==='
<<
  {
    'B': << 1 >>
  }
>>
---
OK!

SELECT * FROM { 'first_names': ['jack', 'jill'], 'last_names': ['smith', 'jones', 'johnson'] } EXCEPT { 'first_names': ['jack'], 'last_names': ['smith'] }
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

