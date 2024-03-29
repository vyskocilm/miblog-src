---
title: "Implement Sql Database Driver in 100 Lines of Go"
date: 2019-02-18T22:59:18+02:00
image: "img/sql-flicker.jpg"
draft: false
---

Go [database/sql](https://golang.org/pkg/database/sql/) defines interfaces
for SQL databases. Actual driver must be implemented in own package. And
dependency injection is done as a part of import and build system. Lets go
deeper to see how it is actually implemented.

This exercise will result in a simple driver backed up by csv files. Who do
not want to `SELECT` from csv file?

Disclaimer: I wrote it in order to learn `database/sql` and to learn more about
design of APIs for go. I regularly use deprecated methods, which have advanced
variants in recent standard library. Consult documentation if you want to know
more.

I have had three goals in mind

1. Learning
2. Simplicity
3. 100 lines of code

## `database/sql` intro

Here is small snippet of Go program using Postgres
[https://github.com/lib/pq/](lib/pq) driver to talk to database. It use generic
methods of `database/sql` interfaces, so switching the driver and fixing SQL
dialects, we can talk to *any* database system without a need to change much
of the code.

```go
import (
    "database/sql"
    _ "github.com/lib/pq"
)

...
    db, err := sql.Open("postgres", dbURI)
    if err != nil {
        panic(err)
    }
    defer db.Close()
    _, err := db.Exec("CREATE TABLE IF NOT EXISTS TBL2(id BIGSERIAL)")
    if err != nil {
        panic(err)
    }
```

## Magic inside

There is a magic inside

```go
import (
    "fmt"
    "database/sql"
    _ "github.com/lib/pq"
    _ "github.com/go-sql-driver/mysql"
)

func main() {
    fmt.Printf("Drivers=%#v\n", sql.Drivers())
}
```

```sh
Drivers=[]string{"mysql", "postgres"}
```

[Driver](https://golang.org/pkg/database/sql/driver/#Driver) is magically
registered during module import. It sounds like a magic, isn't? And it turns
out it is actually very
[simple](https://github.com/lib/pq/blob/a8bb68eb453341d8c44a1333a579a13a8b4be428/conn.go#L49).
Function `init()` is package initializer, which can run arbitrary code before
all other code. In case of SQL driver, it contains the code to register itself.

```go
func init() {
	sql.Register("postgres", &Driver{})
}
```

Similary for [mysql](https://github.com/go-sql-driver/mysql/blob/6be42e0ff99645d7d9626d779001a46e39c5f280/driver.go#L168)

```go
func init() {
	sql.Register("mysql", &MySQLDriver{})
}
```

## Registering SQL Driver

With all those knowledge one can implement dummy SQL driver

```go
package main

import (
        "fmt"
        "database/sql"
)

type DummyDriver struct {
}

// implements sql.Driver interface
func (d *DummyDriver) Open(name string) (driver.Conn, error) {
    fmt.Printf("[dummy]: open(%s)", name)
    return nil, nil
}

func main() {

    // manually register dummy one
    sql.Register("dummy", &DummyDriver{})
    // List registered drivers, now with dummy
    fmt.Printf("Drivers=%#v\n", sql.Drivers())
}
```

And to test it has been registered ...
```
Drivers=[]string{"dummy"}
```

Of course it is really dummy driver as it does not offer ANY functionality.

## Connection

As we see, `Driver` interface returns
[`driver.Conn`](https://golang.org/pkg/database/sql/driver/#Conn), which is yet
another interface. This defines three methods, which can be skipped for the csv case.

First make `Open` real function, so let is open the file

```go
type DummyConn struct {
    name string
}

// implements sql.Driver interface
// Open - open file for reading, return the connection
func (d *DummyDriver) Open(name string) (driver.Conn, error) {
    file, err := os.Open(name)
    if err != nil {
        return nil, err
    }
    defer file.Close()
    return &DummyConn{name}, nil
}
```

Here one can see the beauty of Go error reporting. File operation errors
the same way as database server errors.

This is a bit uncommon for real SQL driver, however we do not have server to
connect to. So `Open` tries to open the file at least to return file access
errors as early as possible. All the *heavy lifting* is done below.

Here is bunch of methods not interesting for the use case ... all those returns
*not implemented error*.

```go
// implements driver.Conn interface
// Prepare - not implemented ...
func (c *DummyConn) Prepare(query string) (driver.Stmt, error) {
    return nil, fmt.Errorf("Prepare method not implemented")
}
// implements driver.Tx interface
// Rollback - not implemented ...
func (c *DummyConn) Rollback() error {
    return fmt.Errorf("Rollback method not implemented")
}
```

So for read-only file access the connection is really dumb method, there is
only one file path, which needs to be passed.

## Do the query

SQL queries are base of the system. Here is a constraint by supporting only
**one** query string. However I do prefer brevity over anything else, so here
we are. The `driver.Queryer` interface

```go
// Queryer interface
func (c *DummyConn) Query(query string, args []driver.Value) (driver.Rows, error) {
    if query != "SELECT * FROM csv" {
        return nil, fmt.Errorf("Only `SELECT * FROM csv` string is implemented!")
    }

    r := csv.NewReader(c.file)
    r.FieldsPerRecord = 0       // enforce the same number of columns
    columns, err := r.Read()  // assume first line gives you column names
    if err != nil {
        return nil, err
    }
    res := &results{r, columns}
    return res, nil
}
```

Again, nothing really complicated. One needs to handle errors and then return
correct structure with proper implemented interface methods.

Query methods returns an object, which allows read of the result. Real database
systems deal well with the read and write locks, concurrent access and more.
You have maybe heard about database `cursor`. In a case of read only csv file,
the situation is *way* more simpler.

As each `Query` can pass independent results, opened file is a part of
`results`. So each `results` will have independent file handle and file
position, which is an equivalent of database `cursor`.

## Read the results

One needs to implement next interface. `driver.Querier` returns object with
`driver.Rows` interface. Here is structure `results` used to read data through
`Query` and `Scan` methods. It provides enough to implement `Rows` interface
```go
type results struct {
    reader *csv.Reader
    columns []string
}
```

And implementation is fairly straightforward. We do assume that first line of
csv file contains names of columns. So this is initialized as a part of query. It simplify `Columns[]` method a lot!

``` go
// driver.Rows interface
func (r *results) Columns() []string {
    return r.columns
}
```

And to not leak file handles, close them once results are read

```go
func (r *results) Close() error {
    return r.file.Close()
}
```

And the most important method. Under the hood it reads data from csv file and
put them to internal buffer `dest[]`, which is then managed by `database/sql`
code. This is the way how data ends up in `Scan` method later on.

```
func (r *results) Next(dest []driver.Value) error {
    d, err := r.reader.Read()
    if err != nil {
        return err
    }
    for i := 0; i != len(r.columns); i++ {
        dest[i] = driver.Value(d[i])
    }
    return nil
}
```

## Running together

```go
    // manually register dummy driver
    sql.Register("dummy", &DummyDriver{})
    ...
    // prepare two return objects
	rows := make([]*sql.Rows, 2)
	for i := 0; i != 2; i++ {
		rows[i], err = db.Query("SELECT * FROM csv")
		if err != nil {
			panic(err)
		}
	}
    // read results
	for _, r := range rows {
		for r.Next() {
			var f1, f2, f3 string
			err := r.Scan(&f1, &f2, &f3)
			if err != nil {
				panic(err)
			}
			fmt.Printf("first_name=%s, last_name=%s, username=%s\n", f1, f2, f3)
		}
	}
```

And a proof, result of running program

```sh
$ go run main.go 
first_name=Rob, last_name=Pike, username=rob
first_name=Ken, last_name=Thompson, username=ken
first_name=Robert, last_name=Griesemer, username=gri
first_name=Rob, last_name=Pike, username=rob
first_name=Ken, last_name=Thompson, username=ken
first_name=Robert, last_name=Griesemer, username=gri
```

## Conclusion

Complete program is available at
[github.com/vyskocilm/gazpacho/dbmagic](https://github.com/vyskocilm/gazpacho/tree/master/dbmagic).
And as last method, `Next` ends at line 100. The goal to implement `dummy` sql driver in 100 lines of Go code was achieved.

After going through the article reader shall be able

1. To understand the *automagic* registration of SQL drivers
2. To understand the design of `database/sql` better
3. Hopefully enjoy the simplicity, but expressiveness of Go interfaces
4. Learn that design of interfaces is hard and `database/sql` offer V2
   interfaces in some cases.

[Reddit thread](https://www.reddit.com/r/golang/comments/as2qjr/implement_sql_database_driver_in_100_lines_of_go/)

Logo by travis_warren123@Flickr: [https://www.flickr.com/photos/travis_warren123/4229031035/]
