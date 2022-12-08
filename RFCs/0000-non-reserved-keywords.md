# Information

- Start Date: 2022-11-18
- PartiQL Issue [#719](https://github.com/partiql/partiql-lang-kotlin/issues/719)
- RFC PR: (TODO: fill me in with the PR link once PR is created)

# Summary
[summary]: #summary

This document proposes establishing PartiQL's list of non-reserved keywords.

# Motivation
[motivation]: #motivation

All keywords specified in the PartiQL grammar cannot act as identifiers within the
PartiQL language. The implication of this scenario is that many keywords, which are unimplemented and may not be
implemented in the near future, have been prematurely reserved and cannot be leveraged as identifiers for our users
to freely use. Along with unimplemented keywords, keywords with limited grammatical scope have been reserved and, thus,
have been removed from PartiQL's customers' repertoire of possible identifiers (even when grammatical context may allow it).

There has been an external customer request asking about PartiQL's intention of allowing non-reserved keywords (see
PartiQL Issue #719 above) -- especially as other SQL-languages have adopted similar concepts. The code changes required
to provide this support to PartiQL's customers are fairly straightforward.

# Guide-Level Explanation
[guide-level-explanation]: #guide-level-explanation

This document proposes drawing a distinction between **reserved** and **non-reserved** keywords within PartiQL. We shall state
that reserved keywords are keywords that may not, under any circumstances, act as identifiers within a PartiQL query unless
wrapped in quotations. Non-reserved keywords, on the other hand, are keywords that may be used as identifiers without the
use of quotations if the current context allows it.

For example, a user may be querying a dataset containing a column name `user`. Currently, as PartiQL reserves all
keywords, this keyword is unable to be referenced unless by wrapping the keyword in quotations. See below:

```partiql
SELECT "user".username, "user".uuid, "user".firstName, "user".lastName
FROM users
LET users AS "user"
WHERE "user".age > 21
```

An important thing to note is that while `user` is actually reserved in PartiQL, it is **not** used in any of the existing
grammar rules. It is, in fact, a hidden keyword that cannot be leveraged by our customers. There are several similar examples
of unused keywords in the existing PartiQL implementation.

Nevertheless, by adding `user` to the list of non-reserved keywords, it is possible for users to write queries such as:

```partiql
SELECT user.username, user.uuid, user.firstName, user.lastName
FROM users
LET users AS user
WHERE user.age > 21
```

As another example, a user may be querying a dataset containing a column name `leading`:

```partiql
SELECT leading
FROM [ { 'leading': 0 }, { 'leading': 1 } ] AS src
WHERE leading > 0
```

You might have thought that the above query might be wrong, but in PartiQL, it's not. While it is true that `leading`
is a keyword in many SQL languages, it is actually surprisingly not in PartiQL.

Let's take a look at how PartiQL gets around
this by examining the existing rule in the Kotlin codebase (we'll ignore Rust for now):

```antlrv4
grammar PartiQL;

trimFunction
    : TRIM PAREN_LEFT ( specification=IDENTIFIER? substring=expr? FROM )? target=expr PAREN_RIGHT;
```

The `TRIM` specification, in PartiQL, allows a `specification` (an identifier) in its grammar, which may lead you
to believe that any identifier is allowed as the first parameter. However, upon visiting the internal parse tree, we actually can see
that we assert that the `specification` is one of `LEADING`, `TRAILING`, or `BOTH`.

Taking a step back, we can see that the Kotlin implementation of PartiQL has unintentionally added support for non-reserved
keywords. Yes, it requires more manual coding in the backend to assert the correct inputs, but we are able to successfully
use `LEADING`, `TRAILING`, and `BOTH` as identifiers and as the specification for `TRIM`.

By providing our customers with a formal table of keywords (and their distinctions) and adding formal support for
non-reserved keywords, we can better enable customers to read our grammar specifications with ease,
provide our contributors with an easier way to add non-reserved keywords, and give users the ability to use keywords directly
when the context allows them.

# Reference-Level Explanation
[reference-level-explanation]: #reference-level-explanation

In the section above, the last snippet of the last paragraph alludes to an important component of this RFC: context.
By introducing non-reserved keywords, it is vital to test and verify any addition of non-reserved keywords in the contexts
where they both act as keywords and as identifiers.

The example regarding `LEADING` given in the section above provides us with a simple example to think about. The only scope
in which `LEADING` acts as a keyword is when it is the first argument of the `TRIM` function. With that in mind, any other
scenario will be treated as an identifier. Take the following query:

```
SELECT leading + TRIM(LEADING leading FROM leading)
FROM t
```

We can interpret the above query as follows:
- The first `leading` instance in the query (in the projection list) represents an identifier
- The second represents a trim specification keyword
- The third and fourth represent identifiers

```
SELECT leading + TRIM(LEADING FROM leading)
FROM t
```

Similarly, in the above query:
- The first `leading` instance in the query represents an identifier
- The second represents a trim specification
- The third represents an identifier

With that in mind, we may write the ANTLR rule (for the Kotlin codebase) as follows:

```antlrv4
grammar PartiQL;

trimFunction
    : TRIM PAREN_LEFT (specification=(BOTH|LEADING|TRAILING)? substring=expr? FROM)? target=expr PAREN_RIGHT;

expr
    : symbolPrimitive
    | otherExpressions
    ;

symbolPrimitive
    : IDENTIFIER
    | IDENTIFIER_QUOTED
    | nonReservedKeyword
    ;

nonReservedKeyword
    : BOTH
    | LEADING
    | TRAILING
    ;
```

TODO: Add closer

While this is implementation-specific, it does give insight into how this may work for the Kotlin codebase. This is
exactly how the
[Proof-of-Concept (POC) PR is implemented in partiql-lang-kotlin](https://github.com/partiql/partiql-lang-kotlin/pull/780)
(which uses an LL parser). As for the upcoming Rust implementation, there is another
[POC PR that shows the same functionality](https://github.com/partiql/partiql-lang-rust/pull/221) for the LALR parser.

# Drawbacks
[drawbacks]: #drawbacks

The two immediately resounding reasons to not do this are:
1. We may require the use of a non-reserved keyword in a manner that requires its reservation
2. We may switch the parser we use and may be unable to provide the same experience

As for #1, we may gain insight from how
[PostgresQL allows the use of non-reserved keywords as identifiers](https://www.postgresql.org/docs/current/sql-keywords-appendix.html).
The language highly recommends users to continue wrapping keywords with quotations, as the list of non-reserved keywords is subject
to change between versions. This may not be the approach taken by PartiQL, but it is an option.

As for #2, many parser-generators are powerful enough to disambiguate non-reserved keywords from reserved keywords (as
proven by the PartiQL Kotlin draft PR [#780](https://github.com/partiql/partiql-lang-kotlin/pull/780), the PartiQL
Rust POC PR [#221](https://github.com/partiql/partiql-lang-rust/pull/221), the PostgresQL implementation, and other SQL
parsers). However, it remains possible that a future language implementation may not be able to accommodate non-reserved
keywords.

These two drawbacks can be compiled into a single drawback of possibly not being able to provide the same functionality
in the future. A possible solution to this may be proving the ability to parse with all available implementations via
thorough testing and verification (such as the two mentioned POC PRs).

# Rationale and Alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

This proposal provides sufficient ergonomics for both the implementer and the user by providing native support for
non-reserved keywords. Other designs include the existing implementation of `TRIM`. This implementation is not ideal, as
it allows two optional identifiers to be placed next to each other. The implication of this is that if a single identifier
is passed, additional logic regarding syntax needs to be handwritten -- does the single identifier represent a trim specification?
Or, does it represent an identifier? See the [existing logic of trim](https://github.com/partiql/partiql-lang-kotlin/pull/780/files#diff-4e5ed61390667b320ddd501e8b31aa60cced016260a2919bd3c94818ae817f42R1016).

The impact of not doing this relates to how PartiQL compares to other SQL languages when it comes to their feature set.
Users of other SQL engines have access to non-reserved keywords, and as PartiQL is attempting to be the de facto
database-independent language, it is in PartiQL's best interest to handle non-reserved keywords.

# Prior Art
[prior-art]: #prior-art

While I've already linked the PostgreSQL keywords, the Kotlin POC PR, and the Rust POC PR, I'll link some additional resources:
- [IBM Netezza SQL Non-Reserved Keywords](https://www.ibm.com/docs/en/psfa/7.2.1?topic=keywords-non-reserved)
- [PSQL Reserved Keywords (and the words to avoid)](https://docs.actian.com/psql/psqlv13/index.html#page/sqlref/sqlkword.htm)
- [MySQL Reserved/Non-Reserved Distinctions](https://dev.mysql.com/doc/refman/8.0/en/keywords.html)
- [Greenplum Database Reserved/Non-Reserved Keywords](https://gpdb.docs.pivotal.io/6-1/ref_guide/sql-keywords.html)
- [Hive Reserved/Non-Reserved Keywords](https://docs.treasuredata.com/display/public/PD/Hive+Reserved+and+Non-Reserved+Keywords)
- [Snowflake Reserved/Limited Keywords](https://docs.snowflake.com/en/sql-reference/reserved-keywords.html)


# Unresolved Questions
[unresolved-questions]: #unresolved-questions

I'd like to gain insight into where we can establish the list of official keywords and their reservation designation.

1. Will all keywords be listed in the specification?
  - I'm unsure whether this is necessary. Should we add them to our documentation repository instead?
2. How can we effectively determine which keywords are non-reserved?
  - As both LL parsers and LR parsers are increasingly common in industry, my intuition pointed towards proving that the non-reserved
  keyword may be disambiguated by both types of parsers before being added to the official table. In the case of both POC PRs,
  I've indicated that LEADING, TRAILING, USER, PUBLIC, DOMAIN, ACYCLIC, SIMPLE, and BOTH should be non-reserved.
3. How can we easily indicate a change in the keyword designation?
  - Currently, the only mechanism by which users may receive notice about syntactic changes of PartiQL are through our release notes
  and changelogs. That may remain the same.
4. Is a keyword re-designation a breaking change requiring a major version bump? Or would it require a minor version bump?
  - I'd like to say it requires a minor version bump, but I'm not absolutely certain.
  - [MySQL](https://dev.mysql.com/doc/refman/8.0/en/keywords.html) seems to indicate that moving from non-reserved to reserved only
  requires a patch release (which I am not in support of).

# Future Possibilities
[future-possibilities]: #future-possibilities

Eventually, PartiQL should establish a comprehensive table of both reserved and non-reserved keywords to better inform
our users of what they may use as identifiers.

Beyond this, I cannot think of any other future possibilities at the moment.

