- Feature Name: Bag Operators: UNION, INTERSECT, EXCEPT
- Start Date: 2022-06-07

# Summary
[summary]: #summary

This RFC defines the PartiQL specification for the operators: UNION, INTERSECT, and EXCEPT.

# Motivation
[motivation]: #motivation

The operators UNION, INTERSECT, and EXCEPT have been defined since SQL-92 specifications, yet these are missing from the PartiQL spec.
In SQL, these operators are the typical bag (set if DISTINCT) union, intersect, and difference respectively for *union-compatible* relations. Unlike SQL, PartiQL is schemaless and therefore these operators are valid for arbitrary expressions — including a SFW block.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Each bag operator has the form *q S q'* where *q* and *q'* are of type *\<query\>* and *S* is the operator. Additionally, the operator may be suffixed with *ALL* which indicates the output may have duplicate elements. In its absence, duplicates are eliminated from the final result. Arguments of the operators are coerced into bags with the following behavior.  

- *NULL* and *MISSING* become the empty bag *<< >>*
- A scalar value *v* becomes the singleton bag *<< v >>*
- An array is coerced to a bag by discarding ordering

The behavior of bag operators is specified by permissive or conventional type checking mode. The typing mode determines if heterogenous arguments are allowed or not allowed. Conventional typing mode will produce an error if a bag operator is used on heterogenous types (??). All six operators are defined in Figure 1.

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

The following examples show how these operators can be used to construct new heterogenous bags. All examples will use permissive typing mode as conventional typing mode is equivalent to the SQL bag operators.

### Example Environment

TODO: I would like some feedback on this example data. In the real world, we don't always control the shape of our data; but a reader may look at this and think "why isn't this flat if employees is homogenous?". As a database schema, this would be an "Employees" table and a "Department" table with a separate relation table. I would like some feedback on example data shape that would avoid this initial reaction.

```
{ 
  'hr': { 
    'employees': <<
      { 'id': 1, 'name': 'Bob Smith',   'title': 'Consultant' }, 
      { 'id': 2, 'name': 'Susan Smith', 'title': 'Manager' },
      { 'id': 3, 'name': 'Jane Smith',  'title': 'DEI'}
    >>
  },
  'accounting': { 
    'employees': <<
      { 'id': 1, 'name': 'Quincy Jones',   'title': 'Director' }, 
      { 'id': 2, 'name': 'James Early Jones', 'title': 'Manager' },
      { 'id': 3, 'name': 'Indiana Jones',  'title': 'Intern'}
    >>
  },
  'engineering': { 
    'employees': <<
      { 'id': 1, 'name': 'Paul McCharmly',   'title': 'Manager' }, 
      { 'id': 2, 'name': 'George Parasol', 'title': 'SDE' },
      { 'id': 3, 'name': 'Ringo Stone',  'title': 'Principal SDE'},
      { 'id': 4, 'name': 'Eric Lennon',  'title': 'Intern'}
    >>
  }
} 
```

### Union Examples

**Union 1** — No manager meeting invite list
```
PartiQL> SELECT id, name FROM (
   |   SELECT * FROM hr.employees
   |   UNION
   |   SELECT * FROM accounting.employees
   |   UNION
   |   SELECT * FROM engineering.employees
   | )
   | WHERE title != 'Manager'
   | ORDER BY title ASC
   |
==='
[
  {
    'id': 1,
    'name': 'Bob Smith'
  },
  {
    'id': 3,
    'name': 'Jane Smith'
  },
  {
    'id': 1,
    'name': 'Quincy Jones'
  },
  {
    'id': 3,
    'name': 'Indiana Jones'
  },
  {
    'id': 4,
    'name': 'Eric Lennon'
  },
  {
    'id': 3,
    'name': 'Ringo Stone'
  },
  {
    'id': 2,
    'name': 'George Parasol'
  }
]
---
OK!
```

**Union 2** — Union where right operands are expressions rather than SFW blocks. Note that this is stated as possible in the SQL++ text, but the grammar does not support this. This is the result of an initial implementation because our parser actually parsers UNION operators as expressions rather than SFW blocks as described by the grammar.

```
PartiQL> SELECT name FROM (
   |   SELECT * FROM hr.employees UNION engineering.employees UNION accounting.employees
   | ) WHERE title = 'Intern'
   |
==='
<<
  {
    'name': 'Eric Lennon'
  },
  {
    'name': 'Indiana Jones'
  }
>>
---
OK!
```

