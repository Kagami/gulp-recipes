I don't like the existing rev-* plugins (helps with the [caching](https://developers.google.com/speed/docs/best-practices/caching)) because none of them provide flexible enough settings so this is how to implement simple rev plugin directly in the gulpfile:

```coffee
crypto = require "crypto"
through = require "through2"

# rev'd files map.
hashes = {}

rev = (hashes) ->
  ###
  Own version of rev plugin with custom hash.
  ###
  sha1 = (data) ->
    crypto.createHash("sha1").update(data).digest("hex")

  through.obj (file, enc, cb) ->
    filename = path.basename(file.path)
    filedir = path.dirname(file.path)
    hash = sha1(file.contents.toString())
    hashedName = "#{hash[...12]}.#{filename}"
    hashes[filename] = hashedName
    file.path = path.join(filedir, hashedName)
    @push(file)
    cb()

rev.replace = (hashes) ->
  ###
  Replace appearances of rev'd files with their new names.
  ###
  through.obj (file, enc, cb) ->
    str = file.contents.toString()
    for nameOld, nameNew of hashes
      str = str.replace(nameOld, nameNew)
    file.contents = new Buffer(str)
    @push(file)
    cb()
```

Usage:

```coffee
g = require "gulp"
minifyCSS = require "gulp-minify-css"
htmlmin = require "gulp-htmlmin"

g.task "css:release", ->
  css()  # some fn which produces CSS stream
    .pipe(minifyCSS())
    .pipe(rev(hashes))
    .pipe(g.dest("dist/static"))

g.task "html:release", ->
  html()  # some fn which produces index HTML stream
    .pipe(htmlmin())
    .pipe(rev.replace(hashes))
    .pipe(g.dest("dist/static"))
```
