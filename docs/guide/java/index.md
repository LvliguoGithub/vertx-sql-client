# Reactive Postgres Client

The Reactive Postgres Client is a client for Postgres with a straightforward API focusing on
scalability and low overhead.

The client is reactive and non blocking, allowing to handle many database connections with a single thread.

* Event driven
* Lightweight
* Built-in connection pooling
* Prepared queries caching
* Publish / subscribe using Postgres `NOTIFY/LISTEN`
* Batch and cursor support
* Row streaming
* Command pipeling
* RxJava 1 and RxJava 2 support
* Direct memory to object without unnecessary copies
* Java 8 Date and Time support
* SSL/TLS support
* HTTP/1.x CONNECT, SOCKS4a or SOCKS5 proxy support

## Usage

To use the Reactive Postgres Client add the following dependency to the _dependencies_ section of your build descriptor:

* Maven (in your `pom.xml`):

[source,xml,subs#"+attributes"]
```
<dependency>
 <groupId>io.reactiverse</groupId>
 <artifactId>reactive-pg-client</artifactId>
 <version>0.7.0</version>
</dependency>
```

* Gradle (in your `build.gradle` file):

```groovy
dependencies {
 compile 'io.reactiverse:reactive-pg-client:0.7.0'
}
```

## Getting started

Here is the simplest way to connect, query and disconnect

```java
PgPoolOptions options = new PgPoolOptions()
  .setPort(5432)
  .setHost("the-host")
  .setDatabase("the-db")
  .setUsername("user")
  .setPassword("secret")
  .setMaxSize(5);

// Create the client pool
PgPool client = PgClient.pool(options);

// A simple query
client.query("SELECT * FROM users WHERE id='julien'", ar -> {
  if (ar.succeeded()) {
    PgResult<Row> result = ar.result();
    System.out.println("Got " + result.size() + " results ");
  } else {
    System.out.println("Failure: " + ar.cause().getMessage());
  }

  // Now close the pool
  client.close();
});
```

## Connecting to Postgres

Most of the time you will use a pool to connect to Postgres:

```java
PgPoolOptions options = new PgPoolOptions()
  .setPort(5432)
  .setHost("the-host")
  .setDatabase("the-db")
  .setUsername("user")
  .setPassword("secret")
  .setMaxSize(5);

// Create the pooled client
PgPool client = PgClient.pool(options);
```

The pooled client uses a connection pool and any operation will borrow a connection from the pool
to execute the operation and release it to the pool.

If you are running with Vert.x you can pass it your Vertx instance:

```java
PgPoolOptions options = new PgPoolOptions()
  .setPort(5432)
  .setHost("the-host")
  .setDatabase("the-db")
  .setUsername("user")
  .setPassword("secret")
  .setMaxSize(5);

// Create the pooled client
PgPool client = PgClient.pool(vertx, options);
```

You need to release the pool when you don't need it anymore:

```java
pool.close();
```

When you need to execute several operations on the same connection, you need to use a client
[`connection`](../../apidocs/io/reactiverse/pgclient/PgConnection.html).

You can easily get one from the pool:

```java
PgPoolOptions options = new PgPoolOptions()
  .setPort(5432)
  .setHost("the-host")
  .setDatabase("the-db")
  .setUsername("user")
  .setPassword("secret")
  .setMaxSize(5);

// Create the pooled client
PgPool client = PgClient.pool(vertx, options);

// Get a connection from the pool
client.getConnection(ar1 -> {

  if (ar1.succeeded()) {

    System.out.println("Connected");

    // Obtain our connection
    PgConnection conn = ar1.result();

    // All operations execute on the same connection
    conn.query("SELECT * FROM users WHERE id='julien'", ar2 -> {
      if (ar2.succeeded()) {
        conn.query("SELECT * FROM users WHERE id='emad'", ar3 -> {
          // Release the connection to the pool
          conn.close();
        });
      } else {
        // Release the connection to the pool
        conn.close();
      }
    });
  } else {
    System.out.println("Could not connect: " + ar1.cause().getMessage());
  }
});
```

Once you are done with the connection you must close it to release it to the pool, so it can be reused.

## Running queries

When you don't need a transaction or run single queries, you can run queries directly on the pool; the pool
will use one of its connection to run the query and return the result to you.

Here is how to run simple queries:

```java
client.query("SELECT * FROM users WHERE id='julien'", ar -> {
  if (ar.succeeded()) {
    PgResult<Row> result = ar.result();
    System.out.println("Got " + result.size() + " results ");
  } else {
    System.out.println("Failure: " + ar.cause().getMessage());
  }
});
```

You can do the same with prepared queries.

The SQL string can refer to parameters by position, using `$1`, `$2`, etc…​

```java
client.preparedQuery("SELECT * FROM users WHERE id=$1", Tuple.of("julien"),  ar -> {
  if (ar.succeeded()) {
    PgResult<Row> result = ar.result();
    System.out.println("Got " + result.size() + " results ");
  } else {
    System.out.println("Failure: " + ar.cause().getMessage());
  }
});
```

Query methods provides an asynchronous [`PgResult`](../../apidocs/io/reactiverse/pgclient/PgResult.html) instance that works for _SELECT_ queries

```java
client.preparedQuery("SELECT first_name, last_name FROM users", ar -> {
  if (ar.succeeded()) {
    PgResult<Row> result = ar.result();
    for (Row row : result) {
      System.out.println("User " + row.getString(0) + " " + row.getString(1));
    }
  } else {
    System.out.println("Failure: " + ar.cause().getMessage());
  }
});
```

or _UPDATE_/_INSERT_ queries:

```java
client.preparedQuery("INSERT INTO users (first_name, last_name) VALUES ($1, $2)", Tuple.of("Julien", "Viet"),  ar -> {
  if (ar.succeeded()) {
    PgResult<Row> result = ar.result();
    System.out.println(result.updatedCount());
  } else {
    System.out.println("Failure: " + ar.cause().getMessage());
  }
});
```

The [`Row`](../../apidocs/io/reactiverse/pgclient/Row.html) gives you access to your data by index

```java
System.out.println("User " + row.getString(0) + " " + row.getString(1));
```

or by name

```java
System.out.println("User " + row.getString("first_name") + " " + row.getString("last_name"));
```

You can access a wide variety of of types

```java
String firstName = row.getString("first_name");
Boolean male = row.getBoolean("male");
Integer age = row.getInteger("age");
```

You can execute prepared batch

```java
List<Tuple> batch = new ArrayList<>();
batch.add(Tuple.of("julien", "Julien Viet"));
batch.add(Tuple.of("emad", "Emad Alblueshi"));

// Execute the prepared batch
client.preparedBatch("INSERT INTO USERS (id, name) VALUES ($1, $2)", batch, res -> {
  if (res.succeeded()) {

    // Process results
    PgResult<Row> results = res.result();
  } else {
    System.out.println("Batch failed " + res.cause());
  }
});
```

You can cache prepared queries:

```java
options.setCachePreparedStatements(true);

PgPool client = PgClient.pool(vertx, options);
```

## Using connections

When you need to execute sequential queries (without a transaction), you can create a new connection
or borrow one from the pool:

```java
pool.getConnection(ar1 -> {
  if (ar1.succeeded()) {
    PgConnection connection = ar1.result();

    connection.query("SELECT * FROM users WHERE id='julien'", ar2 -> {
      if (ar1.succeeded()) {
        connection.query("SELECT * FROM users WHERE id='paulo'", ar3 -> {
          // Do something with results and return the connection to the pool
          connection.close();
        });
      } else {
        // Return the connection to the pool
        connection.close();
      }
    });
  }
});
```

Prepared queries can be created:

```java
connection.prepare("SELECT * FROM users WHERE first_name LIKE $1", ar1 -> {
  if (ar1.succeeded()) {
    PgPreparedQuery pq = ar1.result();
    pq.execute(Tuple.of("julien"), ar2 -> {
      if (ar2.succeeded()) {
        // All rows
        PgResult<Row> result = ar2.result();
      }
    });
  }
});
```

NOTE: prepared query caching depends on the [`setCachePreparedStatements`](../../apidocs/io/reactiverse/pgclient/PgConnectOptions.html#setCachePreparedStatements-boolean-) and
does not depend on whether you are creating prepared queries or use [`direct prepared queries`](../../apidocs/io/reactiverse/pgclient/PgClient.html#preparedQuery-java.lang.String-io.vertx.core.Handler-)

By default prepared query executions fetch all results, you can use a [`PgCursor`](../../apidocs/io/reactiverse/pgclient/PgCursor.html) to control the amount of rows you want to read:

```java
connection.prepare("SELECT * FROM users WHERE first_name LIKE $1", ar1 -> {
  if (ar1.succeeded()) {
    PgPreparedQuery pq = ar1.result();

    // Create a cursor
    PgCursor cursor = pq.cursor(Tuple.of("julien"));

    // Read 50 rows
    cursor.read(50, ar2 -> {
      if (ar2.succeeded()) {
        PgResult<Row> result = ar2.result();

        // Check for more ?
        if (cursor.hasMore()) {

          // Read the next 50
          cursor.read(50, ar3 -> {
            // More results, and so on...
          });
        } else {
          // No more results
        }
      }
    });
  }
});
```

Cursors shall be closed when they are released prematurely:

```java
connection.prepare("SELECT * FROM users WHERE first_name LIKE $1", ar1 -> {
  if (ar1.succeeded()) {
    PgPreparedQuery pq = ar1.result();
    PgCursor cursor = pq.cursor(Tuple.of("julien"));
    cursor.read(50, ar2 -> {
      if (ar2.succeeded()) {
        // Close the cursor
        cursor.close();
      }
    });
  }
});
```

A stream API is also available for cursors, which can be more convenient, specially with the Rxified version.

```java
connection.prepare("SELECT * FROM users WHERE first_name LIKE $1", ar1 -> {
  if (ar1.succeeded()) {
    PgPreparedQuery pq = ar1.result();

    // Fetch 50 rows at a time
    PgStream<Row> stream = pq.createStream(50, Tuple.of("julien"));

    // Use the stream
    stream.exceptionHandler(err -> {
      System.out.println("Error: " + err.getMessage());
    });
    stream.endHandler(v -> {
      System.out.println("End of stream");
    });
    stream.handler(row -> {
      System.out.println("User: " + row.getString("last_name"));
    });
  }
});
```

The stream read the rows by batch of `50` and stream them, when the rows have been passed to the handler,
a new batch of `50` is read and so on.

The stream can be resumed or paused, the loaded rows will remain in memory until they are delivered and the cursor
will stop iterating.

[`PgPreparedQuery`](../../apidocs/io/reactiverse/pgclient/PgPreparedQuery.html)can perform efficient batching:

```java
connection.prepare("INSERT INTO USERS (id, name) VALUES ($1, $2)", ar1 -> {
  if (ar1.succeeded()) {
    PgPreparedQuery prepared = ar1.result();

    // Create a query : bind parameters
    List<Tuple> batch = new ArrayList();

    // Add commands to the createBatch
    batch.add(Tuple.of("julien", "Julien Viet"));
    batch.add(Tuple.of("emad", "Emad Alblueshi"));

    prepared.batch(batch, res -> {
      if (res.succeeded()) {

        // Process results
        PgResult<Row> results = res.result();
      } else {
        System.out.println("Batch failed " + res.cause());
      }
    });
  }
});
```

## Using transactions

You can execute transaction using SQL `BEGIN`/`COMMIT`/`ROLLBACK`, if you do so you must use
a [`PgConnection`](../../apidocs/io/reactiverse/pgclient/PgConnection.html) and manage it yourself.

Or you can use the transaction API of [`PgConnection`](../../apidocs/io/reactiverse/pgclient/PgConnection.html):

```java
pool.getConnection(res -> {
  if (res.succeeded()) {

    // Transaction must use a connection
    PgConnection conn = res.result();

    // Begin the transaction
    PgTransaction tx = conn.begin();

    // Various statements
    conn.query("INSERT INTO Users (first_name,last_name) VALUES ('Julien','Viet')", ar -> {});
    conn.query("INSERT INTO Users (first_name,last_name) VALUES ('Emad','Alblueshi')", ar -> {});

    // Commit the transaction
    tx.commit(ar -> {
      if (ar.succeeded()) {
        System.out.println("Transaction succeeded");
      } else {
        System.out.println("Transaction failed " + ar.cause().getMessage());
      }
    });
  }
});
```

When Postgres reports the current transaction is failed (e.g the infamous _current transaction is aborted, commands ignored until
end of transaction block_), the transaction is rollbacked and the [`abortHandler`](../../apidocs/io/reactiverse/pgclient/PgTransaction.html#abortHandler-io.vertx.core.Handler-)
is called:

```java
pool.getConnection(res -> {
  if (res.succeeded()) {

    // Transaction must use a connection
    PgConnection conn = res.result();

    // Begin the transaction
    PgTransaction tx = conn
      .begin()
      .abortHandler(v -> {
      System.out.println("Transaction failed => rollbacked");
    });

    conn.query("INSERT INTO Users (first_name,last_name) VALUES ('Julien','Viet')", ar -> {
      // Works fine of course
    });
    conn.query("INSERT INTO Users (first_name,last_name) VALUES ('Julien','Viet')", ar -> {
      // Fails and triggers transaction aborts
    });

    // Attempt to commit the transaction
    tx.commit(ar -> {
      // But transaction abortion fails it
    });
  }
});
```

## Postgres type mapping

### Handling JSON

The [`Json`](../../apidocs/io/reactiverse/pgclient/Json.html) Java type is used to represent the Postgres `JSON` and `JSONB` type.

The main reason of this type is handling `null` JSON values.

```java
Tuple tuple = Tuple.of(
  Json.create(null),
  Json.create(new JsonObject().put("foo", "bar")),
  Json.create(3));

//
tuple = Tuple.tuple()
  .addJson(Json.create(null))
  .addJson(Json.create(new JsonObject().put("foo", "bar")))
  .addJson(Json.create(3));

// JSON object (and arrays) can also be added directly
tuple = Tuple.tuple()
  .addJson(Json.create(null))
  .addJsonObject(new JsonObject().put("foo", "bar"))
  .addJson(Json.create(3));

// Retrieving json
Object value = tuple.getJson(0).value(); // Expect null

//
value = tuple.getJson(1).value(); // Expect JSON object
value = tuple.getJsonObject(1); // Works too

//
value = tuple.getJson(3).value(); // Expect 3
```

### Handling NUMERIC

The [`Numeric`](../../apidocs/io/reactiverse/pgclient/Numeric.html) Java type is used to represent the Postgres `NUMERIC` type.

```java
Numeric numeric = row.getNumeric("value");
if (numeric.isNaN()) {
  // Handle NaN
} else {
  BigDecimal value = numeric.bigDecimalValue();
}
```

## Pub/sub

Postgres supports pub/sub communication channels.

You can set a [`notificationHandler`](../../apidocs/io/reactiverse/pgclient/PgConnection.html#notificationHandler-io.vertx.core.Handler-) to receive
Postgres notifications:

```java
connection.notificationHandler(notification -> {
  System.out.println("Received " + notification.getPayload() + " on channel " + notification.getChannel());
});

connection.query("LISTEN some-channel", ar -> {
  System.out.println("Subscribed to channel");
});
```

The [`PgSubscriber`](../../apidocs/io/reactiverse/pgclient/pubsub/PgSubscriber.html) is a channel manager managing a single connection that
provides per channel subscription:

```java
PgSubscriber subscriber = PgSubscriber.subscriber(vertx, new PgConnectOptions()
  .setPort(5432)
  .setHost("the-host")
  .setDatabase("the-db")
  .setUsername("user")
  .setPassword("secret")
);

// You can set the channel before connect
subscriber.channel("channel1").handler(payload -> {
  System.out.println("Received " + payload);
});

subscriber.connect(ar -> {
  if (ar.succeeded()) {

    // Or you can set the channel after connect
    subscriber.channel("channel2").handler(payload -> {
      System.out.println("Received " + payload);
    });
  }
});
```

You can provide a reconnect policy as a function that takes the number of `retries` as argument and returns an `amountOfTime`
value:

* when `amountOfTime < 0`: the subscriber is closed and there is no retry
* when `amountOfTime ## 0`: the subscriber retries to connect immediately
* when `amountOfTime > 0`: the subscriber retries after `amountOfTime` milliseconds

```java
PgSubscriber subscriber = PgSubscriber.subscriber(vertx, new PgConnectOptions()
  .setPort(5432)
  .setHost("the-host")
  .setDatabase("the-db")
  .setUsername("user")
  .setPassword("secret")
);

// Reconnect at most 10 times after 100 ms each
subscriber.reconnectPolicy(retries -> {
  if (retries < 10) {
    return 100L;
  } else {
    return -1L;
  }
});
```

The default policy is to not reconnect.

## Using SSL/TLS

To configure the client to use SSL connection, you can configure the [`PgConnectOptions`](../../apidocs/io/reactiverse/pgclient/PgConnectOptions.html)
like a Vert.x `NetClient`}.

```java
PgConnectOptions options = new PgConnectOptions()
  .setPort(5432)
  .setHost("the-host")
  .setDatabase("the-db")
  .setUsername("user")
  .setPassword("secret")
  .setSsl(true)
  .setPemTrustOptions(new PemTrustOptions().addCertPath("/path/to/cert.pem"));

PgClient.connect(vertx, options, res -> {
  if (res.succeeded()) {
    // Connected with SSL
  } else {
    System.out.println("Could not connect " + res.cause());
  }
});
```

More information can be found in the [Vert.x documentation](http://vertx.io/docs/vertx-core/java/#ssl).

## Using a proxy

You can also configure the client to use an HTTP/1.x CONNECT, SOCKS4a or SOCKS5 proxy.

More information can be found in the http://vertx.io/docs/vertx-core/java/#_using_a_proxy_for_client_connections[Vert.x documentation].