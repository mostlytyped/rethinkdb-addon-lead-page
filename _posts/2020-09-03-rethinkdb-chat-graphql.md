---
layout: post
#title: "Build a realtime Chat app with RethinkDB and GraphQL"
title: "Bring RethinkDB's realtime magic to the frontend with GraphQL"
tags: rethinkdb graphql heroku
author: Sascha
excerpt_separator: <!--more-->
---

<!-- Bring RethinkDB's reactivity to the frontend with GraphQL subscriptions -->

In a [recent post]({% post_url 2020-08-17-rethinkdb-chat-socketio %}) we explored
how RethinkDB's built-in reactivity is a perfect fit to write a chat app with Socket.io.
In this article you will learn how to use GraphQL subscriptions instead, to access
RethinkDB's reactive nature in the frontend.

RethinkDB is a realtime document database. It is easy to use and schema-less, just
like in MongoDB. In addition, you can subscribe to queries and get notified when
data changes, making it the perfect choice for realtime applications.

<!--more-->

You can also try the [running app](https://rethink-chat-graphql.herokuapp.com/), or
check out the [code repository](https://github.com/mostlytyped/rethink-chat-graphql).
{: .alert-info}

## Application setup

We will build a Node.js app, so you need to have `node` and `npm` installed.
If you want to deploy your app to Heroku, you will also need a
[Heroku account](https://signup.heroku.com/), as well having their
[CLI](https://devcenter.heroku.com/articles/heroku-cli) installed.
To run your app locally, you need to
[install and run a RethinkDB instance](https://rethinkdb.com/).

We will use a simple Node.js server and a Vue.js frontend. Since
the frontend needs to be build we will create a Vue app with the
[Vue CLI](https://cli.vuejs.org/):

```sh
$ vue create -d rethink-chat
$ cd rethink-chat
```

This will create a Node project, create a Vue.js skeleton,
and initialize a git repository.

### Prepare a Heroku app

In order to deploy the application to Heroku we need to create a
Heroku app:

```sh
$ heroku create
```

We will also need a RethinkDB instance to store and subscribe to the chat messages
sent between users. You can do this via the [RethinkDB Cloud add-on](/) as follows:

```sh
$ heroku addons:create rethinkdb
```

The RethinkDB Cloud add-on is currently in alpha. [Request an invite for your
Heroku account email](/).
{: .alert-warning}

## Building the server

We will create our server in the `server` directory. So to start, lets create
the directory and install the required dependencies:

```sh
$ mkdir server
$ npm install rethinkdb apollo-server-express graphql morgan lorem-ipsum
```

Now, let us set up the Node.js server. Create an `index.js` file and add
the following server skeleton.
We use an Express.js server to serve the frontend and the Apollo GraphQL server
to access and subscribe to chat messages.

```js
// server/index.js

// Setup Express server
const express = require("express");
const app = express();
const http = require("http").createServer(app);

// Logging middleware
var morgan = require("morgan");
app.use(morgan("combined"));

// Serve frontend
app.use(express.static("dist"));

// Lazy RethinkDB connection
// ...

// Setup Apollo (GraphQL) server
// ...

// HTTP server (start listening)
const listenPort = process.env.PORT || "3000";
http.listen(listenPort, () => {
  console.log("listening on *:" + listenPort);
});
```

This skeleton serves a static frontend from the `dist` folder. This is where the
compiled Vue.js app is located which we will create later. In addition our server
needs to do three things:

1. Handle connections to the RethinkDB database
2. Setup the Apollo server
3. Create a GraphQL schema including type definitions and resolvers

### RethinkDB connection

We manage our RethinkDB connection lazily, i.e., we only create the (re-)connection
when it is actually needed. The connection parameters are parsed from
environment variables, or the defaults are used.

```js
// server/index.js
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

### Apollo GraphQL server setup

As mentioned earlier, the frontend is static. We do however need to access the data in
a chat room. This will be handled by Apollo, the most used GraphQL server.

```js
// server/index.js
// ...

// Setup Apollo (GraphQL) server
const { ApolloServer } = require("apollo-server-express");
const { typeDefs, resolvers } = require("./schema.js");
const graphqlServer = new ApolloServer({
  typeDefs,
  resolvers,
  context: async (arg) => {
    const conn = await getRethinkDB();
    return {
      conn: conn,
    };
  },
});
graphqlServer.applyMiddleware({ app });
graphqlServer.installSubscriptionHandlers(http);
```

This will create an Apollo server using the type definitions and resolves defined
in our schema file (next section). We also connect to RethinkDB and pass the connection
to our GraphQL context so it can be used in any incoming request.

### Create a GraphQL schema

The main logic of the server resides in defining the GraphQL types and implement
their resolvers. We need to be able to preform three different actions, namely

- Query chat messages in a room
- Send a chat message to a room
- Subscribe to new chat messages in a room

First, we create the GraphQL types. This consists of a `Chat` message type and the
three mentioned actions, namely the `chats` query, the `sendChat` mutation, and
the `chatAdded` subscription.

```js
// server/schema.js

// GraphQL type definitions
const { gql } = require("apollo-server-express");
exports.typeDefs = gql`
  type Chat {
    user: String
    msg: String
    roomId: String
    ts: Float
  }

  type Query {
    chats(room: String!): [Chat]
  }

  type Mutation {
    sendChat(user: String!, message: String!, room: String!): Chat
  }

  type Subscription {
    chatAdded(room: String!): Chat
  }
`;

// GraphQL resolvers
// ...
```

Second, we need to resolve these actions, i.e., implement the code that they invoke. The query
and the mutation are fairly straight-forward and are implemented as a simple RethinkDB
query. The subscription however, requires an async iterator. This is basically spell
to turn the RethinkDB magic into GraphQL subscription magic. In more earthly terms,
the async iterator wraps the RethinkDB change feed so we can subscribe to it via
GraphQL.

```js
// server/schema.js

// GraphQL type definitions
// ...

// GraphQL resolvers
const r = require("rethinkdb");
exports.resolvers = {
  Subscription: {
    chatAdded: {
      async subscribe(parent, args, context, info) {
        return new RethinkIterator(
          r.table("chats").filter({ roomId: args.room }),
          context.conn,
        );
      },
    },
  },
  Mutation: {
    async sendChat(root, args, context) {
      const chatMsg = {
        user: args.user,
        roomId: args.room,
        msg: args.message,
        ts: Date.now(),
      };
      await r.table("chats").insert(chatMsg).run(context.conn);
      return chatMsg;
    },
  },
  Query: {
    async chats(parent, args, context, info) {
      const cursor = await r
        .table("chats")
        .filter({ roomId: args.room })
        .orderBy(r.desc("ts"))
        .run(context.conn);
      return await cursor.toArray();
    },
  },
};

// Async iterator to access the RethinkDB change feed
const { $$asyncIterator } = require("iterall");
class RethinkIterator {
  constructor(query, conn) {
    this.cursor = query.changes().run(conn);
  }

  async next() {
    const val = await (await this.cursor).next();
    return { value: { chatAdded: val.new_val }, done: false };
  }

  async return() {
    await (await this.cursor).close();
    return { value: undefined, done: true };
  }

  async throw(error) {
    return Promise.reject(error);
  }

  [$$asyncIterator]() {
    return this;
  }
}
```

With the server set up, lets move to the frontend

## Creating the frontend

We already created the Vue.js app skeleton we will use for the frontend.
However, since our server implements a standard GraphQL backend,
you might as well use React or any other frontend framework that supports GraphQL.

Our frontend will use two views, one for the home page and one for the chat room as well
as a router to navigate between the two. For this lets add a router to the Vue skeleton
and install all required dependencies:

```sh
$ vue add router
$ npm install apollo-client apollo-link-http apollo-link-ws apollo-cache-inmemory vue-apollo
$ npm install sass sass-loader --save-dev
```

Our Vue app is located in the `src` folder and will be structured as follows:
the entry point is in `main.js` and gets the GraphQL client configuration
from `graphql.js`. Our main file also mounts `App.vue` which displays views
selected by the router in `router/index.js`. Our app contains
two views, `views/Home.vue` and `views/ChatRoom.vue`.

```
src
├── main.js
├── graphql.js
├── App.vue
├── router
│   └── index.js
└── views
    ├── Home.vue
    └── ChatRoom.vue
```

### Main app and router

In a first step, let us modify the main app, home view, and router files that where
initialized in the skeleton Vue app. In `main.js` we import the Apollo GraphQL client
we will define further down and add it to our Vue app. In addition we will also create
a random chat username for the user.

```js
// src/main.js

import Vue from "vue";
import App from "./App.vue";
import router from "./router";
import apolloProvider from "./graphql";

Vue.config.productionTip = false;

// Initialize random username
window.username = Math.random().toString(36).substring(2, 8);

// Create and mount Vue app
new Vue({
  router,
  apolloProvider,
  render: (h) => h(App),
}).$mount("#app");
```

Our `App.vue` is even simpler than the skeleton, it just shows the router view and has
some styling.

```vue
<!-- src/App.vue -->

<template>
  <div id="app">
    <router-view />
  </div>
</template>

<script>
export default {
  name: "App",
};
</script>

<style lang="scss">
// See styles at https://github.com/mostlytyped/rethink-chat-graphql/blob/master/src/App.vue
</style>
```

In our `router.js` we basically replace the "About" route with our "Room" route.

```js
// src/router/index.js

import Vue from "vue";
import VueRouter from "vue-router";
import Home from "@/views/Home";
import ChatRoom from "@/views/ChatRoom";

Vue.use(VueRouter);

const routes = [
  { path: "/", name: "Home", component: Home },
  { path: "/:roomId", name: "Room", component: ChatRoom },
];

const router = new VueRouter({
  routes,
});

export default router;
```

In the home view we remove the `HelloWorld` component and add a form that allows us to join
a room.

```vue
<!-- src/views/Home.vue -->

<template>
  <div class="main">
    <form v-on:submit.prevent="gotoRoom">
      <label>
        Username:
        <input v-model="user" type="text" />
      </label>
      <label>
        Room:
        <input v-model="room" type="text" />
      </label>
      <button>Join</button>
    </form>
  </div>
</template>

<script>
export default {
  name: "Main",
  data() {
    return {
      user: window.username,
      room: "lobby",
    };
  },
  methods: {
    gotoRoom() {
      window.username = this.user;
      this.$router.push({
        name: "Room",
        params: { roomId: this.room },
      });
    },
  },
};
</script>

<style scoped lang="scss">
// See styles at https://github.com/mostlytyped/rethink-chat-graphql/blob/master/src/views/Home.vue
</style>
```

Now that we stuffed the skeleton with the bits an pieces we need, let us tackle the
real meat of the frontend, the GraphQL client and the chat room view.

### GraphQL client

When our frontend loads we need to initiate the GraphQL client.
In our example we use Apollo, the most used GraphQL client, which
has good Vue.js integration with the `vue-apollo` package.

```js
// src/graphql.js

import Vue from "vue";
import VueApollo from "vue-apollo";
import ApolloClient from "apollo-client";
import { createHttpLink } from "apollo-link-http";
import { InMemoryCache } from "apollo-cache-inmemory";
import { split } from "apollo-link";
import { WebSocketLink } from "apollo-link-ws";
import { getMainDefinition } from "apollo-utilities";

Vue.use(VueApollo);

// HTTP connection to the API
const httpLink = createHttpLink({
  // For production you should use an absolute URL here
  uri: `${window.location.origin}/graphql`,
});

// Create the subscription websocket link
const wsLink = new WebSocketLink({
  uri: `wss://${window.location.host}/graphql`,
  options: {
    reconnect: true,
  },
});

// Split link based on operation type
const link = split(
  ({ query }) => {
    const definition = getMainDefinition(query);
    return (
      definition.kind === "OperationDefinition" &&
      definition.operation === "subscription"
    );
  },
  wsLink, // Send subscription traffic to websocket link
  httpLink, // All other traffic to http link
);

// Create apollo client/provider with our link
const apolloClient = new ApolloClient({
  cache: new InMemoryCache(),
  link: link,
});

const apolloProvider = new VueApollo({
  defaultClient: apolloClient,
});

export default apolloProvider;
```

Since we will use GraphQL subscriptions, our Apollo setup
is a bit more complicated than usual. This is because
normal GraphQL should be performed over HTTP but
subscription updates will be pushed over a WebSocket.

### The chat room view

The final piece of the frontend will be the `ChatRoom`
view. Here we actually get to use the GraphQL client
we just initialized. This view basically shows a
list with all the items in the `chats` variable and
provides a form to send a chat message to the backend.

```vue
<!-- src/views/ChatRoom.vue -->
{% raw %}
<template>
  <div class="chatroom">
    <ul id="chatlog">
      <li v-for="chat in chats" v-bind:key="chat.ts">
        <span class="timestamp">
          {{
            new Date(chat.ts).toLocaleString(undefined, {
              dateStyle: "short",
              timeStyle: "short",
            })
          }}
        </span>
        <span class="user">{{ chat.user }}:</span>
        <span class="msg">{{ chat.msg }}</span>
      </li>
    </ul>
    <label id="username"> Username: {{ user }} </label>
    <form v-on:submit.prevent="sendMessage">
      <input v-model="message" autocomplete="off" />
      <button>Send</button>
    </form>
  </div>
</template>
{% endraw %}
<script>
import gql from "graphql-tag";

export default {
  name: "ChatRoom",
  data() {
    return {
      chats: [],
      message: "",
      user: window.username,
      handle: null,
    };
  },
  methods: {
    sendMessage() {
      const msg = this.message;
      this.$apollo.mutate({
        mutation: gql`
          mutation($user: String!, $msg: String!, $room: String!) {
            sendChat(user: $user, room: $room, message: $msg) {
              ts
            }
          }
        `,
        variables: {
          user: this.user,
          msg: msg,
          room: this.$route.params.roomId,
        },
      });
      this.message = "";
    },
  },
  apollo: {
    chats: {
      query: gql`
        query FetchChats($room: String!) {
          chats(room: $room) {
            msg
            user
            ts
          }
        }
      `,
      variables() {
        return {
          room: this.$route.params.roomId,
        };
      },
      subscribeToMore: {
        document: gql`
          subscription name($room: String!) {
            chatAdded(room: $room) {
              msg
              user
              ts
            }
          }
        `,
        variables() {
          return {
            room: this.$route.params.roomId,
          };
        },
        // Mutate the previous result
        updateQuery: (previousResult, { subscriptionData }) => {
          previousResult.chats.unshift(subscriptionData.data.chatAdded);
        },
      },
    },
  },
};
</script>

<style scoped lang="scss">
// See styles at https://github.com/mostlytyped/rethink-chat-graphql/blob/master/src/views/ChatRoom.vue
</style>
```

The `sendMessage` method is bound to the `sendChat` GraphQL mutation. As for the `chats`
variable, the binding is a bit more involved. We bind it to the GraphQL `chats` query
and in addition we use the `chatAdded` subscription to keep the variable up to date.

Now we have a working server and frontend. The last thing we need is to make sure
the `chats` table actually exists in the RethinkDB database when we run the app.

## Database migration

The app does not work without a `chats` table. We thus need a database migration that adds
the table.

```js
// server/migrate.js

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
// server/lorem-bot.js

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

release: node server/migrate.js
web: node server/index.js
lorem-bot: node server/lorem-bot.js
```

The `release` and `web` processes are recognized by Heroku as
the command to run upon release and the main web app respectively. The
`lorem-bot` process is just a worker process that could have any
name.

Deploy the app to Heroku with

```sh
$ git add .
$ git commit -m 'Working rethink-chat app'
$ git push heroku master
```

You will need to manually enable the `lorem-bot` process in your
Heroku app. You can do so on the Resources tab.
{: .alert-info}

## Conclusion

In less than 15 minutes we managed to create and deploy a chat application
with a simple bot. This shows the power and ease of use of RethinkDB.
The ability to subscribe to queries makes it easy to build a reactive
app and can easily be integrated with GraphQL. Further, Heroku makes deployment
a breeze, and with the RethinkDB Cloud add-on you will never have to do the
tedious work of managing a database server yourself.
