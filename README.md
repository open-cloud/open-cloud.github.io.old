# XOS documentation on [guide.xosproject.org](https://guide.xosproject.org)

This documentation uses [GitHub Pages](https://pages.github.com), specifically the [Jekyll Integration](https://help.github.com/articles/using-jekyll-with-pages/), which can be deployed using [Bundler](http://bundler.io/).

## Previewing a local copy during editing

To set up github-pages flavored Jekyll, `bundle install --path ../github-pages` in the repo root, which creates a bundle of gems in the parent directory that are current with the ones GitHub uses to render the site from it's repo.

To view the local edits: `bundle exec jekyll serve`, then visit http://localhost:4000 .

Note that you will have to restart this command if you modify the `_config.yml` file that contains the Jekyll configuration.

## Style notes

**Quoting Text**: When quoting a section of text inline, such as method or variable names, filenames and paths, commands, and similar, use the backtick (`` ` ``) character, except when creating a URL for a filename/path link that points to that location in the code repo in which case quoting isn't needed.

**Code**: For standalone or multi-line code snippets, wrap code in tags for [Jekyll's code highlighting support](http://jekyllrb.com/docs/posts/#highlighting-code-snippets), which looks like this :
```
{% highlight sh %}
Code in shell script
{% endhighlight %}
```

If you're quoting a template that uses the same `{% %}` tags as Jekyll, you may need to [escape the code as described here](http://stackoverflow.com/questions/3426182/how-to-escape-liquid-template-tags).

**Shell Commands**: For shell commands that are intended to be run interactively by a user, prefix line with `$` if run as a unprivileged user, or `#` if being run as root.  Put a space between the prefix and the command. If you need to specify which user is running a commands, or on which host, add that to the prefix (ex: `username@host $`).

**Figures/Images**: When embedding figures:

 1. Name the image file with a prefix of the page where it's being used,
 2. Optionally prefix it with the figure number and a description.
 3. Place in `/figures`
 4. Use the Jekyll `include` command to add it with the figure template and a caption/alttext:

```
{% include figure.html url="/figures/archguide-fig02_service_anatomy.jpg" caption="Figure 2. Anatomy of a Service." %}
```

## Converting documents to Markdown

Note - all the below methods require a bit of cleanup after conversion, but should do the vast majority of the work required.

### PDF

The [Poppler](http://poppler.freedesktop.org) library includes the utilties `pdftotext` and `pdfimages`.

Extract text: `pdftotext -layout -enc UTF-8 file.pdf file.txt`

Extract images: `pdfimages -all file.pdf image_prefix` 

*Note:* Text extraction only works when the document contains text, and isn't a scanned picture of text or obfuscated in some manner.

### Google Docs

The [gdocs2md](https://github.com/mangini/gdocs2md) tool can convert to markdown, then email you a zipfile containing the documents text and images. 

### Other text/document formats

Many file formats are supported by [Pandoc](http://pandoc.org/), which can also provide high quality PDF's. 


