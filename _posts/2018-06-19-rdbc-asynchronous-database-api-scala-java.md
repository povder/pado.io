---
layout: post
title: "Introducing rdbc: asynchronous database access API for Scala and Java"
excerpt: "An introduction to the new asynchronous Scala and Java relational database access API and its Netty-based PostgreSQL driver with simple Akka Streams and Play Framework integration examples. Community feedback is greatly appreciated!"
description: "An introduction to the new asynchronous Scala and Java relational database access API and its Netty-based PostgreSQL driver with simple Akka Streams and Play Framework integration examples."
categories:
comments: true
last_updated:
---

## A generic asynchronous database access API

For quite some time it's been bothering me that many excellent relational database
access libraries for Scala use JDBC underneath. Even those with an asynchronous
interface, like [Slick](http://slick.lightbend.com/), must exercise a pool of threads
that are blocked on I/O actions.<sup>[[1]](#references)</sup>

I realized that the community could benefit from the generic, low&#8209;level,
asynchronous database access API. The great libraries out there, like aforementioned
Slick, could target this API in the way they currently target JDBC. Because I&nbsp;thought
it was an&nbsp;interesting thing to do I&nbsp;went ahead and started designing the API
along with implementing a&nbsp;PostgreSQL driver for it. I also decided to do 
a&nbsp;Java API at the same time. 

It's been like 2 years since I&nbsp;started slowly progressing with this and 
I feel now it's time to announce it for the feedback. It's called **rdbc**. 
The sources are available [on GitHub](https://github.com/rdbc-io).
There's also a&nbsp;fairly<sup>[a](#notes)</sup> complete [documentation](https://rdbc.io).

## Goals and non-goals of the project

After a&nbsp;couple of weeks of work goals of the API emerged:

1. **Provide vendor&#8209;neutral access to most commonly used database features.**

    The API is meant to be vendor neutral in a&nbsp;sense that if clients stick
    to using only standard SQL features no vendor&#8209;specific code should be needed
    and database backends can be switched with no client code changes.

2. **Be asynchronous and reactive.**

    All methods that can potentially perform I/O actions don't block the executing
    thread so the API fits well into non&#8209;blocking application design. rdbc
    allows building applications according to the [Reactive Manifesto](http://www.reactivemanifesto.org/).
   
3. **Provide a&nbsp;foundation for higher&#8209;level APIs.**

    rdbc is a&nbsp;rather low&#8209;level API enabling clients to use plain SQL queries and get results back.
    While it can be used directly it's also meant to provide a&nbsp;foundation for 
    higher&#8209;level APIs like functional or object relational mapping libraries.

I also decided on one non&#8209;goal which is:

1. **Will not offer full type&#8209;safety.**

    rdbc works on a&nbsp;SQL level, meaning that requests made to the database
    are strings. There is no additional layer that would ensure type&#8209;safety
    when working with SQL. The API is also meant to be dynamic and allow type
    converters to be registered at runtime. This approach sacrifices some
    type&#8209;safety but at the same time makes it possible to implement a&nbsp;wider range
    of higher&#8209;level APIs on top of rdbc.

## A basic example

Just to give you a general idea, below are simple code snippets that demonstrate
the basic usage of the API. The code selects a&nbsp;`name` column from `users`
table and returns the result as a collection of strings. 

Scala:
```scala
import io.rdbc.sapi.ConnectionFactory

val db: ConnectionFactory = /*...*/

val names: Future[Vector[String]] = db.withConnection { conn =>
  conn.statement("select name from users where age = :age")
      .bind("age" -> 30)
      .executeForSet() // Future[ResultSet]
      .map { resultSet =>
         resultSet.map(row => row.str("name").toVector
      }
}
```

Java:

```java
import io.rdbc.japi.ConnectionFactory;

ConnectionFactory db = /*...*/

CompletionStage<List<String>> names = db.withConnection(conn ->
    conn.statement("select name from users where age = :age")
        .arg("age", 30)
        .bind()
        .executeForSet() // CompletionStage<ResultSet>
        .thenApply(resultSet ->
            resultSet.getRows()
                     .stream()
                     .map(row -> row.getStr("name"))
                     .collect(Collectors.toList())
        )
);
```


## API highlights

There are a&nbsp;couple of things I&nbsp;particularly like about the API. I&nbsp;described them below.

### Reactive Streams interfaces

The API provides a set of methods returning Scala `Future`s or `CompletionStage`s
in Java variant, but these interfaces aren't the only way to access data from the database.
Rows returned by SQL queries can be streamed with back&#8209;pressure from the database
using Reactive Streams `Publisher` interface which enables easy interop with Akka Streams, 
Play framework and other technologies that support streaming.

Here's a Play framework action implementation returning a stream of JSON documents
as an HTTP chunked transfer encoding.

Scala:
```scala
import akka.stream.scaladsl.{Sink, Source}
import org.reactivestreams.Publisher
import io.rdbc.sapi.SqlInterpolator._
import io.rdbc.sapi.{ConnectionFactory, Row}

/*...*/

val db: ConnectionFactory = /*...*/

def stream = Action.async { _ =>
  db.connection().map { conn =>
    val pub: Publisher[Row] = conn
      .statement(sql"SELECT i, t, v FROM rdbc_demo ORDER BY i, t, v")
      .stream()

    val source = Source.fromPublisher(pub).map { row =>
      Json.toJson(
        Record(row.intOpt("i"), row.instantOpt("t"), row.strOpt("v"))
      )
    }.alsoTo(Sink.onComplete(_ => conn.release()))
    Ok.chunked(source)
  }
}
```

Java:
```java
import akka.NotUsed;
import akka.stream.javadsl.Sink;
import akka.stream.javadsl.Source;
import akka.util.ByteString;
import org.reactivestreams.Publisher;
import io.rdbc.japi.ConnectionFactory;
import io.rdbc.japi.Row;

/*...*/

private final ConnectionFactory db = /*...*/

public CompletionStage<Result> stream() {
    return db.getConnection().thenApply(conn -> {
        Publisher<Row> pub = conn.statement("SELECT i, t, v FROM rdbc_demo ORDER BY i, t, v")
                .noArgs().stream();

        Source<ByteString, NotUsed> recordSource = Source.fromPublisher(pub)
                .map(row -> Json.toJson(
                        new Record(
                                row.getIntOpt("i"),
                                row.getInstantOpt("t"),
                                row.getStrOpt("v")
                        )).toString()
                )
                .map(ByteString::fromString)
                .alsoTo(Sink.onComplete(ignore -> conn.release()));

        return Results.ok().chunked(recordSource).as(Http.MimeTypes.JSON);
    });
}
```

### String interpolation

The API features a&nbsp;[string interpolator](http://docs.api.rdbc.io/scala/statements/#string-interpolator)
for Scala that allows to easily build the  SQL statements without sacrificing
safety against the SQL injection attacks. Statements can be created like this 
(note the `sql` prefix before the SQL statement):

```scala
import io.rdbc.sapi.SqlInterpolator._
import io.rdbc.sapi.Connection

def insertPerson(conn: Connection, person: Person): Future[Unit] = {
    conn.statement(sql"insert into people (name, age) values (${person.name}, ${person.age})")
        .execute()
}
```

In the above snippet, `${person.name}` and `${person.age}` will be replaced by
[prepared statement](https://en.wikipedia.org/wiki/Prepared_statement)
parameters and their values will be passed to the database without doing string
concatenation (as in JDBC's `java.sql.PreparedStatement`).

Unfortunately, that's not possible to do in Java version. Below is an equivalent Java code.

```java
import io.rdbc.japi.Connection;

public CompletionStage<Void> insertPerson(Connection conn, Person person) {
    return conn.statement("insert into people (name, age) values (:name, :age)")
                .arg("name", person.getName())
                .arg("age", person.getAge())
                .bind()
                .execute();
}
```

### No nulls

Another thing that I&nbsp;like is that the API doesn't force you to use `null`.
You can't get SQL NULL value from the database as JVM `null`. If you expect 
some column to hold NULL values you must explicitly call a&nbsp;method 
that returns a&nbsp;Scala's `Option` or Java's `Optional`:

Scala:

```scala
val str: Option[String] = row.strOpt("column")
```

Java:

```java
Optional<String> str = row.strOpt("column");
```

## Asynchronous and non&#8209;blocking PostgreSQL driver

Introducing a&nbsp;new API without any working implementations wouldn't make much sense.
I decided to implement [a PostgreSQL driver](https://github.com/rdbc-io/rdbc-pgsql)
for it. It's a&nbsp;netty&#8209;based library that has been tested to work with PostgreSQL&nbsp;9.6.x
and 10.x but in fact, it should work with any version since 7.4.

Check out complete examples that use real PostgreSQL database 
(including a&nbsp;Play framework app) in the [rdbc&#8209;examples](https://github.com/rdbc-io/rdbc-examples)
repository. They are documented in the repository's README.

## Call for feedback

Before releasing the first stable version I&nbsp;need some feedback
from the community, especially on the API design itself. I&nbsp;accept any constructive
feedback: please
message me on [Twitter](https://twitter.com/messages/compose?recipient_id=4705264092)
or [Gitter](https://gitter.im/rdbc-io/rdbc),
[drop me an e&#8209;mail](mailto:povder@gmail.com)
or comment on this article.

Also, if you think that my effort makes sense please star my repos 
[on GitHub](https://github.com/rdbc-io), thanks!

## Links

* [Code examples](https://github.com/rdbc-io/rdbc-examples)
* [rdbc homepage](https://rdbc.io)
* [rdbc GitHub page](https://github.com/rdbc-io)
* [rdbc Twitter profile](https://twitter.com/rdbc_io)

## Notes

* a. Documentation for the Java version is not yet available.

## References

1. Slick contributors. *Slick documentation*, [Database thread pool chapter](http://slick.lightbend.com/doc/3.2.3/database.html#database-thread-pool)
2. Stephen Colebourne, Roger Riggs, Michael Nascimento Santos. [*JSR 310: Date and Time API*](https://jcp.org/en/jsr/detail?id=310), Section 2.5
