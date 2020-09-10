---
layout: post
title: "Get Historical Stats for your Dev.to Articles"
tags: dev.to stats rethinkdb heroku
author: Sascha
excerpt_separator: <!--more-->
---

Recently, I posted my [first article on Dev.to](https://dev.to/saschat/build-a-chat-app-with-socket-io-and-rethinkdb-3did).
As expected there was an initial wave of some views which eventually stopped.
Two weeks later I posted a
[second article](https://dev.to/saschat/bring-rethinkdb-s-realtime-magic-to-the-frontend-with-graphql-mn7).
It was related to the first article and had a link to it. As expected,
the views of my first post went up again. Since I love data, I wanted to see
how many views belonged to the first wave and how the views on my articles change
based on the what else I publish or other events that might happen.
Unfortunately, Dev.to only shows me the final view count. To still my data hunger,
I created a little app...

<!--more-->

Feel free to visit the [running app](https://devto-stats.herokuapp.com/) for my stats, or
check out the [code repository](https://github.com/mostlytyped/dev.to-stats) to dive
into the code immediately.
{: .alert-info}

## TL;DR (for the efficient and impatient)

You want to deploy the app for your own Dev.to stats? You don't have time to
read step by step tutorials? Well, here you go...

Deploying the app to Heroku only takes a minute but before you can do this, you need to
do the following:

- If you don't have one already, create a [Heroku account](https://signup.heroku.com/)
  and install their [CLI](https://devcenter.heroku.com/articles/heroku-cli)
- Go to [RethinkDB Cloud](https://www.rethinkdb.cloud/) and request free alpha access to the
  RethinkDB Cloud add-on.
- Get an API key from Dev.to (Settings → Account → DEV API Keys)

Now run:

```sh
$ git clone git@github.com:mostlytyped/dev.to-stats.git
$ cd dev.to-stats/
$ heroku create
$ heroku addons:create rethinkdb
$ heroku config:set API_KEY=<YOUR_DEV_TO_API_KEY>
$ git push heroku master
$ heroku open
```

_**Note:** the collection worker needs to be enabled manually on the apps resources dashboard.
Also, free Heroku apps are put to sleep when they are inactive so the stats collector will
not run regularly unless you pay for a dyno. Alternatively, you can visit your site once a
day. For now, this is what I do ;-)_
{: .alert-warning}

Thats it, you are done. Enjoy your historical stats.

## A step by step tutorial

Going fast is great but how about learning new skills and tools? In this section you
will learn how to create the app from the ground up. In particular, you will...

- ... learn how to use RethinkDB, the totally awesome document database. It is like
  MongoDB but has reactivity built. This allows you to subscribe to queries. Oh, and
  it is still open source!
- ... create an embedded Vue.js app. That is a Vue app that you don't have to
  compile. I bet you haven't done that before.
- ... use Chart.js to plot the stats. It is always useful to have a plotting library
  in your tool kit.

### Application setup

We will build a Node.js app, so you need to have `node` and `npm` installed.
If you want to deploy your app to Heroku, you will also need a
[Heroku account](https://signup.heroku.com/), as well having their
[CLI](https://devcenter.heroku.com/articles/heroku-cli) installed.
To run your app locally, you need to
[install and run a RethinkDB instance](https://rethinkdb.com/).

To create the application, run the following in a terminal.

```sh
$ mkdir devto-stats && cd devto-stats
$ npm init -y
$ npm install rethinkdb express morgan axios
```

This will initialize a Node.js app and install all required dependencies.

### Prepare a Heroku app

In order to deploy the application to Heroku we need to create a
Heroku app:

```sh
$ git init
$ heroku create
```

We will also need a RethinkDB instance to store articles and their daily stats.
You can do this via the [RethinkDB Cloud add-on](/) as follows:

```sh
$ heroku addons:create rethinkdb
```

The RethinkDB Cloud add-on is currently in alpha. [Request an invite for your
Heroku account email](/).
{: .alert-warning}

### Get the Dev.to API key

To access the stats of your articles you need an API key from Dev.to. You can
get one under Settings → Account → DEV API Keys. Add the key to your Heroku
app:

```sh
$ heroku config:set API_KEY=<YOUR_DEV_TO_API_KEY>
```

### Collect the stats

To collect the stats we basically need to do two things on repeat:
(i) get the stats for you articles from Dev.to and (ii) save
the stats to RethinkDB. We need to run the stats collection at
least every 24h to make sure we get stats once per day (Dev.to only
updates the stats once daily).

```js
// collect.js

const axios = require("axios");
const r = require("rethinkdb");
const { getRethinkDB } = require("./reql.js");

// Get articles from Dev.to
// ...

// Save article stats to RethinkDB
// ...

// Run once immediately
saveStats();
// Interval should be less than 24h. Running more than once a day
// is not a problem but a missed day cannot be recovered.
const interval = 6 * 60 * 60 * 1000; // 6h
setInterval(saveStats, interval);
```

To get the stats we run a simple `axios` request. Since the articles
are paged we query new pages until we get one that is not
full. The `API_KEY` environment variable contains your Dev.to
API key.

```js
// collect.js
// ...

// Get articles from Dev.to
const getArticles = async function () {
  let articles = [];
  let page = 1;
  while (true) {
    let articles_page = await axios.get(
      "https://dev.to/api/articles/me?page=" + page,
      {
        headers: {
          "api-key": process.env.API_KEY,
        },
      },
    );
    articles.push(...articles_page.data);

    // If a page is not full we are done
    if (articles_page.data.length < 30) {
      break;
    }
  }
  return articles;
};

// ...
```

When saving the stats for the day we first need to check if the article
already exists in our database. If not we add it. Then we save the stats
as long as we have not done so already today.

```js
// collect.js
// ...

// Save article stats to RethinkDB
const saveStats = async function () {
  const now = new Date();
  let day = ("0" + now.getDate()).slice(-2);
  let month = ("0" + (now.getMonth() + 1)).slice(-2);
  let year = now.getFullYear();
  const today = year + "-" + month + "-" + day;
  console.log("Collect stats:", today);

  // Get all articles
  const articles = await getArticles();

  // Save stats
  let conn = await getRethinkDB();
  articles.forEach(async (article) => {
    let db_article = await r.table("articles").get(article.id).run(conn);
    if (!db_article) {
      // New article -> save
      await r
        .table("articles")
        .insert({
          id: article.id,
          title: article.title,
          url: article.url,
          latest_stats: today,
        })
        .run(conn);
      // Save stats
      await r
        .table("stats")
        .insert({
          article_id: article.id,
          date: today,
          comments: article.comments_count,
          reactions: article.public_reactions_count,
          views: article.page_views_count,
        })
        .run(conn);
    } else if (db_article.latest_stats < today) {
      // Existing article -> update
      await r
        .table("articles")
        .get(article.id)
        .update({ latest_stats: today })
        .run(conn);
      // Save stats
      await r
        .table("stats")
        .insert({
          article_id: article.id,
          date: today,
          comments: article.comments_count,
          reactions: article.public_reactions_count,
          views: article.page_views_count,
        })
        .run(conn);
    } else {
      console.log("Already got stats today for article " + article.id);
    }
  });
};

// ...
```

As you might have noticed get the RethinkDB connection from `reql.js`. Let's
implement this now.

### Handling the RethinkDB connection

Connecting to RethinkDB is straight-forward. We only add a bit of logic to handle
disconnections gracefully. The `RETHINKDB_*` environment variables will be set
automatically by the RethinkDB Cloud add-on. The defaults work for a locally
running RethinkDB instance.

```js
// reql.js

const r = require("rethinkdb");

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
    // Handle error
    conn.on("error", function (e) {
      console.log("RDB connection error occurred: ", e);
      conn.close();
    });
    // Handle timeout
    conn.on("timeout", function (e) {
      console.log("RDB connection timed out: ", e);
      conn.close();
    });

    console.log("Connected to RethinkDB");
    rdbConn = conn;
    return conn;
  } catch (err) {
    throw err;
  }
};
exports.getRethinkDB = async function () {
  if (rdbConn != null) {
    return rdbConn;
  }
  return await rdbConnect();
};
```

### Building the server

The server is a simple Express.js app that serves a static
frontend from the `public` directory. The server listens to
get requests for one route (`/article_stats`) and returns an
array of articles and their stats.

```js
// index.js

// Express app
const express = require("express");
const app = express();

// Logging middleware
const morgan = require("morgan");
app.use(morgan("combined"));

// Serve frontend
app.use(express.static("public"));

// Lazy RethinkDB connection
const r = require("rethinkdb");
const { getRethinkDB } = require("./reql.js");

// Route to get stats
app.get("/article_stats", async (req, res) => {
  const conn = await getRethinkDB();
  let article_cursor = await r.table("articles").run(conn);
  let articles = await article_cursor.toArray();
  let article_stats = await Promise.all(
    articles.map(async (article) => {
      let stats_cursor = await r
        .table("stats")
        .filter({ article_id: article.id })
        .orderBy(r.asc("date"))
        .run(conn);
      let stats = await stats_cursor.toArray();
      article.stats = stats;
      return article;
    }),
  );
  res.json(article_stats);
});

// Start server
const listenPort = process.env.PORT || "3000";
app.listen(listenPort, () => {
  console.log(`Listening on ${listenPort}`);
});
```

### Building the frontend

For our frontend we will use an embedded Vue.js app. This makes the frontend simple,
but gives us access to all of Vue's awesome features. The frontend consists of a layout file
as well as JavaScript and CSS assets.

#### HTML layout

The layout file only serves as a mount point for the Vue app in addition to importing
the dependencies and assets.

```html
<!-- public/index.html -->

<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Dev.to Historical Stats</title>
    <link href="/css/main.css" rel="stylesheet" />
  </head>

  <body>
    <div id="app"></div>
    <script src="https://cdn.jsdelivr.net/npm/vue/dist/vue.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/chart.js@2.8.0"></script>
    <script src="/js/app.js" language="javascript"></script>
  </body>
</html>
```

#### Style sheet

The CSS asset mostly contains the styling for the article table. It is nothing fancy.

```css
/* public/css/main.css */

#app {
  margin: auto;
  width: 80%;
}

#articles {
  font-family: "Trebuchet MS", Arial, Helvetica, sans-serif;
  border-collapse: collapse;
  width: 100%;
}

#articles td,
#articles th {
  border: 1px solid #ddd;
  padding: 8px;
}

#articles tr:nth-child(even) {
  background-color: #f2f2f2;
}

#articles tr:hover {
  background-color: #ddd;
}

#articles th {
  padding-top: 12px;
  padding-bottom: 12px;
  text-align: left;
  background-color: #5c9ead;
  color: white;
}
```

#### The Vue app

The JavaScript asset `app.js` contains the actual Vue app. It consists of a simple
component with a Chart.js canvas and an article table in the template. When the
component gets created we will get the stats data from the server and create
the actual Chart.js chart.

```js
// public/js/app.js

// Vue App
const App = Vue.component("app", {
  data() {
    return {
      articleStats: [],
      chart: {},
    };
  },
  async created() {
    /* Get stats data */
    // ...
    /* Create Chart.js plot from data */
    // ...
  },

  template: `{% raw %}
  <div id="app">
    <div>
        <canvas id="chart"></canvas>
    </div>
    <table id="articles">
      <tr>
        <th></th>
        <th>Article</th>
        <th>Views</th>
        <th>Reactions</th>
      </tr>
      <tr v-for="article in articleStats">
        <td :style="{'background-color': article.color, width: '10px'}"></td>
        <td><a :href=article.url class="title">{{ article.title }}</a></td>
        <td>{{ article.stats[article.stats.length - 1].views }}</td>
        <td>{{ article.stats[article.stats.length - 1].reactions }}</td>
      </tr>
    </table>
  </div>{% endraw %}
      `,
});

// Mount Vue app
var app = new Vue({
  render: (h) => h(App),
}).$mount("#app");
```

We fetch the article stats from the `/article_stats` route on the
server. In addition we add a random color to each article that we will
use for the line in the chart.

```js
// public/js/app.js
// ...

/* Get stats data */

// Fetch article stats from server
const url = new URL(
  document.location.protocol + "//" + document.location.host + "/article_stats",
);
const articleStatsResp = await fetch(url);
let articleStats = await articleStatsResp.json();

// Assign random color to article
const randomColor = function () {
  var r = Math.floor(Math.random() * 255);
  var g = Math.floor(Math.random() * 255);
  var b = Math.floor(Math.random() * 255);
  return "rgb(" + r + "," + g + "," + b + ")";
};
articleStats.forEach((article) => {
  article.color = randomColor();
});
this.articleStats = articleStats;

// ...
```

Now we need to transform the stats into a Chart.js config object. We will do so in
three steps:

1. We need the x-axis labels. For this we will use the date fields from the longest
   stats array of all articles (oldest article).
2. Then we transform the article stats into datasets Chart.js can plot. Most importantly
   we need to prepend `0` values to the stats array of newer articles to make sure they are
   all the same length.
3. Create a Chart.js config object with all the display options we want.

Once we have the Chart.js config object we create a new chart and mount it in the
designated HTML canvas element.

```js
// public/js/app.js
// ...

/* Create Chart.js plot from data */

// Get x-Axis labels
let labels = [];
let minDate = "9"; // This will work for the next ~8000 years
this.articleStats.forEach((article) => {
  if (article.stats[0].date < minDate) {
    minDate = article.stats[0].date;
    labels = article.stats.map((stat) => {
      return stat.date;
    });
  }
});

// Transform article stats into Chart.js datasets
let datasets = this.articleStats.map((article) => {
  let data = [];
  // Fill with 0 until first view
  for (let date of labels) {
    if (date >= article.stats[0].date) {
      break;
    }
    data.push(0);
  }
  // Append views
  data.push(
    ...article.stats.map((stat) => {
      return stat.views;
    }),
  );
  // Return data set for this article
  return {
    label: article.title,
    data: data,
    fill: false,
    borderColor: article.color,
    backgroundColor: article.color,
  };
});

// Chart.js config
let chartConfig = {
  type: "line",
  data: {
    datasets: datasets,
    labels: labels,
  },
  options: {
    responsive: true,
    // aspectRatio: 3,
    title: {
      display: true,
      text: "Dev.to Article Stats",
    },
    tooltips: {
      mode: "index",
      intersect: false,
    },
    hover: {
      mode: "nearest",
      intersect: true,
    },
    scales: {
      xAxes: [
        {
          display: true,
          scaleLabel: {
            display: true,
            labelString: "Date",
          },
        },
      ],
      yAxes: [
        {
          display: true,
          scaleLabel: {
            display: true,
            labelString: "Views",
          },
        },
      ],
    },
  },
};

// Create chart
let ctx = document.getElementById("chart").getContext("2d");
this.chart = new Chart(ctx, chartConfig);

// ...
```

Now we have the fronend and the server that serves it. Before we can deploy and run
our app we only need a migration script to create the actual tables in the database.

### Database migration

The app does not work without a `articles` and `stats` tables. We thus need a database
migration that adds these.

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
  async function (err, conn) {
    if (err) throw err;
    console.log("Get table list");
    let cursor = await r.tableList().run(conn);
    let tables = await cursor.toArray();

    // Check if articles table exists
    if (!tables.includes("articles")) {
      // Table missing --> create
      console.log("Creating articles table");
      await r.tableCreate("articles").run(conn);
      console.log("Creating articles table -- done");
    }

    // Check if stats table exists
    if (!tables.includes("stats")) {
      // Table missing --> create
      console.log("Creating stats table");
      await r.tableCreate("stats").run(conn);
      console.log("Creating stats table -- done");
      // Create index
      await r
        .table("stats")
        .indexCreate("article_date", [r.row("article_id"), r.row("date")])
        .run(conn);
      console.log("Creating article-date secondary index -- done");
    }

    await conn.close();
  },
);
```

This migration checks if the tables exists and creates them if they are missing. For the
`stats` table we will also create a secondary index to make sure there is only ever one
stats document for the same `article_id` and `date`.

## Deploy the application to Heroku

To deploy our working application to Heroku we need
to create a `Procfile`. This file basically tells Heroku
what processes to run.

```
// Procfile

release: node migrate.js
web: node index.js
collect: node collect.js
```

The `release` and `web` processes are recognized by Heroku as
the command to run upon release and the main web app respectively. The
`collect` process is just a worker process that could have any name.

Deploy the app to Heroku with

```sh
$ echo "node_modules/" > .gitignore
$ git add .
$ git commit -m 'Working dev.to stats app'
$ git push heroku master
```

You will need to manually enable the `collect` process in your
Heroku app. You can do so on the Resources tab.
{: .alert-info}

## Wrapping up

With this app running on Heroku I can finally go back to coding and writing articles
without missing any move my article stats might be making. Let me know if this
app is useful to you, if there are any bugs, or if there are any features you would
want me to add.
