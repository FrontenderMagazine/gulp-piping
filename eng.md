# Gulp-style stream piping in Grunt, or anywhere else

![gulp][gulp]

Recently a new front-end build tool, [Gulp][1], has been getting quite some
attention. What makes it really elegant is its use of Streams to orchestrate the
build flow. Since everything is passed down in the buffer, it gets rid of the
need to create temporary on-disk files.

It already has an active community contributing plugins for it, but still
lacking a few plugins that I need for testing purposes, so I have to stick with
Grunt for now. I can of course use Gulp and Grunt side by side, but I really
don’t want two build systems in a single project. The other problem I have with
Gulp is its task concurrency model. Everything runs in parallel by default and
it’s a bit awkward when you simply want to have a list of tasks to run in order
(which Grunt does by default).

Luckily, one of the authors of Gulp has refactored gulp’s file system out of its
core as a standalone module: [vinyl-fs][2]. Its `.src()` and `.dest()` methods
are actually all you need to leverage existing Gulp plugins. The module is not
in npm yet as of this writing, but you can install it using its github
shorthand:

    $ npm install wearefractal/vinyl-fs

With `vinyl-fs` we can easily use Gulp plugins inside Grunt tasks:

    var fs = require('vinyl-fs'),
        gzip = require('gulp-gzip')

    grunt.registerTask('zip', function () {
      fs.src('./build/build.js')
        .pipe(gzip())
        .pipe(fs.dest('./build'))
        .on('end', this.async())
    })

You might have noticed that some Gulp plugins really just do one trivial thing,
like renaming, adding a header, adding a footer, etc. I’m okay with these tiny
extra dependencies, but what annoys me a little bit is the fact that almost all
of these plugins have the entire [event-stream][3] module as a dependency but
really only uses its .map() function, which is available as the standalone [map-
stream][4] module. Also, what if I want to do something to the file that doesn’t
have a plugin for?

It’s in fact super easy to hand-roll some of these functionalities with just 
`map-stream`:

    var fs = require('vinyl-fs'),
        map = require('map-stream'),
        zlib = require('zlib')
    grunt.registerTask('zip', function () {
      fs.src('./build/build.js')
        .pipe(map(gzip))
        .pipe(fs.dest('./build'))
        .on('end', this.async())
    })
    // all you need to do is modify the file's
    // contents and then call the callback.
    // Just remember it's always buffer in and buffer out.
    function gzip (file, cb) {
      zlib.gzip(file.contents, function (err, buffer) {
        file.contents = buffer
        file.path = file.path + '.gz'
        cb(err, file)
      })
    }

By piping things through a mapped Stream, you can do whatever you want with the
file — alter the contents, change file name, etc. The nice thing about this is
you can use raw dependencies (e.g. just use uglify-js for an uglify task) with a
custom script, run it with Grunt, make, npm run or whatever you like instead of
waiting for someone to write a gulp plugin for it.

[1]: http://gulpjs.com/
[2]: https://github.com/wearefractal/vinyl-fs
[3]: https://github.com/dominictarr/event-stream
[4]: https://github.com/dominictarr/map-stream

[gulp]: img/gulp.png