# Cliche

A library for caching processing results as files; makes it easy to skip expensive operations (e.g. template compilation; minification; http requests). The cache is reused across multiple executions of your program.

## Key features

- Caches the results of long running operations, such as:
  - HTTP requests to a remote server
  - file transformations such as template compilation, jshint, uglification, packaging etc.
- File-based cache (not in-memory); the cache is reused across multiple executions of your program.
- Zero config global cache (e.g. shared across invocations) by default; can be configured with an explicit path and invalidation method.
- Stream-based API
- Handles cache invalidation based on:
  - For files:
    - the last modified date and file size of an input file
    - the md5/sha1/sha256/sha512 of an input file
  - For http requests:
    - URL and parameters
    - optionally, a max time to live

## Intro

Each expensive operation consists of four things:

- An input (a file or a remote resource)
- A task (fetch; minify; compile template)
- A set of options (e.g. arguments to the task)
- An output (a stream)

This library is designed to cache the results of any task that produces an output stream that can be written as a file.

The assumption is that if:

- the input doesn't change
- the task doesn't change and
- the set of options doesn't change

then the result from a previous execution can be reused - skipping the expensive operation.


## Examples:

Basic configuration:

    var Cache = require('cliche'),
        cache = new Cache({
          method: 'stat'
          maxAge: '1 hour'
        });

Options - all of these are optional:

- `cachepath`: the cache directory path. A directory where to store the cached results. Default: `os.tmpdir('./cache')`.
- `fs`: the method to use. Default: `stat`
    - `stat`: calls `fs.stat` on the input file path; the cached version is used if the file size and date last modified have not changed.
    - `md5` | `sha1` | `sha256` | `sha512`: reads the input file in full and calculates the given hash using Node's crypto; this uses openSSL so it is still quite fast.
  - `maxAge`: an additional constraint on the age. Supports user-friendly formats via [english-time](https://github.com/azer/english-time).
- `http`: the method to use. Default: `none`
  - `none`: only use etags and expires headers,
      - an etag is stored for each http request if the server returns one; the a `If-None-Match` header is sent.
      - any expiry headers ("Expires" or "Cache-Control: max-age") are honored.
  - `minAge`: normally, a remote request is made for any requests that do not have a expiry header set. This setting ensures that at least `minAge` time has elapsed between checks against the server.
  remote requests are never made more frequently than this.
  - `maxAge`: an additional constraint on the age. Supports user-friendly formats via [english-time](https://github.com/azer/english-time).

## API: http

- `.http`/ `.https`: wrappers for the Node native `http`/`https` module
  - `.http.get(options, callback)`: works like `http.get` but checks the cache first and returns a Readable stream
  - `.http.request(options, callback)`: like `http.request` but checks the cache first and returns a Readable stream

Note that the return object from the http interface are always Readable Streams, but not necessarily instances of `http.IncomingMessage` since that object is only created by Node when a real HTTP request is made. For interoperability, the cache adds the following commonly used properties on the return value: `.statusCode`, `.headers`, `.httpVersion`, `.httpVersionMajor`, `.httpVersionMinor`, `.trailers`. These will have whatever values the cached request returned.

## API: tasks / files

Asynchronous:

- `.lookup(options)`: returns a file name given a set of options
- `.stream(path, options)`:
    - `options.options`: a description of the options used for this task.

Good for things which output a readable stream.

Synchronous:

- `.lookup(options)`: returns a file name given a set of options
- `.filename(options)`: returns a new output file name given a set of options
- `.complete(cacheFilename, options)`: marks a given output file as completed and saves the cache metadata
- `.clear(options)`: clears the given cache directory

Good for writing simple tasks which use synchronous processing or functions (rather than streams).

### Caching HTTP client requests

Without caching:

    require('https').get({
        host : 'api.github.com',
        path : '/users/octocat'
    }, function(res) {
      // ...
    });

With caching:

    Cache.https.get({
      host : 'api.github.com',
      path : '/users/octocat'
    }, function(res) {
      // ...
    });

### Caching template compilation

    var Cache = require('cliche');
    var Handlebars = require('handlebars');

    var cacheFile = Cache.lookup(opts);
    if(cacheFile) {
      // get the result from the cache file
      fs.createReadStream(cacheFile).pipe(process.stdout);
    } else {
      // create the file in the cache folder
      cacheFile = Cache.filename(opts);

      // compile
      var js = Handlebars.precompile(fs.readFileSync()),
          result = "var Handlebars = require('handlebars-runtime');\n" +
          "module.exports = Handlebars.template(" + js.toString() + ");\n";


      // update the cached filed directly, then mark as complete
      fs.writeFileSync(cacheFile, result);
      Cache.complete(cacheFile, opts);

      // use the result
      console.log(result);
    }

### Pipe

    var Cache = require('cliche');
    var Handlebars = require('handlebars');

    var cacheFile = Cache.lookup(opts);
    if(cacheFile) {
      fs.createReadStream(cacheFile).pipe(process.stdout);
    } else {
      var foo = require('child_process').spawn('wc', ['-c']);

      // maybe better .pipe() ?
      foo.pipe(Cache.stream(opts)).pipe(process.stdout);
    }


## Command line tool

The `cliche` command line tool makes it easy to incrementally run shell commands.

It acts like a filter; only files that have been changed since the last time the command was run are passed on.

This means that commands are only run on the files that have changed; this is mostly useful for things like linters, template compilers etc. that operate on single files.

    --include <path>      Path to include; these files will be checked for changes.
    --command <cmd>       Shell command to run; any paths that have been modified will be appended to this command.
    --each <expr>         Some programs expect arguments such as "--include <path>". You can specify this via:
                          --each "--include {{path}}"; this changes the appended string.
    --dry-run             Do not run the command; just print out what the command would be.
    --method <method>     Specify the method to use for modification checks. Default: "stat".
    --cache <path>        Specify the directory to use for caching. Default: `os.tmpdir()`.
    --max-age <age>       The maximum age of files in minutes. This forces the cache to be invalidated every now and then.
    --watch               Automatically rerun the command when any of the input paths change.

### Example: Speed up jshint runs

.. by skipping files that have not changed:

    cliche --include ./bar --command "jshint"

### Example: Speed up Google Closure linter runs

.. by skipping files that have not changed:

    cliche --include ./foo \
          --include ./bar \
          --each "-r {{path}}" \
          --command "gjslint --nojsdoc --jslint_error=all"

### Example: Piping to xargs

    cliche --include ./foo | xargs -n 1 echo "Hello"
