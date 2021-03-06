Rust-Postgres
=============
A native PostgreSQL driver for Rust.

Documentation is available at http://www.rust-ci.org/sfackler/rust-postgres/doc/postgres/.

[![Build Status](https://travis-ci.org/sfackler/rust-postgres.png?branch=master)](https://travis-ci.org/sfackler/rust-postgres)

Overview
========
Rust-Postgres is a pure-Rust frontend for the popular PostgreSQL database. It
exposes a high level interface in the vein of JDBC or Go's `database/sql`
package.
```rust
extern crate postgres;
extern crate time;

use time::Timespec;

use postgres::{PostgresConnection, NoSsl};
use postgres::types::ToSql;

struct Person {
    id: i32,
    name: String,
    time_created: Timespec,
    data: Option<Vec<u8>>
}

fn main() {
    let conn = PostgresConnection::connect("postgres://postgres@localhost",
                                           &NoSsl).unwrap();

    conn.execute("CREATE TABLE person (
                    id              SERIAL PRIMARY KEY,
                    name            VARCHAR NOT NULL,
                    time_created    TIMESTAMP NOT NULL,
                    data            BYTEA
                  )", []).unwrap();
    let me = Person {
        id: 0,
        name: "Steven".to_string(),
        time_created: time::get_time(),
        data: None
    };
    conn.execute("INSERT INTO person (name, time_created, data)
                    VALUES ($1, $2, $3)",
                 [&me.name, &me.time_created, &me.data]).unwrap();

    let stmt = conn.prepare("SELECT id, name, time_created, data FROM person")
            .unwrap();
    for row in stmt.query([]).unwrap() {
        let person = Person {
            id: row[0u],
            name: row[1u],
            time_created: row[2u],
            data: row[3u]
        };
        println!("Found person {}", person.name);
    }
}
```

Requirements
============

* **Rust** - Rust-Postgres is developed against the *master* branch of the Rust
    repository. It will most likely not build against the versioned releases on
    http://www.rust-lang.org.

* **PostgreSQL 7.4 or later** - Rust-Postgres speaks version 3 of the
    PostgreSQL protocol, which corresponds to versions 7.4 and later. If your
    version of Postgres was compiled in the last decade, you should be okay.

Usage
=====

Connecting
----------
Connect to a Postgres server using the standard URI format:
```rust
let conn = try!(PostgresConnection::connect("postgres://user:pass@host:port/database?arg1=val1&arg2=val2",
                                            &NoSsl));
```
`pass` may be omitted if not needed. `port` defaults to `5432` and `database`
defaults to the value of `user` if not specified. The driver supports `trust`,
`password`, and `md5` authentication.

Unix domain sockets can be used as well. The `host` portion of the URI should be
set to the absolute path to the directory containing the socket file. Since `/`
is a reserved character in URLs, the path should be URL encoded.
```rust
let conn = try!(PosgresConnection::connect("postgres://postgres@%2Frun%2Fpostgres", &NoSsl));
```
Paths which contain non-UTF8 characters can be handled in a different manner;
see the documentation for details.

Statement Preparation
---------------------
Prepared statements can have parameters, represented as `$n` where `n` is an
index into the parameter array starting from 1:
```rust
let stmt = try!(conn.prepare("SELECT * FROM foo WHERE bar = $1 AND baz = $2"));
```

Querying
--------
A prepared statement can be executed with the `query` and `execute` methods.
Both methods take an array of parameters to bind to the query represented as
`&ToSql` trait objects. `execute` returns the number of rows affected by the
query (or 0 if not applicable):
```rust
let stmt = try!(conn.prepare("UPDATE foo SET bar = $1 WHERE baz = $2"));
let updates = try!(stmt.execute([&1i32, &"biz"]));
println!("{} rows were updated", updates);
```
`query` returns an iterator over the rows returned from the database. The
fields in a row can be accessed either by their indices or their column names,
though access by index is more efficient. Unlike statement parameters, result
columns are zero-indexed.
```rust
let stmt = try!(conn.prepare("SELECT bar, baz FROM foo"));
for row in try!(stmt.query([])) {
    let bar: i32 = row[0u];
    let baz: String = row["baz"];
    println!("bar: {}, baz: {}", bar, baz);
}
```
In addition, `PostgresConnection` has a utility `execute` method which is useful
if a statement is only going to be executed once:
```rust
let updates = try!(conn.execute("UPDATE foo SET bar = $1 WHERE baz = $2",
                                [&1i32, &"biz"]));
println!("{} rows were updated", updates);
```

Transactions
------------
The `transaction` method will start a new transaction. It returns a
`PostgresTransaction` object which has the functionality of a
`PostgresConnection` as well as methods to control the result of the
transaction:
```rust
let trans = try!(conn.transaction());
try!(trans.execute(...));
let stmt = try!(trans.prepare(...));

if a_bad_thing_happened {
    trans.set_rollback();
}

if the_coast_is_clear {
    trans.set_commit();
}

drop(trans);
```
The transaction will be active until the `PostgresTransaction` object falls out
of scope. A transaction will commit by default. Nested transactions are
supported via savepoints.

Connection Pooling
------------------
A very basic fixed-size connection pool is provided in the `pool` module. A
single pool can be shared across tasks and `get_connection` will block until a
connection is available.
```rust
let pool = try!(PostgresConnectionPool::new("postgres://postgres@localhost",
                                            NoSsl, 5));

for _ in range(0, 10) {
    let pool = pool.clone();
    spawn(proc() {
        let conn = pool.get_connection();
        conn.query(...).unwrap();
    })
}
```

Type Correspondence
-------------------
Rust-Postgres enforces a strict correspondence between Rust types and Postgres
types. The driver currently supports the following conversions:

<table>
    <thead>
        <tr>
            <td>Rust Type</td>
            <td>Postgres Type</td>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>bool</td>
            <td>BOOL</td>
        </tr>
        <tr>
            <td>i8</td>
            <td>"char"</td>
        </tr>
        <tr>
            <td>i16</td>
            <td>SMALLINT, SMALLSERIAL</td>
        </tr>
        <tr>
            <td>i32</td>
            <td>INT, SERIAL</td>
        </tr>
        <tr>
            <td>i64</td>
            <td>BIGINT, BIGSERIAL</td>
        </tr>
        <tr>
            <td>f32</td>
            <td>REAL</td>
        </tr>
        <tr>
            <td>f64</td>
            <td>DOUBLE PRECISION</td>
        </tr>
        <tr>
            <td>str/String</td>
            <td>VARCHAR, CHAR(n), TEXT</td>
        </tr>
        <tr>
            <td>[u8]/Vec&lt;u8&gt;</td>
            <td>BYTEA</td>
        </tr>
        <tr>
            <td>serialize::json::Json</td>
            <td>JSON</td>
        </tr>
        <tr>
            <td>uuid::Uuid</td>
            <td>UUID</td>
        </tr>
        <tr>
            <td>time::Timespec</td>
            <td>TIMESTAMP, TIMESTAMP WITH TIME ZONE</td>
        </tr>
        <tr>
            <td>types::range::Range&lt;i32&gt;</td>
            <td>INT4RANGE</td>
        </tr>
        <tr>
            <td>types::range::Range&lt;i64&gt;</td>
            <td>INT8RANGE</td>
        </tr>
        <tr>
            <td>types::range::Range&lt;Timespec&gt;</td>
            <td>TSRANGE, TSTZRANGE</td>
        </tr>
        <tr>
            <td>types::array::ArrayBase&lt;Option&lt;bool&gt;&gt;</td>
            <td>BOOL[], BOOL[][], ...</td>
        </tr>
        <tr>
            <td>types::array::ArrayBase&lt;Option&lt;Vec&lt;u8&gt;&gt;&gt;</td>
            <td>BYTEA[], BYTEA[][], ...</td>
        </tr>
        <tr>
            <td>types::array::ArrayBase&lt;Option&lt;i8&gt;&gt;</td>
            <td>"char"[], "char"[][], ...</td>
        </tr>
        <tr>
            <td>types::array::ArrayBase&lt;Option&lt;i16&gt;&gt;</td>
            <td>INT2[], INT2[][], ...</td>
        </tr>
        <tr>
            <td>types::array::ArrayBase&lt;Option&lt;i32&gt;&gt;</td>
            <td>INT4[], INT4[][], ...</td>
        </tr>
        <tr>
            <td>types::array::ArrayBase&lt;Option&lt;String&gt;&gt;</td>
            <td>TEXT[], CHAR(n)[], VARCHAR[], TEXT[][], ...</td>
        </tr>
        <tr>
            <td>types::array::ArrayBase&lt;Option&lt;Json&gt;&gt;</td>
            <td>JSON[], JSON[][], ...</td>
        </tr>
        <tr>
            <td>types::array::ArrayBase&lt;Option&lt;i64&gt;&gt;</td>
            <td>INT8[], INT8[][], ...</td>
        </tr>
        <tr>
            <td>types::array::ArrayBase&lt;Option&lt;Timespec&gt;&gt;</td>
            <td>TIMESTAMP[], TIMESTAMPTZ[], TIMESTAMP[][], ...</td>
        </tr>
        <tr>
            <td>types::array::ArrayBase&lt;Option&lt;f32&gt;&gt;</td>
            <td>FLOAT4[], FLOAT4[][], ...</td>
        </tr>
        <tr>
            <td>types::array::ArrayBase&lt;Option&lt;f64&gt;&gt;</td>
            <td>FLOAT8[], FLOAT8[][], ...</td>
        </tr>
        <tr>
            <td>types::array::ArrayBase&lt;Option&lt;Uuid&gt;&gt;</td>
            <td>UUID[], UUID[][], ...</td>
        </tr>
        <tr>
            <td>types::array::ArrayBase&lt;Option&lt;Range&lt;i32&gt;&gt;&gt;</td>
            <td>INT4RANGE[], INT4RANGE[][], ...</td>
        </tr>
        <tr>
            <td>types::array::ArrayBase&lt;Option&lt;Range&lt;Timespec&gt;&gt;&gt;</td>
            <td>TSRANGE[], TSTZRANGE[], TSRANGE[][], ...</td>
        </tr>
        <tr>
            <td>types::array::ArrayBase&lt;Option&lt;Range&lt;i64&gt;&gt;&gt;</td>
            <td>INT8RANGE[], INT8RANGE[][], ...</td>
        </tr>
        <tr>
            <td>std::collections::HashMap&lt;String, Option&lt;String&gt;&gt;</td>
            <td>HSTORE</td>
        </tr>
    </tbody>
</table>

More conversions can be defined by implementing the `ToSql` and `FromSql`
traits.

Development
===========
Like Rust itself, Rust-Postgres is still in the early stages of development, so
don't be surprised if APIs change and things break. If something's not working
properly, file an issue or submit a pull request!
