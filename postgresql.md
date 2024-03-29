# PostgreSQL Cheat Sheet

My personal Cheatsheet for PostgreSQL

## Quick Commands 

### Creating a DB and Setting its Ownership

To create a database, you need to first ensure that the database's role exists first. Role and User are synonymous
in PostgreSQL. Once you create the `ROLE`, you can create the Database and set the `OWNER` as the `ROLE`.

**Create Role**

```psql
CREATE ROLE <USERNAME> ENCRYPTED PASSWORD '<PASSWORD>' LOGIN;
```

**Create Database**

```psql
CREATE DATABASE <DATABASE-NAME> OWNER "<USERNAME";
```

**Dropping a Database**

```psql
DROP DATABASE "<DATABASE-NAME>";
```

**Deleting a ROLE**

```psql
DROP ROLE "<USERNAME>";
```

## PSQL Commands

The following commands are for PSQL. You can access these and more by writing `\?`.

```sql
  \da[S]   [PATTERN]      list aggregates
  \db[+]   [PATTERN]      list tablespaces
  \dc[S+]  [PATTERN]      list conversions
  \dC[+]   [PATTERN]      list casts
  \dd[S]   [PATTERN]      show object descriptions not displayed elsewhere
  \ddp     [PATTERN]      list default privileges
  \dD[S+]  [PATTERN]      list domains
  \det[+]  [PATTERN]      list foreign tables
  \des[+]  [PATTERN]      list foreign servers
  \deu[+]  [PATTERN]      list user mappings
  \dew[+]  [PATTERN]      list foreign-data wrappers
  \df[antw][S+] [PATRN]  list [only agg/normal/trigger/window] functions
  \dF[+]   [PATTERN]      list text search configurations
  \dFd[+]  [PATTERN]      list text search dictionaries
  \dFp[+]  [PATTERN]      list text search parsers
  \dFt[+]  [PATTERN]      list text search templates
  \dg[+]   [PATTERN]      list roles
  \di[S+]  [PATTERN]      list indexes
  \dl                    list large objects, same as \lo_list
  \dL[S+]  [PATTERN]      list procedural languages
  \dm[S+]  [PATTERN]      list materialized views
  \dn[S+]  [PATTERN]      list schemas
  \do[S]   [PATTERN]      list operators
  \dO[S+]  [PATTERN]      list collations
  \dp      [PATTERN]      list table, view, and sequence access privileges
  \drds    [PATRN1 [PATRN2]] list per-database role settings
  \ds[S+]  [PATTERN]      list sequences
  \dt[S+]  [PATTERN]      list tables
  \dT[S+]  [PATTERN]      list data types
  \du[+]   [PATTERN]      list roles
  \dv[S+]  [PATTERN]      list views
  \dE[S+]  [PATTERN]      list foreign tables
  \dx[+]   [PATTERN]      list extensions
  \dy      [PATTERN]      list event triggers
  \l[+]    [PATTERN]      list databases
  \sf[+]   FUNCNAME        show a function's definition
  \z       [PATTERN]      same as \dp
```

Here are a couple of quick commands for common tasks that we do.

```sql
\l  - list all the databases in the server
\dt - shows all the tables in the selected database
\d+ <table-name> - describe the table, show it's columns, their types and their storage.
```

# User Access Control

## Create a User

Before creating a database, you need to create a user.

**NOTE:** you will need to be the `postgres` user to run this command. 

```
createuser <new-user> --pwprompt
```

*--pwprompt* will require you to set the password for the user you're creating.

You can access the database created for the user (db name is the same name as the username), or you can access the `postgres` database as such:

```
psql postgres -U <new-user>
```


## Connection to a Database

```
psql -h <host> -p <port> -u <database>
psql -h <host> -p <port> -U <username> -W <password> <database>
```

## Databases

### Create Database

