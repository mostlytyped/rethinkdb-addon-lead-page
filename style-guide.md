---
layout: page
title: "Style Guide"
sitemap: false
---

## Plain ol' blockquote

> Keelhaul long boat scuppers haul wind trysail long clothes Sink me port parley list. `<p>Some html</p>`. Topgallant Nelsons folly splice the main brace strike colors come about, *italic text*, nipper ahoy hogshead quarter holystone. [This is a link](#). Reef list hempen halter ye mutiny hardtack, **bold text**, swab killick driver no prey, no pay.

## Multiline blockquote

> ### Deadlights jack lad schooner `h3`
>
> - Shiver me timbers to go on.
> - Trysail Sail ho Corsair red.
>
> #### Fluke jib scourge of the seven seas `h4`
>
> 1. Shiver me timbers to go on.
> 2. Trysail Sail ho Corsair red.
>
> ```js
> r = require("rethinkdb");
> r.connect(
>   {
>     host: process.env.RETHINKDB_HOST || "localhost",
>     port: process.env.RETHINKDB_PORT || 28015,
>     username: process.env.RETHINKDB_USERNAME || "admin",
>     password: process.env.RETHINKDB_PASSWORD || "",
>     db: process.env.RETHINKDB_NAME || "test",
>   },
>   function (err, conn) {
>     if (err) throw err;
>     r.tableCreate("tv_shows").run(conn, function (err, res) {
>       if (err) throw err;
>       console.log(res);
>       r.table("tv_shows")
>         .insert({ name: "Star Trek TNG" })
>         .run(conn, function (err, res) {
>           if (err) throw err;
>           console.log(res);
>         });
>     });
>   },
> );
> ```
>
>  *Everything* is going according to **plan**.

## Alerts

Format alerts by adding a Kramdown class to an element. Preferring `<p>` probably makes more semantic sense than `<blockquote>`.

### Primary `.alert`

```markdown
Keelhaul long boat...
{: .alert }
```

Keelhaul long boat scuppers haul wind trysail long clothes Sink me port parley list. `<p>Some html</p>`. Topgallant Nelsons folly splice the main brace strike colors come about, *italic text*, nipper ahoy hogshead quarter holystone. [This is a link](#). Reef list hempen halter ye mutiny hardtack, **bold text**, swab killick driver no prey, no pay.
{: .alert }

### Secondary `.alert-secondary`

Keelhaul long boat scuppers haul wind trysail long clothes Sink me port parley list. `<p>Some html</p>`. Topgallant Nelsons folly splice the main brace strike colors come about, *italic text*, nipper ahoy hogshead quarter holystone. [This is a link](#). Reef list hempen halter ye mutiny hardtack, **bold text**, swab killick driver no prey, no pay.
{: .alert-secondary}

### Success `.alert-success`

Keelhaul long boat scuppers haul wind trysail long clothes Sink me port parley list. `<p>Some html</p>`. Topgallant Nelsons folly splice the main brace strike colors come about, *italic text*, nipper ahoy hogshead quarter holystone. [This is a link](#). Reef list hempen halter ye mutiny hardtack, **bold text**, swab killick driver no prey, no pay.
{: .alert-success}

### Info `.alert-info`

Keelhaul long boat scuppers haul wind trysail long clothes Sink me port parley list. `<p>Some html</p>`. Topgallant Nelsons folly splice the main brace strike colors come about, *italic text*, nipper ahoy hogshead quarter holystone. [This is a link](#). Reef list hempen halter ye mutiny hardtack, **bold text**, swab killick driver no prey, no pay.
{: .alert-info}

### Warning `.alert-warning`

Keelhaul long boat scuppers haul wind trysail long clothes Sink me port parley list. `<p>Some html</p>`. Topgallant Nelsons folly splice the main brace strike colors come about, *italic text*, nipper ahoy hogshead quarter holystone. [This is a link](#). Reef list hempen halter ye mutiny hardtack, **bold text**, swab killick driver no prey, no pay.
{: .alert-warning}

### Danger `.alert-danger`

Keelhaul long boat scuppers haul wind trysail long clothes Sink me port parley list. `<p>Some html</p>`. Topgallant Nelsons folly splice the main brace strike colors come about, *italic text*, nipper ahoy hogshead quarter holystone. [This is a link](#). Reef list hempen halter ye mutiny hardtack, **bold text**, swab killick driver no prey, no pay.
{: .alert-danger}

## Multiline blockquote alerts

> ### Deadlights jack lad schooner `h3`
>
> - Shiver me timbers to go on.
> - Trysail Sail ho Corsair red.
>
> #### Fluke jib scourge of the seven seas `h4`
>
> 1. Shiver me timbers to go on.
> 2. Trysail Sail ho Corsair red.
>
> ```js
> r = require("rethinkdb");
> r.connect(
>   {
>     host: process.env.RETHINKDB_HOST || "localhost",
>     port: process.env.RETHINKDB_PORT || 28015,
>     username: process.env.RETHINKDB_USERNAME || "admin",
>     password: process.env.RETHINKDB_PASSWORD || "",
>     db: process.env.RETHINKDB_NAME || "test",
>   },
>   function (err, conn) {
>     if (err) throw err;
>     r.tableCreate("tv_shows").run(conn, function (err, res) {
>       if (err) throw err;
>       console.log(res);
>       r.table("tv_shows")
>         .insert({ name: "Star Trek TNG" })
>         .run(conn, function (err, res) {
>           if (err) throw err;
>           console.log(res);
>         });
>     });
>   },
> );
> ```
>
>  *Everything* is going according to **plan**.
{: .alert-secondary }

> ### Deadlights jack lad schooner `h3`
>
> - Shiver me timbers to go on.
> - Trysail Sail ho Corsair red.
>
> #### Fluke jib scourge of the seven seas `h4`
>
> 1. Shiver me timbers to go on.
> 2. Trysail Sail ho Corsair red.
>
> ```js
> r = require("rethinkdb");
> r.connect(
>   {
>     host: process.env.RETHINKDB_HOST || "localhost",
>     port: process.env.RETHINKDB_PORT || 28015,
>     username: process.env.RETHINKDB_USERNAME || "admin",
>     password: process.env.RETHINKDB_PASSWORD || "",
>     db: process.env.RETHINKDB_NAME || "test",
>   },
>   function (err, conn) {
>     if (err) throw err;
>     r.tableCreate("tv_shows").run(conn, function (err, res) {
>       if (err) throw err;
>       console.log(res);
>       r.table("tv_shows")
>         .insert({ name: "Star Trek TNG" })
>         .run(conn, function (err, res) {
>           if (err) throw err;
>           console.log(res);
>         });
>     });
>   },
> );
> ```
>
>  *Everything* is going according to **plan**.
{: .alert-success }

> ### Deadlights jack lad schooner `h3`
>
> - Shiver me timbers to go on.
> - Trysail Sail ho Corsair red.
>
> #### Fluke jib scourge of the seven seas `h4`
>
> 1. Shiver me timbers to go on.
> 2. Trysail Sail ho Corsair red.
>
> ```js
> r = require("rethinkdb");
> r.connect(
>   {
>     host: process.env.RETHINKDB_HOST || "localhost",
>     port: process.env.RETHINKDB_PORT || 28015,
>     username: process.env.RETHINKDB_USERNAME || "admin",
>     password: process.env.RETHINKDB_PASSWORD || "",
>     db: process.env.RETHINKDB_NAME || "test",
>   },
>   function (err, conn) {
>     if (err) throw err;
>     r.tableCreate("tv_shows").run(conn, function (err, res) {
>       if (err) throw err;
>       console.log(res);
>       r.table("tv_shows")
>         .insert({ name: "Star Trek TNG" })
>         .run(conn, function (err, res) {
>           if (err) throw err;
>           console.log(res);
>         });
>     });
>   },
> );
> ```
>
>  *Everything* is going according to **plan**.
{: .alert-info }

> ### Deadlights jack lad schooner `h3`
>
> - Shiver me timbers to go on.
> - Trysail Sail ho Corsair red.
>
> #### Fluke jib scourge of the seven seas `h4`
>
> 1. Shiver me timbers to go on.
> 2. Trysail Sail ho Corsair red.
>
> ```js
> r = require("rethinkdb");
> r.connect(
>   {
>     host: process.env.RETHINKDB_HOST || "localhost",
>     port: process.env.RETHINKDB_PORT || 28015,
>     username: process.env.RETHINKDB_USERNAME || "admin",
>     password: process.env.RETHINKDB_PASSWORD || "",
>     db: process.env.RETHINKDB_NAME || "test",
>   },
>   function (err, conn) {
>     if (err) throw err;
>     r.tableCreate("tv_shows").run(conn, function (err, res) {
>       if (err) throw err;
>       console.log(res);
>       r.table("tv_shows")
>         .insert({ name: "Star Trek TNG" })
>         .run(conn, function (err, res) {
>           if (err) throw err;
>           console.log(res);
>         });
>     });
>   },
> );
> ```
>
>  *Everything* is going according to **plan**.
{: .alert-warning }

> ### Deadlights jack lad schooner `h3`
>
> - Shiver me timbers to go on.
> - Trysail Sail ho Corsair red.
>
> #### Fluke jib scourge of the seven seas `h4`
>
> 1. Shiver me timbers to go on.
> 2. Trysail Sail ho Corsair red.
>
> ```js
> r = require("rethinkdb");
> r.connect(
>   {
>     host: process.env.RETHINKDB_HOST || "localhost",
>     port: process.env.RETHINKDB_PORT || 28015,
>     username: process.env.RETHINKDB_USERNAME || "admin",
>     password: process.env.RETHINKDB_PASSWORD || "",
>     db: process.env.RETHINKDB_NAME || "test",
>   },
>   function (err, conn) {
>     if (err) throw err;
>     r.tableCreate("tv_shows").run(conn, function (err, res) {
>       if (err) throw err;
>       console.log(res);
>       r.table("tv_shows")
>         .insert({ name: "Star Trek TNG" })
>         .run(conn, function (err, res) {
>           if (err) throw err;
>           console.log(res);
>         });
>     });
>   },
> );
> ```
>
>  *Everything* is going according to **plan**.
{: .alert-danger }