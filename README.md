# RethinkDB Heroku Add-on landing page

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

## Content Authoring

[Kramdown](https://jekyllrb.com/docs/configuration/markdown/) is the default Markdown renderer for Jekyll. It supports adding classes to markdown elements like so:

```markdown
A paragraph with a class of `.alert`.
{: .alert }
```

Elements can be styled as alerts by adding alert classes, as in the example above. See `style-guide.md` for details.
