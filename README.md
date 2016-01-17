# XOS documentation on [guide.xosproject.org](https://guide.xosproject.org)

This documentation uses [GitHub Pages](https://pages.github.com), specifically the [Jekyll Integration](https://help.github.com/articles/using-jekyll-with-pages/), which can be deployed using [Bundler](http://bundler.io/). 

## Previewing a local copy during editing

To set up github-pages flavored Jekyll, `bundle install --path ../github-pages` in the repo root, which creates a bundle of gems in the parent directory that are current with the ones GitHub uses to render the site from it's repo.

To view the local edits: `bundle exec jekyll serve`, then visit http://localhost:4000 .

## Style notes

When quoting a section of text inline, such as method or variable names, filenames and paths, commands, and similar, use the backtick (`` ` ``) character.  One exception - when creating a URL for a filename/path link that points to that location in the code repo, quoting isn't needed.

For standalone or multi-line code snippets, wrap code in tags for [Jekyll's code highlighting support](http://jekyllrb.com/docs/posts/#highlighting-code-snippets). You do not need to add additional indentation to the code.

For shell commands that are intended to be run interactively by a user, prefix each line with `$` if run as a unprivledged user, or `#` if being run as root.  Put a space between the prefix and the command. If you need to specify which user is running a commands, or on which host, add that to the prefix (ex: `username@host $`).

When embedding figures, name the image file with a prefix of the section where it's being used, place in `/figures`, and then use the Jekyll `include` command to add it with the figure template and a caption/alttext:

```
    {% include figure.html url="/figures/archguide-fig02_service_anatomy.jpg" caption="Figure 2. Anatomy of a Service." %}
```
