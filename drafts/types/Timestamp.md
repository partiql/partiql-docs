# Timestamp Type in PartiQL

## Background

As of today, PartiQL’s timestamp type is based on `ion`. This document covers the syntax, semantics and example usages of generic `Timestamp` types supported in PartiQL. 

## Syntax

### SQL standard

SQL 92 Grammar:

```
-- TIMESTAMP TYPE
<timestamp type> ::= [ <left paren> <timestamp precision> <right paren> ] 
                        [ WITH TIME ZONE ]
<timestamp precision> ::= <time fractional seconds precision>   
<time fractional seconds precision> ::= <unsigned integer>                  

-- TIMESTAMP literal
<timestamp literal>  ::= TIMESTAMP <timestamp string>
<timestamp string>  ::= 
    <quote> <date value> <space> <time value> [ <time zone interval> ] <quote>
<time value> ::= <hours value> <colon> <minutes value> <colon> <seconds value>     
<time zone interval>  ::= <sign> <hours value> <colon> <minutes value>
<date value> ::= <years value> <minus sign> <months value> <minus sign> <days value>
<years value> ::= <datetime value>
<datetime value> ::= <unsigned integer>
<months value> ::= <datetime value>
<days value> ::= <datetime value>
<hours value>  ::= <datetime value>
<minutes value> ::= <datetime value>
<seconds value> ::= <seconds integer value> [ <period> [ <seconds fraction> ] ]
<seconds integer value> ::= <unsigned integer>
<seconds fraction> ::= <unsigned integer>
<sign>  ::= <plus sign> | <minus sign>
<plus sign> ::= +
<minus sign> ::= -
<colon> ::= :      

```

SQL timestamp Type:

|Syntax	|	|
|---	|---	|
|TIMESTAMP	|A timestamp type, without time zone  without fraction seconds precision	|
|TIMESTAMP(p)	|A timestamp type, without time zone, with fraction seconds precision	|
|TIMESTAMP WITH TIME ZONE	|A timestamp type, with time zone, without fraction seconds precision	|
|TIMESTAMP(p) WITH TIME ZONE	|A timestamp type, with timezone, with fraction seconds precision	|

### PartiQL