```sql
CREATE DATABASE name
    [ [ WITH ] [ OWNER [=] user_name ]
               [ TEMPLATE [=] template ]
               [ ENCODING [=] encoding ]
               [ LC_COLLATE [=] lc_collate ]
               [ LC_CTYPE [=] lc_ctype ]
               [ TABLESPACE [=] tablespace_name ]
               [ ALLOW_CONNECTIONS [=] allowconn ]
               [ CONNECTION LIMIT [=] connlimit ]
               [ IS_TEMPLATE [=] istemplate ] ]
```

More Information: https://www.postgresql.org/docs/current/static/sql-createdatabase.html

You can create the database as the `postgres` user using the `createdb <db-name>` command. 

### Drop Database

```sql
DROP DATABASE [ IF EXISTS ] name
```

More Information: https://www.postgresql.org/docs/current/static/sql-dropdatabase.html

### Alter Database

```sql
ALTER DATABASE name [ [ WITH ] option [ ... ] ]

where option can be:

    CONNECTION LIMIT connlimit

ALTER DATABASE name RENAME TO new_name

ALTER DATABASE name OWNER TO new_owner

ALTER DATABASE name SET TABLESPACE new_tablespace

ALTER DATABASE name SET configuration_parameter { TO | = } { value | DEFAULT }
ALTER DATABASE name SET configuration_parameter FROM CURRENT
ALTER DATABASE name RESET configuration_parameter
ALTER DATABASE name RESET ALL
```

More Information: https://www.postgresql.org/docs/current/static/sql-alterdatabase.html

## Tables

### Create Table

There are a lot of details, so you're probably better off just going to the official docs to figure out how to do this.

https://www.postgresql.org/docs/current/static/sql-createtable.html

### Drop Table

```sql
DROP TABLE [ IF EXISTS ] name [, ...] [ CASCADE | RESTRICT ]
```

More Information: https://www.postgresql.org/docs/current/static/sql-droptable.html

### Alter Table

There are a lot of details, so you're probably better off just going to the official docs to figure out how to do this.

https://www.postgresql.org/docs/9.6/static/sql-altertable.html

## Basic CRUD Operations 

### Insert 

```sql
[ WITH [ RECURSIVE ] with_query [, ...] ]
INSERT INTO table_name [ AS alias ] [ ( column_name [, ...] ) ]
    { DEFAULT VALUES | VALUES ( { expression | DEFAULT } [, ...] ) [, ...] | query }
    [ ON CONFLICT [ conflict_target ] conflict_action ]
    [ RETURNING * | output_expression [ [ AS ] output_name ] [, ...] ]

where conflict_target can be one of:

    ( { index_column_name | ( index_expression ) } [ COLLATE collation ] [ opclass ] [, ...] ) [ WHERE index_predicate ]
    ON CONSTRAINT constraint_name

and conflict_action is one of:

    DO NOTHING
    DO UPDATE SET { column_name = { expression | DEFAULT } |
                    ( column_name [, ...] ) = ( { expression | DEFAULT } [, ...] ) |
                    ( column_name [, ...] ) = ( sub-SELECT )
                  } [, ...]
              [ WHERE condition ]
```

More Information: https://www.postgresql.org/docs/current/static/sql-insert.html

### Delete

```sql
[ WITH [ RECURSIVE ] with_query [, ...] ]
DELETE FROM [ ONLY ] table_name [ * ] [ [ AS ] alias ]
    [ USING using_list ]
    [ WHERE condition | WHERE CURRENT OF cursor_name ]
    [ RETURNING * | output_expression [ [ AS ] output_name ] [, ...] ]
```

More Information: https://www.postgresql.org/docs/current/static/sql-delete.html

### Update

```sql
[ WITH [ RECURSIVE ] with_query [, ...] ]
UPDATE [ ONLY ] table_name [ * ] [ [ AS ] alias ]
    SET { column_name = { expression | DEFAULT } |
          ( column_name [, ...] ) = ( { expression | DEFAULT } [, ...] ) |
          ( column_name [, ...] ) = ( sub-SELECT )
        } [, ...]
    [ FROM from_list ]
    [ WHERE condition | WHERE CURRENT OF cursor_name ]
    [ RETURNING * | output_expression [ [ AS ] output_name ] [, ...] ]
```

More Information: https://www.postgresql.org/docs/current/static/sql-update.html

