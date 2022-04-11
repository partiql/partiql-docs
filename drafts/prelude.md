<!-- 
Copyright Amazon.com, Inc. or its affiliates. All Rights Reserved.
SPDX-License-Identifier: CC-BY-SA-4.0 
-->

# PartiQL Prelude

## Terminology

1. **PartiQL Library**: The open source PartiQL library components which implement the **PartiQL language**.
2. **PartiQL Host**: An application/service/library that uses some or all of the PartiQL library, potentially extending its functionality.
3. **end user**: A user of a **PartiQL Host** who issues queries/commands in the PartiQL language.
5. **expression language**: The subset of PartiQL that enables referencing, combining, and transforming values (e.g., `a+b*10`, `lower(col1) LIKE 'exception'`).
4. **Query**: The subset of PartiQL relevant to querying data; This includes SQL-style `SELECT ... FROM ... WHERE ...` (`SFW`) queries as well as **expression language** queries.
5. **DML**: **D**ata **M**anipulation **L**anguage -- The subset of PartiQL relevant to inserting/updating data (e.g., `UPDATE ... WHERE ...`, `INSERT INTO ... VALUES ...`).
6. **DDL**: **D**ata **D**efinition **L**anguage -- The subset of PartiQL relevant to specifying schema.

## Background

The PartiQL library needs to support a spectrum of PartiQL hosts. Some of these hosts will wish to embed and expose the total PartiQL functionality, while some will wish to expose only a subset. Thus PartiQL needs to support 'dialects' of the PartiQL language. For example, CLI application host for querying against an `Ion` or `json` file may wish to expose all query functionality but no DML/DDL. On the other hand, a host that translates PartiQL queries to a series of manipulations against a (possibly remote) key-value store may wish to expose Query/DML/DDL but with none of the *built-in* functions and operators that PartiQL provides.

A complete discussion of dialects (i.e., Query vs. DML vs. DDL and which clauses are supported) is outside the scope of this document. Herein is proposed a mechanism to model the ability to define the set of PartiQL functions and operators that a host makes available to end users for use in the expression language of PartiQL.

## A Model for function imports

The general idea is to model the functions and operators available to the expression language as a set of namespaced functions that are made available un-namespaced to the end users via the use of a "prelude". The prelude consists of a number of `import` statements that hoist functions out of their namespaces and into the "top-level" namespace. 

At the very least, this serves as a conceptual model to explain and document the expression language features available to end-users. 

*Note* For now, only hoisting is specified -- there is no 'renaming' of imported functions.
*Note* There is no ability provided to allow end users to use the un-imported, namespaced functions; only functions imported into the 'top-level' will be available to the expression language.

### Functions and Namespaces

*NOTE* the syntax in this section is illustrative for purposed of explaining the namespacing, not normative. PartiQL library implementations will have implementation-specific plugin mechanisms.

PartiQL will model vended (and, where supported, host-injected) functions as though they were defined with namespacing and types similar to the below. 

Selected SQL92 specified functions (`sql92` and `string` are namespace components, each `fn` line defines a function signature):

```
    sql92 {
        ...
        string {
            fn || (l: text, r: text) => text;
            fn concat(...args: text) => text;
            fn lower(s: text) => text;
            fn upper(s: text) => text;
            fn trim(LEADING? chars?:text FROM str:text) => text;
            fn trim(LEADING? FROM? str:text, chars?:text) => text;
            ...
        }
        ... 
    }
```

Selected Native PartiQL functions:

```
    partiql {
        ...
        string {
            fn btrim(str:text, chars?:text) => text;
            fn ltrim(str:text, chars?:text) => text;
            fn rtrim(str:text, chars?:text) => text;
            ...
        }
		struct {
		    ...
		}
		sequence { // e.g. list, bag, sexp
		    ...
		}
		list {
		    ...
		}
	    set {
		    ...
		}
		bag {
		    ...
		}
		sexp {
		    ...
		}
        ...
    }
```

Straw-man host-vended functions:

```
    my_application {
        ...
        string {
            fn my_specialized_trim(str:text, chars?:text) => text;
            ...
        }
        ...
    }
```

### Prelude

PartiQL hosts will define a 'prelude' script (Cf. the [Rust prelude](https://doc.rust-lang.org/std/prelude/) and the [Haskell Prelude](https://hackage.haskell.org/package/base-4.16.1.0/docs/Prelude.html)) to describe the functions that are available to end-users of a hosted application of PartiQL.

An example of a 'complete' usage of available PartiQL functions might look something like (where `**` and `*` are analogous to shell-globing, so `**` means 'match all sub-namespaces, recursively' and `*` means 'match all functions`): 

```
    import sql92::**::*;
    import partiql::**::*;
    import my_application::**::*;
```

While a service host that wishes to translate a PartiQL query into a simple key-value lookup might have an empty prelude:

```
    // import nothing

```

<!--  LocalWords:  namespace PartiQL DML DDL namespaced namespaces
 -->
<!--  LocalWords:  namespacing
 -->