An valid SQL timestamp string and a valid ~~ISO 8601~~ [RFC 3339](https://www.rfc-editor.org/rfc/rfc3339) timestamp string is a valid PartiQL Timestamp string.

```
<timestamp literal>             ::= TIMESTAMP[ <left paren> <timestamp precision> <right paren> ]
                                    <timestamp string>
<timestamp string>              ::= <sql standard timestamp string>
                                  | <RFC 3339 timestamp string> 
<sql standard timestamp string> ::= <quote> <date value> <space> <time value> 
                                    [ <time zone interval> ] <quote>
<time value>                    ::= <hours value> <colon> 
                                    <minutes value> <colon> 
                                    <seconds value>     
<time zone interval>            ::= <sign> <hours value> <colon> <minutes value>
<date value>                    ::= <years value> <minus sign> 
                                    <months value> <minus sign> 
                                    <days value>
<years value>                   ::= <datetime value>
<datetime value>                ::= <unsigned integer>
<months value>                  ::= <datetime value>
<days value>                    ::= <datetime value>
<hours value>                   ::= <datetime value>
<minutes value>                 ::= <datetime value>
<seconds value>                 ::= <seconds integer value> 
                                    [ <period> [ <seconds fraction> ] ]
<seconds integer value>         ::= <unsigned integer>
<seconds fraction>              ::= <unsigned integer>
<sign>                          ::= <plus sign> | <minus sign>
<plus sign>                     ::= +
<minus sign>                    ::= -
<colon>                         ::= :
<RFC 3339 timestamp string>     ::= <quote> <date value> T <time value> 
                                    <RFC 3339 offset>
<RFC 3339 offset>               :: = Z
                                   | <time zone interval>                                                                                                                                                                                                                       
```

NOTE:
* SQL spec and RFC 3339 both agrees that: Within a <datetime literal>, the <years value> shall contain four digits. The <seconds integer value> and other datetime components, with the exception of <seconds fraction>, shall each contain two digits.  
  ** the "T" and "Z" characters in this syntax may alternatively be lower case "t" or "z" respectively.
  *** Notice RFC 3339 requires a timestamp to have “full time”, which include the time offset

Example:

```
// The following are required to be parse by a compatible PartiQL implementation
// SQL style
TIMESTAMP    '2023-06-01 00:00:00' // With no second fraction
TIMESTAMP    '2023-06-01 00:00:00.0000' // Arbitrary second fraction
TIMESTAMP(0) '2023-06-01 00:00:00' // With precision zero
TIMESTAMP(1) '2023-06-01 00:00:00' // With precision, no second fraction
TIMESTAMP(1) '2023-06-01 00:00:00.000' // with Precision, second fraction exceeding precision
TIMESTAMP    '2023-06-01 00:00:00-07:00' // with offset 

// RFC 3339 style Timestamp
TIMESTAMP    '2023-06-01T00:00:00+00:00' // UTC with explicit offset 
TIMESTAMP    '2023-06-01T00:00:00Z' // UTC with Z offset
 TIMESTAMP    '2023-06-01T00:00:00z' // UTC with z (lower case) offset
TIMESTAMP    '2023-06-01t00:00:00z' // Lower case t
TIMESTAMP    '2023-06-01T00:00:00-00:00' // Unknown timestamp
TIMESTAMP    '2023-06-01T00:00:00-07:00' // explicit time zone in PDT

// The following shall not be required
TIMESTAMP    '2023-06-01 00:00:00Z' // using Z with SQL Sytle timestamp string
TIMESTAMP    '2023-06-01' // SQL Style string, incomplete part 
TIMESTAMP    '2023-06' // SQL style string, incomplete part
TIMESTAMP    '2023-06-01T00:00:00' // RFC 3339 style, no time zone
TIMESTAMP    '2023-06-01T' // RFC 3339 style, incomplete part
TIMESTAMP    '2023-06T'   // RFC 3339 style, incomplete part
```

Open question:

* Do we want to allow using `TIMESTAMP WITH TIME ZONE` constructor?
    * For example: `TIMESTAMP WITH TIME ZONE '2023-01-01 00:00:00'`
    * We currently allows constructing a TIME literal using `TIME WITH TIME ZONE` constructor.
        * The decision is modeled after postgresql as far as I know.
        * However, postgresql also disallow the creation of a Time/Timestamp literal of type `TIME/TIMESTAMP WITH TIME ZONE` using the TIME/TIMESTAMP constructor.
        * For example: `TIMESTAMP '2023-01-01 00:00:00-07:00'` will be treated as a `TIMESTAMP WITHOUT TIME ZONE` type.
        * This is a behavior that breaks SQL spec.
    * If so, we need to decide what the semantics is for such literal. (unknown time zone vs append session offset)
        * Semantics section below discussed more about this.

## Type Semantics

**Assumption**
The PartiQL Timestamp should be able to fully express an Ion Timestamp, meaning we need `UNKNOWN OFFSET` ([RFC3339](http://www.ietf.org/rfc/rfc3339.txt)) and support arbitrary precision second fraction.

|Type	|SQL Semantics	|PartiQL Semantics	|
|---	|---	|---	|
|TIMESTAMP	|TIMESTAMP(6)	|A Timestamp with no time zone, with arbitrary number of digits in second fraction.	|
|TIMESTAMP(p)	|A Timestamp with no time zone, with p digits in second fraction. 	|Compatible with SQL. 	|
|TIMESTAMP WITH TIME ZONE	|TIMESTAMP(6) WITH TIME ZONE	|A timestamp with  time zone, with arbitrary number of digits in second fraction.	|
|TIMESTAMP(p) WITH TIME ZONE	|A Timestamp with time zone, with p digits in second fraction	|Compatible with SQL. 	|

** SQL spec states that if <timestamp precision> is not specified, then `p` should be set to `6` implicitly.  In PartiQL, `TIMESTAMP` type without <timestamp precision> means the underlying data can have unlimited digits in fraction second.


## Value Semantics

A timestamp value is made up with **date time fields**.

|Keyword	|Meaning	|Valid values	|
|---	|---	|---	|
|YEAR	|Year	|0001 to 9999	|
|MONTH	|Month within year	|01 to 12	|
|DAY	|Day within month	|Based on Gregorian calendar	|
|HOUR	|Hour within day	|00 to 23	|
|MINUTE	|Minute within hour	|00 to 59	|
|SECOND	|Second and possibly second fraction within a minute	|00 to 61.9(N) where 9(N) indicates a sequence of N instances of digit 9 (leap second) 	|
|TIMEZONE_HOUR	|Hour value of timezone displacement	|-23 to 23, unkown premited	|
|TIMEZONE_MINUTE	|Minute value of time zone displacement	|-59 to 59, unknown premited 	|

For the non-optional date time fields `YEAR`, `MONTH`, `DAY`, `HOUR`, `MINUTE`, `SECOND`, there is an ordering of significance. This is, from most significant to least significant: `YEAR`, `MONTH`, `DAY`, `HOUR`, `MINUTE`, `SECOND`.

Note: The choice of setting max timezone offset to `+23:59` and min timezone offset to `-23:59` is coming from ION. In SQL standard the value is between `+14:00` to `-14:00`.

### Timestamp value of Type `TIMESTAMP WITHOUT TIME ZONE`

* The timestamp value is made up with date time fields `YEAR`, `MONTH`, `DAY`, `HOUR`, `MINUTE`, `SECOND`.
* The timestamp value **MAY** represent **local time**.
* If a timestamp value with time zone is to be implicitly derived from one without(for example, in a simple assignment operation), PartiQL assumes the value without time zone to be local, subtracts the current default time zone displacement of the Session from the value to give UTC, and associate that time zone displacement with the result.

```
SET TIME ZONE to 'PDT' // presudo syntax

CAST(TIMESTAMP '2023-06-01 00:00:00' AS TIMESTAMP WITH TIME ZONE)
-- 2023-06-01 00:00:00-07:00
```

Example:

```
TIMESTAMP '2023-06-01 00:00:00'
-- Year: 2023, Month: 6, day: 1, 
-- hour: 0, minute: 0, second: 0. (no fraction second)
TIMESTAMP '2023-06-01 00:00:00.0000'
-- Year: 2023, Month: 6, day: 1, 
-- hour: 0, minute: 0, second: 0.0000 (preserve the fraction second)
TIMESTAMP(0) '2023-06-01 00:00:00.0000'
-- Year: 2023, Month: 6, day: 1, 
-- hour: 0, minute: 0, second: 0. 
-- implementation defined rounding/truncation the fraction second based on precision
```

>PartiQL will make no assumption about time zone displacement for value of type `TIMESTAMP WITHOUT TIME ZONE`. However, should a time zone displacement be required during subsequent processing, the current default time zone displacement of session will be applied at that time.


Reference:
PostgreSQL

* In PostgreSQL, value of TIMESTAMP is an absolute time(always UTC), and it does not change based on session offset. However, when casting to `TIMESTAMP WITH TIME ZONE`, it assumes the timestamp value to be local time.
* I think this behavior is quite confusing.
* https://www.db-fiddle.com/f/tTJrrjHou1YPVMEitfXKrs/1

MySQL

* MySQL converts `TIMESTAMP` values from the current time zone to UTC for storage, and back from UTC to the current time zone for retrieval.
* Seemingly mysql has dropped the concept of `TIMESTAMP WITHOUT TIME ZONE` completely, and rely on session time if timezone interval has not been set.
* https://www.db-fiddle.com/f/eUqJk2NujXmN2sicZ393fo/0

### Timestamp value of Type `TIMESTAMP WITH TIME ZONE`

* The timestamp value is made up with date time fields `YEAR`, `MONTH`, `DAY`, `HOUR`, `MINUTE`, `SECOND`, `TIMEZONE_HOUR`, `TIMEZONE_MINUTE`.
* The timestamp value **MUST** represent UTC.
* If a timestamp value without time zone is to be implicitly derived from on with, PartiQL assumes the value with time zone to be UTC, adds the time zone displacement to give local time, and the result, without any time zone displacement, is **local**.

```
SET TIME ZONE to 'PDT' // presudo syntax

CAST(TIMESTAMP '2023-06-01 00:00:00+00:00' AS TIMESTAMP WITHOUT TIME ZONE)
-- 2023-05-31 17:00:00
```

See more about conversion logic between all date time types at Appendix:

* Special Note about `UNKNOWN TIME ZONE`(-00:00):
    * Per RFC 3339:  If the time in UTC is known, but the offset to local time is unknown, this can be represented with an offset of "-00:00".  This differs semantically from an offset of "Z" or "+00:00", which imply that UTC is the preferred reference point for the specified time.
    * A timestamp value whose time zone value is unknown shall have `TIMEZONE_HOUR`, `TIMEZONE_MINUTE` field set to `UNKNOWN`. The exact representation of UNKNOWN is implementation defined.
    * The timestamp value of UNKNOWN TIME ZONE(-00:00) is equal to the a timestamp value with the same `YEAR`, `MONTH`, `DAY`, `HOUR`, `MINUTE`, `SECOND` fields of UTC timezone (+00:00 or Z)
    * An Unknown `TIMESTAMP_HOUR` **SHALL NOT** be equal to a `TIMESTAMP_HOUR` whose value is 0. An unknown `TIMESTAMP_MINUTE` **SHALL NOT** be equal to a `TIMESTAMP_MINUTE` whose value is 0.
        * This is debatable. If we want this, we have to force every implementation to preserve the original timezone_hour and timezone_minute at runtime, and store those information at storage layer.
        * See more about this point in following section about Extract function.

Example:

```
TIMESTAMP '2023-06-01 00:00:00+00:00'
-- Year: 2023, Month: 6, day: 1, 
-- hour: 0, minute: 0, second: 0. (no fraction second)
-- timezone_hour: 0, timezone_minute : 0

TIMESTAMP '2023-06-01 00:00:00-05:30'
-- Year: 2023, Month: 6, day: 1, 
-- hour: 0, minute: 0, second: 0.
-- timezone_hour: -5, timezone_minute: -30

TIMESTAMP(0) '2023-06-01 00:00:00-00:00'
-- Year: 2023, Month: 6, day: 1, 
-- hour: 0, minute: 0, second: 0. 
-- timezone_hour: UNKNOWN, timezone_minute: UNKNOWN
```

>For value of type TIMESTAMP WITH TIME ZONE, the <timestamp literal> is interpreted as local time with the specified time zone displacement. However, it is effectively converted to UTC while retaining the original time zone displacement.

Reference:
PostgreSQL:

* In PostgreSQL value of TIMESTAMP is also an absolute time.
* https://www.db-fiddle.com/f/wDNjiHXQyiRp2zBJi1jqNw/0

### **Comparison**

* SQL spec leverage the semantics of Interval Types to explain timestamp value comparison, but essentially, it is saying that: comparison results obey the natural rules associated with date and times and yields validate results according to the Gregorian calendar.

The result of comparison should be the same as following: (not indicating implementation algorithm)

* TIMESTAMP WITHOUT TIME ZONE <op> TIMESTAMP WITHOUT TIME ZONE
    * Compare from most significant field to least significant field.
* TIMESTAMP WITH TIME ZONE <op> TIMESTAMP WITH TIME ZONE.
    * Comparing the two timestamp value based on chronological order.
* WOLG: TIMESTAMP WITH TIME ZONE <op> TIMESTAMP WITHOUT TIME ZONE
    * Convert the timestamp value with out timezone to TIMESTAMP WITH TIME ZONE.
    * Comparing the two timestamp value based on chronological order.

Reference:
PostgreSQL:

* https://www.db-fiddle.com/f/2NwxuQeVpr5wryJiupwHvX/0

### Extract

* Syntax: `EXTRACT(edp FROM t)`
* If `t` is of type `TIMESTAMP WITHOUT TIME ZONE`, `edp` can be `YEAR`, `MONTH`, `DAY`, `HOUR`, `MINUTE`, `SECOND`.
* If `t` is of type `TIMESTAMP WITH TIME ZONE`, `edp` can be `YEAR`, `MONTH`, `DAY`, `HOUR`, `MINUTE`, `SECOND`.
* It should returns an numeric value representing the corresponding field value.

Example:

```
EXTRACT (YEAR FROM TIMESTAMP '2023-06-01 11:00:00') 
-- 2023
EXTRACT (SECOND FROM TIMESTAMP '2023-06-01 11:00:00.0000000') 
-- 0.0000000
```

Open question:

* What to do when attempting to extract timezone related information from `TIMESTAMP WITHOUT TIME ZONE`

```
EXTRACT (TIMEZONE_HOUR FROM TIMESTAMP '2023-06-01 11:00:00')
-- ERROR or null
-- I think it should be an error
-- as this is semantically/conceputally no timezone_hour
```

* Do we want implementation to store the original timezone offset as entered in timestamp literal (at run time/storage layer)?

```
CREATE TABLE foo (
    TIMESTAMP WITH TIME ZONE bar
)

INSERT INTO foo (bar) VALUES (TIMESTAMP '2023-06-01 11:00:00-07:00') // foo syntax
EXTRACT (TIMEZONE_HOUR FROM bar)
-- -7 or depends on session? 
EXTRACT (HOUR FROM bar)
-- 11 or depends on session?
```

* What to do when attempting to extract time zone related information if the time zone is unknown:

```
EXTRACT (TIMEZONE_HOUR FROM TIMESTAMP '2023-06-01 11:00:00-00:00')
-- null or depends on session. 
```

Option 1: preserve timezone offset:

* Query running on different session setting shall be able to return the same result.
* Seemingly SQL spec also implies this option.

Option 2: convert to UTC:

* Definitely easy implementation, and arguably easier semantics.
* It may be true that no one needs to preserve the original offset value.
* To my knowledge, everyone except Oracle and HANA just store UTC.


Reference:

* Postgresql
    * Since timestamp in PostgreSQL is just absolute value, the timezone is just session offset if the type is TIMESTAMP WITH TIME ZONE, other field will be the field in corresponding UTC value.
    * If the type is TIMESTAMP WITH OUT TIME ZONE, PostgreSQL throws an error.
    * https://www.db-fiddle.com/f/mtBeggZjsi1uZfFGzieWb8/0

### Ion Serde

* value of type TIMESTAMP WITHOUT TIME ZONE:

```
// assume PDT
SET TIME ZONE = 'PDT' // foo syntax
TIMESTAMP '2023-06-01 00:00:00'

Option 1: Ion Struct with annotation
TIMESTAMP WITHOUT TIME ZONE::{
    year: 2023,
    month: 6
    day: 1,
    hour: 0,
    minute: 0,
    second: 0.
}

Option 2: Ion Timestamp with annotation, use session offset as timezone
TIMESTAMP WITH TIME ZONE::2023-06-01T00:00:00-07:00
```

* value of Type TIMESTAMP WITH TIME ZONE:

```
// assume PDT
SET TIME ZONE = 'PDT' // foo syntax
TIMESTAMP '2023-06-01 00:00:00-04:00'

Option 1: IonTimestamp value, value as originally input
`2023-06-01T00:00:00-04:00`

Option 2: IonTimestamp value, with UTC offset
`2023-06-01T04:00:00+00:00`

Option 3: IonTimestamp value, with Session offset
`2023-05-31T21:00:00-07:00`
```

## Appendix

### Conversion logic

SQL SPEC 6.13 <cast specification> Page 244
We include the conversion logic between all 5 date time type we support for completeness.
When source is :

1. Date:
    1. To `TIME WITHOUT TIME ZONE` : NOT supported.
    2. To `TIME WITH TIME ZONE`: NOT supported.
    3. To `TIMESTAMP WITHOUT TIME ZONE` :
        1. copy date from source value, set hr, minute, second to 0
    4. To `TIMESTAMP WITH TIME ZONE`:
        1. convert from `DATE` to `TIMESTAMP WITHOUT TIME ZONE` (1c)
        2. convert from `TIMESTAMP WITHOUT TIME ZONE` TO `TIMESTAMP WITH TIME ZONE`
2. TIME WITHOUT TIME ZONE:
    1. To `DATE` : NOT supported.
    2. To `TIME WITH TIME ZONE` :
        1. Assumes the value without timezone to be local
        2.  Subtracts the current dafault time zone displacement of Session from it to give UTC, and associate the time zone displacement with the result.
        3. `Cast( TIME '17:00:00.00' TO TIME WITH TIME ZONE)`  => `TIME WITH TIME ZONE 17:00:00.00-07:00`
    3. To TIMESTAMP WITHOUT TIME ZONE:
        1. copy date from `CURRENT_DATE` and cope time fields from source value
    4. To TIMESTAMP WITH TIME ZONE:
        1. convert from `TIME WITHOUT TIME ZONE` to `TIMESTAMP WITHOUT TIME ZONE` (2c)
        2. convert from `TIMESTAMP WITHOUT TIME ZONE` to `TIMESTAMP WITH TIME ZONE`
3. TIME WITH TIME ZONE:
    1. To `DATE` : NOT supported.
    2. To `TIME WITHOUT TIME ZONE` :
        1. Assumes the value with time zone to be UTC, add the time zone displacement to it, so the result is local.
        2. In practice it means drop the time zone.
        3. `CAST ( TIME WITH TIME ZONE '17:00:00.00-4:00’ TO TIME)` => `TIME '17:00:00.00'`
    3. To `TIMESTAMP WITHOUT TIME ZONE`
        1. Convert from `TIME WITH TIME ZONE` to `TIMESTAMP WITH TIME ZONE`
        2. Convert from `TIMESTAMP WITH TIME ZONE` to `TIMESTAMP WITH OUT TIME ZONE`
    4. To `TIMESTAMP WITH TIME ZONE`:
        1. copy date from `CURRENT_DATE` and time and time zone from source value.
4. TIMESTAMP WITHOUT TIME ZONE:
    1. To `DATE` : copy date from source value.
    2. To `TIME WITHOUT TIME ZONE` :
        1. Copy time from source value
    3. To `TIME WITH TIME ZONE`
        1. convert from `TIMESTAMP WITHOUT TIME ZONE` to `TIMESTAMP WITH TIME ZONE`
        2. convert from `TIMESTAMP WITH TIME ZONE` to `TIME WITH TIME ZONE`.
    4. To `TIMESTAMP WITH TIME ZONE`
        1. Similar to the conversion from  `TIME` to `TIME WITH TIME ZONE`
        2. `CAST ( TIMESTAMP '2023-05-31 17:00:00.00' TO TIMESTAMP WITH TIME ZONE)`  => `TIMESTAMP WITH TIME ZONE '2023-05-31 17:00:00.00-07:00'`
5. TIMESTAMP WITH TIME ZONE:
    1. To DATE : copy date from source value.
    2. To `TIME WITHOUT TIME ZONE` :
        1. convert from `TIMESTAMP WITH TIME ZONE` to `TIMESTAMP WITHOUT TIME ZONE`
        2. convert from `TIMESTAMP WITHOUT TIME ZONE` to `TIME WITHOUT TIME ZONE`
    3. To TIME WITH TIME ZONE
        1. Copy time and time zone field from source value.
    4. TIMESTAMP WITHOUT TIME ZONE
        1. Similar to conversion from TIME WITH TIME ZONE TO TIME
        2. `CAST ( TIMESTAMP WITHOUT TIME ZONE '2023-05-31 17:00:00.00-07:00' TO TIMESTAMP)`  => `TIMESTAMP '2023-05-31 17:00:00.00'`