### Querying

```sql
[WITH with_queries] SELECT select_list FROM table_expression [sort_specification]
```

More Information: https://www.postgresql.org/docs/current/static/queries.html

## Full Text Search

FUll Text Search allows put a set of text and search from it.

Let's do an initial setup of the database.

```sql
CREATE TABLE posts (
  id serial primary key,
  title VARCHAR(100),
  body TEXT
);


INSERT INTO posts VALUES
  (1, '13 Things that have changed since BJP Took Over Three Years Ago', 'Caste violence, unemployment and crimes against women have worsened. Poverty and GDP have improved.'),
  (2, 'The Hindu Mahasabha Rejected Statues Of Gandhi’s Assassin For Looking Too Chubby', 'You think people would not fat shame you just because you’re an inanimate object? Think again, guy.'),
  (3, 'U.S. Court Issues Summons To Indian PM Over Alleged Role In Deadly Gujarat Riots', 'Narendra Modi arrives in New York Friday for a five-day visit to the U.S.');
```

### Basic Introduction 

### Querying for Records

```sql
SELECT id
  FROM posts
  WHERE to_tsvector('english', title) @@ to_tsquery('english', 'BJP');

#  id 
# ----
#   1
# (1 row)

```

**Note:** For `to_tsvector(...)` and `to_tsquery(...)`, the first record is the language, and the second value is the field and value respectively.

