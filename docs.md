---
layout: post
title: "Documentation"
---

[RethinkDB Cloud](https://elements.heroku.com/addons/rethinkdb) provides
a hosted RethinkDB database instance. It is fully managed and elastic, i.e.,
you can add or remove storage instantly and without the loss of data.

## Getting started

Start by installing the add-on:

```sh
$ heroku addons:create rethinkdb
```

Once the add-on has been added you’ll notice new variables in
`heroku config`:

```sh
$ heroku config
...
RETHINKDB_HOST:           bf1036d9-c2ba-4ae3-ad58-8d14a50117b3.rdb.lf.memcachier.com
RETHINKDB_NAME:           bf1036d9-c2ba-4ae3-ad58-8d14a50117b3
RETHINKDB_PASSWORD:       078c1d4cd2dec203d2db05f788fe5d3a09393bb9
RETHINKDB_PORT:           28015
RETHINKDB_USERNAME:       bf1036d9-c2ba-4ae3-ad58-8d14a50117b3
...
```

These variables are required when configuring your client to connect to your RethinkDB instance.

## Node.js

Install the driver with npm:

```sh
$ npm install rethinkdb
```

You can use the drivers from Node.js like this:

```js
r = require("rethinkdb");
r.connect(
  {
    host: process.env.RETHINKDB_HOST || "localhost",
    port: process.env.RETHINKDB_PORT || 28015,
    username: process.env.RETHINKDB_USERNAME || "admin",
    password: process.env.RETHINKDB_PASSWORD || "",
    db: process.env.RETHINKDB_NAME || "test",
  },
  function (err, conn) {
    if (err) throw err;
    r.tableCreate("tv_shows").run(conn, function (err, res) {
      if (err) throw err;
      console.log(res);
      r.table("tv_shows")
        .insert({ name: "Star Trek TNG" })
        .run(conn, function (err, res) {
          if (err) throw err;
          console.log(res);
        });
    });
  },
);
```

> Check out the official [ten-minute guide](https://rethinkdb.com/docs/guide/javascript/) and
> the [ReQL command reference](https://rethinkdb.com/api/javascript/) to learn how to use RethinkDB.

## Python

Install the driver with pip:

```sh
$ pip install rethinkdb
```

You can use the drivers from Python like this:

```python
from rethinkdb import r
r.connect(host=os.getenv('RETHINKDB_HOST', 'localhost'),
          port=os.getenv('RETHINKDB_PORT', 28015),
          user=os.getenv('RETHINKDB_USERNAME', 'admin'),
          password=os.getenv('RETHINKDB_PASSWORD', ''),
          db=os.getenv('RETHINKDB_NAME', 'test')).repl()
r.table_create('tv_shows').run()
r.table('tv_shows').insert({ 'name': 'Star Trek TNG' }).run()
```

> Check out the official [ten-minute guide](https://rethinkdb.com/docs/guide/python/) and
> the [ReQL command reference](https://rethinkdb.com/api/python/) to learn how to use RethinkDB.

## Ruby

Install the driver with gem:

```sh
$ gem install rethinkdb
```

You can use the drivers from Ruby like this:

```ruby
require 'rubygems'
require 'rethinkdb'
include RethinkDB::Shortcuts
r.connect(:host => ENV['RETHINKDB_HOST'] || 'localhost',
          :port => ENV['RETHINKDB_PORT'] || 28015,
          :user => ENV['RETHINKDB_USERNAME'] || 'admin',
          :password => ENV['RETHINKDB_PASSWORD'] || '',
          :db => ENV['RETHINKDB_NAME'] || 'test', ).repl
r.table_create('tv_shows').run
r.table('tv_shows').insert({ 'name'=>'Star Trek TNG' }).run
```

> Check out the official [ten-minute guide](https://rethinkdb.com/docs/guide/ruby/) and
> the [ReQL command reference](https://rethinkdb.com/api/ruby/) to learn how to use RethinkDB.

## Java

### Installation

#### Maven

If you’re using Maven, add this to your `pom.xml` file:

```xml
<dependencies>
  <dependency>
    <groupId>com.rethinkdb</groupId>
    <artifactId>rethinkdb-driver</artifactId>
    <version>2.4.1</version>
  </dependency>
</dependencies>
```

#### Gradle

If you’re using Gradle, modify your `build.gradle` file:

```java
dependencies {
    compile group: 'com.rethinkdb', name: 'rethinkdb-driver', version: '2.4.1'
}
```

#### Ant

If you’re using Ant, add the following to your `build.xml`:

```xml
<artifact:dependencies pathId="dependency.classpath">
  <dependency groupId="com.rethinkdb" artifactId="rethinkdb-driver" version="2.4.1" />
</artifact:dependencies>
```

#### SBT

If you’re using SBT, add the following to your `build.sbt`:

```java
libraryDependencies += "com.rethinkdb" % "rethinkdb-driver" % "2.4.1"
```

### Usage

You can use the drivers from Java like this:

```java
import com.rethinkdb.RethinkDB;
import com.rethinkdb.gen.exc.ReqlError;
import com.rethinkdb.gen.exc.ReqlQueryLogicError;
import com.rethinkdb.model.MapObject;
import com.rethinkdb.net.Connection;

public static final RethinkDB r = RethinkDB.r;

Map<String,String> envs = System.getenv();

Connection conn = r.connection()
    .hostname(envs.getOrDefault("RETHINKDB_HOST","localhost"))
    .port(Integer.parseInt(envs.getOrDefault("RETHINKDB_PORT","28015")))
    .user(envs.getOrDefault("RETHINKDB_USERNAME","admin"), envs.getOrDefault("RETHINKDB_PASSWORD",""))
    .dbname(envs.getOrDefault("RETHINKDB_NAME","test"))
    .connect();

r.tableCreate("tv_shows").run(conn);
r.table("tv_shows").insert(r.hashMap("name", "Star Trek TNG")).run(conn);
```

> Check out the official [ten-minute guide](https://rethinkdb.com/docs/guide/java/) and
> the [ReQL command reference](https://rethinkdb.com/api/java/) to learn how to use RethinkDB.

## Other languages

Find drivers and documentation for other languages on the official [RethinkDB page](https://rethinkdb.com/docs/install-drivers/).