**Union 3** -- Get all distinct titles
```
PartiQL> SELECT DISTINCT title FROM (
   |   SELECT * FROM hr.employees
   |   UNION
   |   SELECT * FROM accounting.employees
   |   UNION
   |   SELECT * FROM engineering.employees
   | )
   | ORDER BY title ASC
   |
==='
[
  {
    'title': 'Consultant'
  },
  {
    'title': 'DEI'
  },
  {
    'title': 'Director'
  },
  {
    'title': 'Intern'
  },
  {
    'title': 'Manager'
  },
  {
    'title': 'Principal SDE'
  },
  {
    'title': 'SDE'
  }
]
---
OK!
```

**Union 4** — Combining heterogenous collections

```
{
  'colors': {
    'rgb': <<
      { 'red': [ 255, 0, 0 ] },
      { 'green': [ 0, 255, 0 ] },
      { 'blue': [ 0, 0, 255 ] },
      { 'black': [ 0, 0, 0 ] }
    >>,
    'cmyk': <<
      { 'name': 'cyan', 'CMYK': [ 1, 0, 0, 0 ] },
      { 'name': 'magenta', 'CMYK': [ 0, 1, 0, 0 ] },
      { 'name': 'yellow', 'CMYK': [ 0, 0, 1, 0 ] },
      { 'name': 'black', 'CMYK': [ 0, 0, 0, 1 ] }
    >>
  }
}
```

```
-- RGB values of all colors
SELECT * FROM colors.rgb
UNION ALL
PIVOT [
  255*(1-c.CMYK[0])*(1-c.CMYK[3]),
  255*(1-c.CMYK[1])*(1-c.CMYK[3]),
  255*(1-c.CMYK[2])*(1-c.CMYK[3])
] AT c.name FROM colors.cmyk as c

PartiQL> SELECT * FROM colors.rgb
   | UNION ALL
   | PIVOT [
   |   255*(1-c.CMYK[0])*(1-c.CMYK[3]),
   |   255*(1-c.CMYK[1])*(1-c.CMYK[3]),
   |   255*(1-c.CMYK[2])*(1-c.CMYK[3])
   | ] AT c.name FROM colors.cmyk as c
   |
==='
<<
  {
    'red': [
      255,
      0,
      0
    ]
  },
  {
    'green': [
      0,
      255,
      0
    ]
  },
  {
    'blue': [
      0,
      0,
      255
    ]
  },
  {
    'black': [
      0,
      0,
      0
    ]
  },
  {
    'cyan': [
      0,
      255,
      255
    ],
    'magenta': [
      255,
      0,
      255
    ],
    'yellow': [
      255,
      255,
      0
    ],
    'black': [
      0,
      0,
      0
    ]
  }
>>
---
OK!
```

### Intersect Examples

**Intersect 1** — Find common job titles across ALL organizations.

```
PartiQL> SELECT title FROM (
   |   SELECT * FROM hr.employees
   |   INTERSECT
   |   SELECT * FROM accounting.employees
   |   INTERSECT
   |   SELECT * FROM engineering.employees
   | )
   | ORDER BY title ASC
   |
==='
[
  {
    'title': 'Manager'
  }
]
---
OK!
```

### Except Examples

**Except 1** — Find all job titles unique to the engineering department.

```
PartiQL> SELECT title FROM engineering.employees EXCEPT (
   |   SELECT title FROM hr.employees
   |   UNION
   |   SELECT title FROM accounting.employees
   | )
   |
==='
<<
  {
    'title': 'SDE'
  },
  {
    'title': 'Principal SDE'
  }
>>
---
OK!
```

# Drawbacks
[drawbacks]: #drawbacks

We do not recognized drawbacks to implementing these features as these are clearly defined in the specification yet missing from the implementation.

TODO: Stating no drawbacks might come off as indolent or naive.

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

- Do we want other configuration to allow for singleton bag of *<< NULL >>* and *<< MISSING >>* rather than the empty bag?
- Should we allow for combining heterogenous bags in conventional mode? How would we type check all arguments of the bag?
- Is the definition for `INTERSECT_ALL` too clunky? Is it correct and clear?
- Should the below grammar be included? The existing spec lists the bag op argument as *\<sfw_query\>*.
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