You can also avoid this if `default_text_search_config(string)` is set (More information: https://www.postgresql.org/docs/current/static/runtime-config-client.html#GUC-DEFAULT-TEXT-SEARCH-CONFIG)

The last query can be done as:

```sql
SELECT id FROM posts WHERE to_tsvector(title) @@ to_tsquery('BJP');
```

### Creating Indexes

To speed up the search, we need to create a `GIN` index on the column we will be querying on. To do this, we must do the following:

```sql
CREATE INDEX posts_title_idx ON posts USING GIN (to_tsvector('english', title));
```

To do this on multiple columns, we can do this:

```sql
CREATE INDEX posts_fulltxt_idx ON posts USING GIN(to_tsvector('english', title || ' ' || body));
```

Now, we can query based  like such:

```sql
SELECT * FROM posts WHERE title @@ to_tsquery('BJP');
```

## Dealing with Unstructured Data

PostgreSQL can index JSON data, and query data from them. The following describes how to do this.

There are 3 types of data types that support Unstructured Data in PostgreSQL:

* **JSONB** - In most cases
* **JSON** - If you’re just processing logs, don’t often need to query, and use as more of an audit trail
* **hstore** - Can work fine for text based key-value looks, but in general JSONB can still work great here

Here are some great resources: https://www.citusdata.com/blog/2016/07/14/choosing-nosql-hstore-json-jsonb/

### JSONB

The B in JSONB is supposed to stand for 'better', but it really means that the data is Binary. Similar to BSON, JSONB removes spaces, and stores the data in a binary format into the database.

#### Creating Table with JSONB Column

```sql
CREATE TABLE product (id UUID, name VARCHAR(200), price NUMERIC(3, 2), attributes JSONB);
```

#### Inserting Data 

Following from Previous example, this is how we insert data into JSONB column

```sql
INSERT INTO product VALUES 
( uuid_generate_v4(), 'Carrots', 1.45, '{ "color": "orange", "size": "medium", "is_cleaned": false, "type": "vegetable" }'),
( uuid_generate_v4(), 'Apples', 2.95, '{ "color": "red", "size": "large", "is_cleaned": false, "type": "fruit" }'),
( uuid_generate_v4(), 'Orange', 1.95, '{ "color": "orange", "size": "small", "is_cleaned": true, "type": "fruit" }'),
( uuid_generate_v4(), 'Cucumber', 3.55, '{ "color": "green", "size": "medium", "is_cleaned": false, "type": "vegetable" }'),
( uuid_generate_v4(), 'Lettuce', 1.35, '{ "color": "green", "size": "large", "type": "vegetable" }'),
( uuid_generate_v4(), 'Telephone', 99.99, '{ "speed": "1GHz", "memory": "4Gb", "storage": "128Gb" }')
```

#### Creating Index



#### Querying Data

**Exact Match (Equality)**

Find records with exact JSON match

```sql
test-db=# SELECT * FROM product WHERE product.attributes = '{ "color": "green", "size": "large", "type": "vegetable" }';
                  id                  |  name   | price |                        attributes                        
--------------------------------------+---------+-------+----------------------------------------------------------
 b1487e38-e384-411e-b51a-591898642116 | Lettuce |  1.35 | {"size": "large", "type": "vegetable", "color": "green"}
(1 row)
```

**Containment**

Finds rows with JSON objects that contain the queried JSON object. 

Essentially, this equates to "find records where this query is a subset of the record."

```sql
test-db=# SELECT * FROM product WHERE product.attributes @> '{ "color": "green" }';
                  id                  |   name   | price |                                   attributes               
                    
--------------------------------------+----------+-------+------------------------------------------------------------
--------------------
 b9d83d67-325b-4bc9-bd05-f0867c36dc58 | Cucumber |  3.55 | {"size": "medium", "type": "vegetable", "color": "green", "
is_cleaned": false}
 b1487e38-e384-411e-b51a-591898642116 | Lettuce  |  1.35 | {"size": "large", "type": "vegetable", "color": "green"}
(2 rows)
```

**Check if a Key Exists**

Find rows where the JSON object has a given key:

```sql
test-db=# SELECT * FROM product WHERE attributes ? 'speed';
                  id                  |   name    | price |                       attributes                       
--------------------------------------+-----------+-------+--------------------------------------------------------
 a733b586-6ff4-49d4-9f3e-0bff9a42a4bc | Telephone |  3.99 | {"speed": "1GHz", "memory": "4Gb", "storage": "128Gb"}
(1 row)
```

**Check if ANY key in a list of Keys exists**

Find rows where the JSON object matches *any* key in the given set of keys:

```sql
test-db=# SELECT * FROM product WHERE attributes ?| array['speed', 'size'];
                  id                  |   name    | price |                                   attributes                                    
--------------------------------------+-----------+-------+---------------------------------------------------------------------------------
 6d4aa0e9-6ec0-4f73-9427-ecf3d3b164ec | Carrots   |  1.45 | {"size": "medium", "type": "vegetable", "color": "orange", "is_cleaned": false}
 de83c998-98fb-4bbb-bb4f-3fe2b00040a9 | Apples    |  2.95 | {"size": "large", "type": "fruit", "color": "red", "is_cleaned": false}
 b22d242f-1a0e-4c61-b219-aed9289df7bc | Orange    |  1.95 | {"size": "small", "type": "fruit", "color": "orange", "is_cleaned": true}
 b9d83d67-325b-4bc9-bd05-f0867c36dc58 | Cucumber  |  3.55 | {"size": "medium", "type": "vegetable", "color": "green", "is_cleaned": false}
 b1487e38-e384-411e-b51a-591898642116 | Lettuce   |  1.35 | {"size": "large", "type": "vegetable", "color": "green"}
 a733b586-6ff4-49d4-9f3e-0bff9a42a4bc | Telephone |  3.99 | {"speed": "1GHz", "memory": "4Gb", "storage": "128Gb"}
(6 rows)

```

**Check if ALL key in a list of Keys exists**

Find rows where the JSON object matches *all* key in the given set of keys:

```sql
test-db=# SELECT * FROM product WHERE attributes ?& array['speed', 'size'];
 id | name | price | attributes 
----+------+-------+------------
(0 rows)

test-db=# SELECT * FROM product WHERE attributes ?& array['speed', 'storage'];
                  id                  |   name    | price |                       attributes                       
--------------------------------------+-----------+-------+--------------------------------------------------------
 a733b586-6ff4-49d4-9f3e-0bff9a42a4bc | Telephone |  3.99 | {"speed": "1GHz", "memory": "4Gb", "storage": "128Gb"}
(1 row)
```

You can get more information here: http://schinckel.net/2014/05/25/querying-json-in-postgres/