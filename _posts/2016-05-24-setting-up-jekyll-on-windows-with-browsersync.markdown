---
layout: post
title:  "Setting up Jekyll on Windows with BrowserSync"
date:   2016-05-23 10:06:18 +0100
summary:    Straight to the point - you need Jekyll for all your static site needs. I've written a quick guide to get up and running fast.
---

I assume you have [Chocolately Nuget][choco] installed on your machine.

From an administrative command prompt run the following:

~~~shell
choco install nodejs
choco install ruby -version 2.2.4 
choco install ruby2.devkit
~~~

At time of writing ruby version 2.2.4 works, later versions do not; otherwise bundle install fails later on.

Next, edit `C:\tools\DevKit2\dk.rb` and change the REG_KEYS to look like the following snippet:

~~~shell
REG_KEYS = [
    'Software\RubyInstaller\MRI',
    'Software\RubyInstaller\Rubinius',
    'Software\Wow6432Node\RubyInstaller\MRI'
]
~~~

Now relaunch the administrative prompt, goto `C:\tools\DevKit2` and run:

~~~shell
run ruby dk.rb init
run ruby dk.rb install
~~~

Now relaunch the administrative prompt and run:

~~~shell
gem install bundler
gem install jekyll
~~~

Navigate to your newly minted git repository and run:

~~~shell
jekyll new .
~~~

Then make a `GemFile` with no extension with the following content:

~~~
source 'https://rubygems.org'
gem 'github-pages'
~~~

Now run the following:

~~~shell
bundle install
gem update
~~~

Now lets try compiling using the Jekyll gem:

~~~shell
bundle exec jekyll serve
~~~

Jekyll should compile the site without any errors.  If you get any errors, [go here][google]

Lets now try and add livereload to the project, shamelessly cribbed from [here][nvbn]

Create a `gulpfile.js` with the following content:

~~~js
var gulp = require('gulp');
var shell = require('gulp-shell');
var browserSync = require('browser-sync').create();

// Task for building blog when something changed:
gulp.task('build', shell.task(['bundle exec jekyll build --watch']));
// Or if you don't use bundle:
// gulp.task('build', shell.task(['jekyll build --watch']));

// Task for serving blog with Browsersync
gulp.task('serve', function () {
    browserSync.init({server: {baseDir: '_site/'}});
    // Reloads page when some of the already built files changed:
    gulp.watch('_site/**/*.*').on('change', browserSync.reload);
});

gulp.task('default', ['build', 'serve']);
~~~

Now run the following from your repository (use any defaults when asked)

~~~shell
npm init
npm install -g gulp
npm install --save-dev gulp-shell lodash gulp browser-sync
~~~

Now edit `_config.yml` and add the following statement to the bottom to prevent any watch issues:

~~~
exclude: [node_modules, gulpfile.js]
~~~

Lastly, run `gulp`

~~~
gulp
~~~

[nvbn]: https://nvbn.github.io/2015/06/19/jekyll-browsersync/
[google]:http://lmgtfy.com/
[choco]: https://chocolatey.org/
