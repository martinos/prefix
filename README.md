# Prefix Opclass

This module is written by Dimitri Fontaine with a great amount of help from
RhodiumToad (formely known as AndrewSN), who was the one advising for a
*GiST opclass* to solve the prefix matching problem.

## Presentation

Prefix matching is both very common and important in telephony
applications, where call routing and costs depend on matching
caller/callee phone number to an operator prefix.

Let's say the prefixes table is called `prefixes`, a typical query
will try to match a phone number to the longest prefix in the table:

    SELECT * 
      FROM prefixes
     WHERE prefix @> '0123456789'
    ORDER BY length(prefix) DESC
      LIMIT 1;

## Installation

### debian and ubuntu packages

Replace the PostgreSQL major version number here by the one you're running
(pick either `9.1` or `9.2`):

    apt-get install postgresql-9.2-prefix
	psql ...
	=# create extension prefix;

### fedora, red hat, centos and scientific linux packages

Replace the PostgreSQL major version number here by the one you're running
(pick either `91` or `92`):

    yum install prefix92
	psql ...
	=# create extension prefix;

### from sources

Check `$PATH`, then

    make
    make install
    psql <connection string> -c 'CREATE EXTENSION prefix;'

The `make install` step might have to be done as `root`, and the psql one
has to be done as a PostgreSQL *superuser*.

### before 9.1 (consider an upgrade)

If you're running a very old PostgreSQL version such as `8.3`, `8.4` of
`9.0` you can still install this extension manually:

    psql ...
	=# \i /usr/share/postgresql/X.Y/extension/prefix--1.2.0.sql

You still have to edit this example to replace X.Y with your local
PostgreSQL version number, such as `8.4`.

## Uninstall

It's as easy as:

    DROP TYPE prefix_range CASCADE;

## Usage

### Table and index creation

    create table prefixes (
           prefix    prefix_range primary key,
           name      text not null,
           shortname text,
           status    char default 'S',

           check( status in ('S', 'R') )
    );
    comment on column prefixes.status is 'S:   - R: reserved';
    
    \copy prefixes from 'prefixes.fr.csv' with delimiter ; csv quote '"'

    create index idx_prefix on prefixes using gist(prefix);
 
### Simple tests

    dim=# select '123'::prefix_range @> '123456';
     ?column?
    ----------
     t
    (1 row)

Please note earlier versions of `prefix` didn't use any restriction
nor join selectivity estimator functions for the `@>` operator, so you
had to `set enable_seqscan to off` to use the index. That's no more
the case, the `@>` operator uses `contsel` and `contjoinsel` and the
planner is able to figure out by itself when to use the index or not.

If you don't understand previous mention, ignore it and use a more
recent version of `prefix` than `0.2`.

#### Forcing seqcan

    dim=# select * from ranges where prefix @> '0146640123';
     prefix |      name      | shortname | state
    --------+----------------+-----------+-------
     0146[] | FRANCE TELECOM | FRTE      | S
    (1 row)
    
    Time: 4,071 ms

    dim=# select * from ranges where prefix @> '0100091234';
      prefix  |    name    | shortname | state
    ----------+------------+-----------+-------
     010009[] | LONG PHONE | LGPH      | S
    (1 row)
    
    Time: 4,110 ms

#### Using the Index

    dim=# select * from ranges where prefix @> '0146640123';
     prefix |      name      | shortname | state
    --------+----------------+-----------+-------
     0146[] | FRANCE TELECOM | FRTE      | S
    (1 row)
    
    Time: 1,036 ms

    dim=# select * from ranges where prefix @> '0100091234';
      prefix  |    name    | shortname | state
    ----------+------------+-----------+-------
     010009[] | LONG PHONE | LGPH      | S
    (1 row)
    
    Time: 0,771 ms

As of version 1.0, `prefix_range` GiST index supports also queries using the
`<@`, `&&` and `=` operators (see below).

### creating prefix_range, cast to and from text

There's a *constructor* function:

    prefix=# select prefix_range('123');
     prefix_range 
    --------------
     123
    (1 row)

    prefix=# select prefix_range('123', '4', '5');
     prefix_range 
    --------------
     123[4-5]
    (1 row)

Casting from unknown literal or text is as easy as usual:

    prefix=# select '123'::prefix_range;
     prefix_range 
    --------------
     123
    (1 row)

    prefix=# select x, x::prefix_range from (values('123'), ('123[4-5]'), ('[2-3]')) as t(x);
        x     |    x     
    ----------+----------
     123      | 123
     123[4-5] | 123[4-5]
     [2-3]    | [2-3]
    (3 rows)

And two casts are provided:

    CREATE CAST (text as prefix_range) WITH FUNCTION prefix_range(text) AS IMPLICIT;
    CREATE CAST (prefix_range as text) WITH FUNCTION text(prefix_range);

Which means you can use text expression in a context where a `prefix_range`
is expected and it'll get implicit casting, but `prefix_range` to text has
to be asked explicitely, so that you don't get strange behavior.

### Provided operators

The prefix module is all about indexing prefix lookups, but in order to be
able to do this with some efficiency, it has to know a lot about prefix
ranges, such as basic comparing, containment, union, intersection and
overlapping.

The operators `<=`, `<`, `=`, `<>`, `>=` and `>` are read as usual, `@>` is
read *contains*, `<@` is read *is contained by*, `&&` is read *overlaps*,
and `|` is *union* and `&` is *intersect*.

    prefix=# select a, b,
      a <= b as "<=", a < b as "<", a = b as "=", a <> b as "<>", a >= b as ">=", a > b as ">",
      a @> b as "@>", a <@ b as "<@", a && b as "&&"
    from  (select a::prefix_range, b::prefix_range
             from (values('123', '123'),
                         ('123', '124'),
                         ('123', '123[4-5]'),
                         ('123[4-5]', '123[2-7]'),
                         ('123', '[2-3]')) as t(a, b)
          ) as x;
        a     |    b     | <= | < | = | <> | >= | > | @> | <@ | && 
    ----------+----------+----+---+---+----+----+---+----+----+----
     123      | 123      | t  | f | t | f  | t  | f | t  | t  | t
     123      | 124      | t  | t | f | t  | f  | f | f  | f  | f
     123      | 123[4-5] | t  | t | f | t  | f  | f | t  | f  | t
     123[4-5] | 123[2-7] | f  | f | f | t  | t  | t | f  | t  | t
     123      | [2-3]    | t  | t | f | t  | f  | f | f  | f  | f
    (5 rows)

    prefix=# select a, b, a | b as union, a & b as intersect 
      from  (select a::prefix_range, b::prefix_range
               from (values('123', '123'), 
                           ('123', '124'), 
                           ('123', '123[4-5]'),
                           ('123[4-5]', '123[2-7]'), 
                           ('123', '[2-3]')) as t(a, b)
            ) as x;
        a     |    b     |  union   | intersect 
    ----------+----------+----------+-----------
     123      | 123      | 123      | 123
     123      | 124      | 12[3-4]  | 
     123      | 123[4-5] | 123      | 123
     123[4-5] | 123[2-7] | 123[2-7] | 123[4-5]
     123      | [2-3]    | [1-3]    | 
    (5 rows)

## See also

This [TESTS.md](TESTS.md) page is more developper oriented material, but
still of interest.
