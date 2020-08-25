---
layout: post
title: "Build a Chat app with SocketIO and RethinkDB"
tags: rethinkdb socketio heroku
author: Sascha
excerpt_separator: <!--more-->
---

A lot of tutorials can be found that teach you how to build a chat app with Socket.io.
However, have you ever wondered how to best persist those chat messages?

Enter RethinkDB, a realtime schema-less database. You can store and handle documents
easily, just like in MongoDB, but it has reactivity built into it. That means you can
subscribe to queries and get notified when data changes, making it the perfect choice when it
comes to storing chat messages.

In this article you will learn how to create a simple chat app with Socket.io and
persist the messages in RethinkDB. To show the usefulness of a reactive database,
we will also add a simple bot that reacts when you address it.

<!--more-->

> You can try the [running app](https://rethink-chat-socketio.herokuapp.com/), or
> check out the [code repository](https://github.com/mostlytyped/rethink-chat-socketio).

## Application setup

We will build a Node.js app, so you need to have `node` and `npm` installed.
If you want to deploy your app to Heroku, you will also need a
[Heroku account](https://signup.heroku.com/), as well having their
[CLI](https://devcenter.heroku.com/articles/heroku-cli) installed.
To run your app locally, you need to
[install and run a RethinkDB instance](https://rethinkdb.com/).

To create the application, run the following in a terminal.

```sh
$ mkdir rethink-chat && cd rethink-chat
$ npm init -y
$ npm install rethinkdb express morgan http socket.io lorem-ipsum
```

This will initialize a Node.js app and install all required dependencies.

### Prepare a Heroku app

In order to deploy the application to Heroku we need to create a
Heroku app:

```sh
$ git init
$ heroku create
```

We will also need a RethinkDB instance to store and subscribe to the chat messages
sent between users. You can do this via the [RethinkDB Cloud add-on](/) as follows:

```sh
$ heroku addons:create rethinkdb
```

> The RethinkDB Cloud add-on is currently in alpha. [Request an invite for your
> Heroku account email](/).

## Building the server

To begin, let us set up the Node.js server. Create an `index.js` file and add
the following server skeleton.
We use an Express.js server to handle http traffic and Socket.io
to handle WebSocket connections with clients.

```js
// index.js

// Setup Express and Socket.io servers
var express = require("express");
const app = express();
var http = require("http").createServer(app);
var io = require("socket.io")(http);

// Logging middleware
var morgan = require("morgan");
app.use(morgan("combined"));

// Serve frontend
app.use(express.static("public"));

// Keep track of room subscriptions in RethinkDB
const watchedRooms = {};

// Lazy RethinkDB connection
// ...

// Route to access a room
// ...

// Socket.io (listen for new messages in any room)
// ...

// HTTP server (start listening)
const listenPort = process.env.PORT || "3000";
http.listen(listenPort, () => {
  console.log("listening on *:" + listenPort);
});
```

This skeleton serves a static frontend from the `public` folder. We will create the frontend
code later. In addition our server needs to do three things:

1. Handle connections to the RethinkDB database
2. Create an Express.js route that will give a user access to the chat room
3. Configure the Socket.io server to listen to incoming chat messages

### RethinkDB connection

We manage our RethinkDB connection lazily, i.e., we only create the (re-)connection
when it is actually needed. The connection parameters are parsed from
environment variables, or the defaults are used.

```js
// index.js
// ...

// Lazy RethinkDB connection
var r = require("rethinkdb");
let rdbConn = null;
const rdbConnect = async function () {
  try {
    const conn = await r.connect({
      host: process.env.RETHINKDB_HOST || "localhost",
      port: process.env.RETHINKDB_PORT || 28015,
      username: process.env.RETHINKDB_USERNAME || "admin",
      password: process.env.RETHINKDB_PASSWORD || "",
      db: process.env.RETHINKDB_NAME || "test",
    });

    // Handle close
    conn.on("close", function (e) {
      console.log("RDB connection closed: ", e);
      rdbConn = null;
    });

    console.log("Connected to RethinkDB");
    rdbConn = conn;
    return conn;
  } catch (err) {
    throw err;
  }
};
const getRethinkDB = async function () {
  if (rdbConn != null) {
    return rdbConn;
  }
  return await rdbConnect();
};
```

On Heroku, the RethinkDB Cloud add-on will set the environment variables. For a locally running
instance of RethinkDB, the defaults should work.

### Route to access room

As mentioned earlier, the frontend is static. We do however need a route to access
a chat room. The route will return the message history of a given room, as well as a
WebSocket handle to access it.

```js
// index.js
// ...

// Route to access a room
app.get("/chats/:room", async (req, res) => {
  const conn = await getRethinkDB();

  const room = req.params.room;
  let query = r.table("chats").filter({ roomId: room });

  // Subscribe to new messages
  if (!watchedRooms[room]) {
    query.changes().run(conn, (err, cursor) => {
      if (err) throw err;
      cursor.each((err, row) => {
        if (row.new_val) {
          // Got a new message, send it via Socket.io
          io.emit(room, row.new_val);
        }
      });
    });
    watchedRooms[room] = true;
  }

  // Return message history & Socket.io handle to get new messages
  let orderedQuery = query.orderBy(r.desc("ts"));
  orderedQuery.run(conn, (err, cursor) => {
    if (err) throw err;
    cursor.toArray((err, result) => {
      if (err) throw err;
      res.json({
        data: result,
        handle: room,
      });
    });
  });
});
```

This is where the RethinkDB magic happens.
The first time this route is called for a particular room (when the first person joins),
we subscribe to a RethinkDB query to get notified whenever a new chat message is
available. We send new chat messages via Socket.io to any clients listening for the
room's handle.

### Listen for new messages

The last puzzle piece of the server is to listen and save all incoming chat
messages. Whenever a message comes in via the `chats` handle of the Socket.io
connection, we save it to the `chats` table in RethinkDB.

```js
// index.js
// ...

// Socket.io (listen for new messages in any room)
io.on("connection", (socket) => {
  socket.on("chats", async (msg) => {
    const conn = await getRethinkDB();
    r.table("chats")
      .insert(Object.assign(msg, { ts: Date.now() }))
      .run(conn, function (err, res) {
        if (err) throw err;
      });
  });
});
```

Saving a value in the `chats` table will trigger the subscription we added above, causing
the message to be sent to all clients listening to this room, including the sender of the
message.

## Frontend

For our frontend we will use an embedded Vue.js app. This makes the frontend simple,
but gives us access to all of Vue's awesome features. The frontend consists of a layout file
as well as JavaScript and CSS assets.

- The layout file only serves as a mount point for the Vue app in addition to importing
  the dependencies.

  ```html
  <!-- public/index.html -->

  <!DOCTYPE html>
  <html lang="en">
    <head>
      <meta charset="UTF-8" />
      <meta name="viewport" content="width=device-width, initial-scale=1.0" />
      <title>RethinkDB Chat with SocketIO</title>
      <link href="/css/main.css" rel="stylesheet" />
    </head>

    <body>
      <div id="app">
        <router-view></router-view>
      </div>
      <script src="/socket.io/socket.io.js"></script>
      <script src="https://cdn.jsdelivr.net/npm/vue/dist/vue.js"></script>
      <script src="https://unpkg.com/vue-router/dist/vue-router.js"></script>
      <script src="/js/app.js" language="javascript"></script>
    </body>
  </html>
  ```

- The CSS asset contains the styling of the frontend. It is long, not very interesting, and can be found
  [here](https://github.com/mostlytyped/rethink-chat-socketio/blob/master/public/css/main.css).
- The JavaScript asset `app.js` contains the actual Vue app.

  ```js
  // public/js/app.js

  // Create random username
  let username = Math.random().toString(36).substring(2, 8);

  // Setup Socket.io
  var socket = io();

  // Main view
  // ...

  // Room view, holds a chat room component
  // ...

  // Chat room component
  // ...

  // Setup routes
  const router = new VueRouter({
    routes: [
      { path: "/", component: MainView },
      { path: "/:roomId", name: "room", component: RoomView },
    ],
  });

  // Mount Vue app
  var app = new Vue({
    router,
  }).$mount("#app");
  ```

  The Vue app contains two routes. The `/` path points to the main view
  and the `/:roomId` path points to the room view.

### Main view

The main view lets you choose a username (default is a random string) and
allows you to join a room with a given name.

```js
// public/js/app.js
// ...

// Main view
const MainView = Vue.component("main-view", {
  data() {
    return {
      room: "lobby",
      user: username,
    };
  },
  methods: {
    gotoRoom() {
      username = this.user;
      this.$router.push({ name: "room", params: { roomId: this.room } });
    },
  },
  template: `
<div class="main">
    <form class="main" v-on:submit.prevent="gotoRoom">
    <label>Username: <input v-model="user" type="text" /></label>
    <label>Room: <input v-model="room" type="text" /></label>
    <button>Join</button>
    </form>
</div>
    `,
});
```

Whenever you join a room, the Vue router will load the chat room view.

### Chat room

The chat room, a room view containing a chat room component. makes a request to the
Express route to join the given room when it is created. It also registers
a Socket.io handler that listens for incoming chat messages and adds them
to the list of messages.

The chat room allows the user to type and send a message which will then be
sent to the server via the WebSocket handled by Socket.io.

```js
// public/js/app.js
// ...

// Room view, holds a chat room component
const RoomView = Vue.component("room-view", {
  template: `<chat-room :roomId="$route.params.roomId"/>`,
});

// Chat room component
const ChatRoom = Vue.component("chat-room", {
  props: ["roomId"],
  data() {
    return {
      chats: [],
      message: "",
      username: username,
      handle: null,
    };
  },
  async created() {
    const url = new URL(document.location.protocol + "//" + document.location.host + "/chats/" + this.roomId);
    const chatsResp = await fetch(url);
    const { data, handle } = await chatsResp.json();
    this.chats = data;
    this.handle = handle;
    socket.on(this.handle, (msg) => {
      this.chats.unshift(msg);
    });
  },
  beforeDestroy() {
    socket.off(this.handle);
  },
  methods: {
    sendMessage() {
      socket.emit("chats", {
        msg: this.message,
        user: this.username,
        roomId: this.roomId,
      });
      this.message = "";
    },
  },
  template: `{% raw %}
<div class="chatroom">
    <ul id="chatlog">
        <li v-for="chat in chats">
            <span class="timestamp">
                {{ new Date(chat.ts).toLocaleString(undefined, {dateStyle: 'short', timeStyle: 'short'}) }}
            </span>
            <span class="user">{{ chat.user }}:</span>
            <span class="msg">{{ chat.msg }}</span>
        </li>
    </ul>
    <label id="username">Username:
        {{ username }}
    </label>
    <form v-on:submit.prevent="sendMessage">
        <input v-model="message" autocomplete="off" />
        <button>Send</button>
    </form>
</div>{% endraw %}
    `,
});
```

Now we have a working server and frontend. The last thing we need is to make sure
the `chats` table actually exists in the RethinkDB database when we run the app.

## Database migration

The app does not work without a `chats` table. We thus need a database migration that adds
the table.

```js
// migrate.js

var r = require("rethinkdb");

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

    r.tableList().run(conn, (err, cursor) => {
      if (err) throw err;
      cursor.toArray((err, tables) => {
        if (err) throw err;

        // Check if table exists
        if (!tables.includes("chats")) {
          // Table missing --> create
          console.log("Creating chats table");
          r.tableCreate("chats").run(conn, (err, _) => {
            if (err) throw err;
            console.log("Creating chats table -- done");
            conn.close();
          });
        } else {
          // Table exists --> exit
          conn.close();
        }
      });
    });
  },
);
```

This migration checks if the `chats` table exists, and if it is missing, it creates it.

## A simple chat bot

As we saw, one of RethinkDBs great features is the baked in reactivity that allows us to subscribe
to queries. This feature also comes in handy when creating a simple chat bot. The
bot simply needs to subscribe to changes in the `chats` table and react to them whenever
appropriate.

Our Lorem bot will reply with a random section of Lorem Ipsum whenever prompted with
`@lorem`. The bot subscribes to the `chats` table and scans the beginning of the message.
If it starts with `@lorem`, it will reply with a message in the same room.

```js
// lorem-bot.js

const LoremIpsum = require("lorem-ipsum").LoremIpsum;
const lorem = new LoremIpsum({
  sentencesPerParagraph: {
    max: 8,
    min: 4,
  },
  wordsPerSentence: {
    max: 16,
    min: 4,
  },
});

// Run Lorem bot
const runBot = function (conn) {
  console.log("Lorem bot started");
  r.table("chats")
    .changes()
    .run(conn, (err, cursor) => {
      if (err) throw err;
      cursor.each((err, row) => {
        const msg = row.new_val.msg.trim().split(/\s+/);
        // Is the message directed at me?
        if (msg[0] === "@lorem") {
          let num = 10;
          if (msg.length >= 1) {
            num = parseInt(msg[1]) || num;
          }
          r.table("chats")
            .insert({
              user: "lorem",
              msg: lorem.generateWords(num),
              roomId: row.new_val.roomId,
              ts: Date.now(),
            })
            .run(conn, function (err, res) {
              if (err) throw err;
            });
        }
      });
    });
};

// Connect to RethinkDB
const r = require("rethinkdb");
const rdbConnect = async function () {
  try {
    const conn = await r.connect({
      host: process.env.RETHINKDB_HOST || "localhost",
      port: process.env.RETHINKDB_PORT || 28015,
      username: process.env.RETHINKDB_USERNAME || "admin",
      password: process.env.RETHINKDB_PASSWORD || "",
      db: process.env.RETHINKDB_NAME || "test",
    });

    // Handle close
    conn.on("close", function (e) {
      console.log("RDB connection closed: ", e);
      setTimeout(rdbConnect, 10 * 1000); // reconnect in 10s
    });

    // Start the lorem bot
    runBot(conn);
  } catch (err) {
    throw err;
  }
};
rdbConnect();
```

## Deploy the application to Heroku

To deploy our working application and bot to Heroku we need
to create a `Procfile`. This file basically tells Heroku
what processes to run.

```
// Procfile

release: node migrate.js
web: node index.js
lorem-bot: node lorem-bot.js
```

The `release` and `web` processes are recognized by Heroku as
the command to run upon release and the main web app respectively. The
`lorem-bot` process is just a worker process that could have any
name.

Deploy the app to Heroku with

```sh
$ echo "node_modules/" > .gitignore
$ git add .
$ git commit -m 'Working rethink-chat app'
$ git push heroku master
```

> You will need to manually enable the `lorem-bot` process in your
> Heroku app. You can do so on the Resources tab.

## Conclusion

In less than 15 minutes we managed to create and deploy a chat application
with a simple bot. This shows the power and ease of use of RethinkDB.
The ability to subscribe to queries makes it easy to build a reactive
app and a natural fit to interact with Socket.io. Further, Heroku makes deployment a breeze, and
with the RethinkDB Cloud add-on you will never have to do the tedious work of
managing a database server yourself.
