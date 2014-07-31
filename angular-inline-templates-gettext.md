[gulp-angular-gettext](https://www.npmjs.org/package/gulp-angular-gettext) has the ability to extract translation strings from JS and HTML files (using [angular-gettext-tools](https://www.npmjs.org/package/angular-gettext-tools)) but it can't extract inline templates defined in directives.

So here comes the `inlineTemplates` helpers:

```coffee
File = require "vinyl"
through = require "through2"

inlineTemplates = ->
  ###
  Add inline templates from directives to the stream.
  ###
  through.obj (file, enc, cb) ->
    @push(file)
    htmlPath = file.path.replace /\.js$/, ".html"
    data = file.contents.toString()
    directiveTemplateRe = ///
      \.directive\([^{]*{[^}]*                 # .directive({ ...
        \s+template:\s*((["'])[^\2]+?[^\\]\2)  #   template: "..."
    ///gm
    while (matched = directiveTemplateRe.exec(data))?
      # Get new template chunk and unquote it.
      template = JSON.parse(matched[1])
      @push(new File(path: htmlPath, contents: new Buffer(template)))
    cb()
```

Usage example:

```coffee
g = require "gulp"
coffee = require "gulp-coffee"
jade = require "gulp-jade"
gettext = require "gulp-angular-gettext"
gulpif = require "gulp-if"

g.task "i18n:extract", ->
  g.src(["app/**/*.coffee", "app/**/*.jade"])
   .pipe(gulpif("**/*.coffee", coffee()))
   .pipe(gulpif("**/*.jade", jade()))
   .pipe(gulpif("**/*.js", inlineTemplates()))
   .pipe(gettext.extract("i18n.pot"))
   .pipe(g.dest("i18n"))
```
