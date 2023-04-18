# RethinkDB Heroku Add-on landing page ([rethinkdb.cloud](https://www.rethinkdb.cloud))

A simple landing page, built as a static site on [Jekyll](https://jekyllrb.com/).

## Running locally

Install Jekyll:

```
gem install bundler jekyll
```

Serve locally:

```bash
bundle exec jekyll serve
```

Build for production:

```bash
bundle exec jekyll build
```

## Theme Customization

This site uses the Minima Jekyll theme, with the Solarized Dark skin configured in `_config.yml`. Any skin should be able to be configured without issue.

Minima sass has not been modified, all custom styling is layered on in `_sass/minima/custom-styles.scss` and `_sass/minima/custom-vaiables.scss`. Avoid editing Minima CSS if possible. It should generally not be necessary to do so.

## Formatting a Blog Post

Inside the front matter you need to include the

- Title
- Tags
- Author

A default post `layout` and `excerpt_separator` are configured in `_config.yml`.

Example:

```markdown
---
title: How to be a pirate
tags: pirates how-to sailing
author: Sascha
---

Swing the lead league galleon capstan Cat o'nine tails pinnace marooned broadside long clothes wench...
```

### Do not add the title as a h1 header to the top of your post body.

BAD:

```markdown
---
title: How to be a pirate
tags: pirates sailing
author: Sascha
---

# How to be a pirate

Swing the lead league galleon capstan Cat o'nine tails pinnace marooned broadside long clothes wench...
```

### Add a teaser tag

After about the first paragraph or so, add a teaser tag that will appear on the
main page of the blog.

```markdown
---
title: "Sea creatures in my beard this week"
author: Sascha
---

American Main measured fer yer chains Buccaneer brig strike colors careen pillage spanker gangplank furl. Pressgang Brethren of the Coast warp brig walk the plank Nelsons folly trysail keelhaul hornswaggle **Letter of Marque**. Splice the main brace rutters.

<!--more-->

Landlubber or just lubber wench Sea Legs reef run a rig cable black jack crack Jennys tea cup...
```

### Alerts

[Kramdown](https://jekyllrb.com/docs/configuration/markdown/) is the default Markdown renderer for Jekyll. It supports adding classes to markdown elements like so:

```markdown
A paragraph with a class of `.alert`.
{: .alert }
```

Elements can be styled as alerts by adding alert classes, as in the example above. See `style-guide.md` for details.

### Code Blocks

When creating a code block, unless you want to implement custom `<pre>` tags,
**aways** use back-tics followed by the language tag. This is the only way to
add custom syntax highlighting.

````markdown
```javascript
var memjs = require('memjs')

var mc = memjs.Client.create(process.env.MEMCACHIER_SERVERS, {
  failover: true,  // default: false
  timeout: 1,      // default: 0.5 (seconds)
  keepAlive: true  // default: false
})

mc.set('hello', 'memcachier', {expires:0}, function(err, val) {
  if(err != null) {
    console.log('Error setting value: ' + err)
  }
})

...
```
````

#### Including code blocks in lists.

If you would like to ensure that your code block is indented at the same level
as the rest of a list, you need to _add four spaces_ to the beginning of every line.

````markdown
1. Foo

2. Bar

   \```javascript
   console.log("foobar")
   \```

   and then some stuff after the code block is done.

3. Bazz and a bunch of other stuff that will hopefully fall into a nice indented
   line together or something like that. yadda yadda yadda.
````
