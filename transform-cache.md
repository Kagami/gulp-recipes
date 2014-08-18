What is the most common operation for which build systems are used to? That's simple sequence of actions: read files → compile them → transform → concatenate → write. When we develop we often do a lots of changes and really want to see them instantly in the browser. We setup some sort of watch callbacks, maybe some script/plugin to reload page automatically and hope what it will work reasonably fast.

Unfortunately I almost never see (in JavaScript world) correctly implemented caching for this sequence of actions. It basically has some of this flaws:
* No caching at all because many application is small enough or we can buy better hardware (but what we should do if we have a big one…)
* Disk cache which means tons of read and writes (Grunt)
* Don't use concatenation in development mode and caching only separate files which means tons of writes for initial build (and tons of reads to load such app in a browser)

One nice exception is [Brunch](http://brunch.io/) build tool which cache exactly like it should be but I've not found plugins with similar functionality for Grunt or gulp (none of gulp-changed/gulp-cached/gulp-remember/gulp-newer helps).

So this is how to easily implement good enough caching for gulp (maybe I will make a plugin someday):

```coffee
through = require "through2"
bufferEqual = require "buffer-equal"

cache = do ->
  ###
  Cache file transformations.
  ###
  # TODO: Cache file reads.
  _caches = {}
  defaultName = "_default"

  (cacheMissTransform, opts) ->
    cacheName = opts?.name ? defaultName
    _caches[cacheName] ?= {}
    _cache = _caches[cacheName]

    through.obj (file, enc, cb) ->
      cacheKey = file.path
      cached = _cache[cacheKey]
      contents = file.contents
      if cached? and bufferEqual(contents, cached.originalContents)
        cb(null, cached.transformed.clone())
      else
        stream = through.obj()
        stream.write(file)
        # XXX: Stream manipulating should be done better.
        cacheMissTransform(stream).on "data", (transformedFile) ->
          _cache[cacheKey] =
            originalContents: contents
            transformed: transformedFile.clone()
          cb(null, transformedFile)
```

Usage (say we have coffee source files which we'd like to compile, annotate and concatenate):

```coffee
g = require "gulp"
watch = require "gulp-watch"
coffee = require "gulp-coffee"
ngAnnotate = require "gulp-ng-annotate"
concat = require "gulp-concat"

g.task "js:dev", ->
    makeTransform = (stream) ->
      stream = stream.pipe(coffee())
      stream = stream.pipe(ngAnnotate())
      stream

    g.src("src/**/*.coffee")
     .pipe(cache(makeTransform))
     .pipe(concat("app.js"))
     .pipe(g.dest("dist"))

g.task "watch:js", ->
    watch("src/**/*.coffee", ["js:dev"])
```

What does this do? It will:
* Watch for changes in src directory
* Read all files (this should be optimized actually)
* Run coffee and ngAnnotate for changed files
* Instantly return transformed result for non-changed files
* Concatenate it all together and write single resulting file to the disk
