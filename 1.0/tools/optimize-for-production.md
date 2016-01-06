---
layout: default
type: guide
shortname: Docs
title: Optimize for production
subtitle: Developer guide
---

{% include toc.html %}

Reducing network requests is important for a performant app experience. In the {{site.project_title}} world, [Vulcanize](https://github.com/Polymer/vulcanize) is the name given to a build tool that lets you **concatenate** a set of elements and their HTML imported dependencies into a single file. Vulcanize recursively pulls in all your imports, flattens their dependencies and spits out something that can **reduce the number of network requests** your app makes.

**Note:** for more great info on performance considerations worth keeping in mind when using HTML Imports, see [HTML Imports - #include for the web](http://www.html5rocks.com/en/tutorials/webcomponents/imports/#performance)
{: .alert .alert-info }

## Installation

Install Vulcanize and its dependencies using [NPM](http://npmjs.org):

    npm install -g vulcanize

We recommend installing Vulcanize globally so you can run it from anywhere on the command-line. To install locally, simply omit the `-g` flag.

## Usage

Vulcanize can be used standalone from the command line, or as part of a gulp/grunt build chain.

### Use vulcanize from the command line

Assuming an input HTML file, elements.html, using a number of HTML imports, we can run it through Vulcanize by passing it as an argument as follows:

    vulcanize elements.html

The output will be printed to stdout in the terminal.

To write the output to a file, use the `--output` or `-o` flag:

    vulcanize -o elements.vulcanized.html elements.html

No additional configuration is necessary. elements.vulcanized.html now contains a version of elements.html with all imports inlined and dependencies flattened. Paths to any URLs are automatically adjusted for the new output location, with the exception of those set in JavaScript.

You can pass additional options to vulcanize in the form of flags. For a full list of supported flags, see [the official Vulcanize documentation](https://github.com/Polymer/vulcanize#using-vulcanize-programmatically).

Here’s the same example from above, but this time we’ll tell Vulcanize to strip comments and inline any scripts or CSS.

    vulcanize -o elements.vulcanized.html elements.html --strip-comments --inline-scripts --inline-css

### Use vulcanize with gulp

Although Vulcanize does a great job of flattening imports, you may have an existing build system setup that needs to uglify/minify your code or run your CSS through a preprocessor. This section shows you how to add vulcanize to gulp using  [gulp-vulcanize](https://github.com/sindresorhus/gulp-vulcanize).

*Using Grunt?* You can use the [grunt-vulcanize]() task. It supports a similar set of options [maybe link to README?]
{: .alert .alert-info }

To add vulcanize to our build process:

Install `gulp-vulcanize`

    npm install --save-dev gulp gulp-vulcanize

Require the vulcanize module in your gulpfile and add a task to run it.

    var gulp = require('gulp');
    var vulcanize = require('gulp-vulcanize');

    gulp.task('vulcanize', function() {
      return gulp.src(app/elements/elements.html')
        .pipe(vulcanize())
        .pipe(gulp.dest('dist/elements'));
    });

    gulp.task('default', ['vulcanize']);

This sample assumes your project has a single `elements.html` file that imports your other web component dependencies.
{: .alert .alert-info }

You should now be able to run `gulp` and vulcanize your dependencies.

To configure the task with the same `stripComments`, `inlineScripts`, and `inlineCss` options from above, pass them to the vulcanize task in a configuration object:

    gulp.task('vulcanize', function() {
      return gulp.src(app/elements/elements.html')
        .pipe(vulcanize({
          stripComments: true,
          inlineScripts: true,
          inlineCss: true
        }))
        .pipe(gulp.dest('dist/elements'));
    });

Depending on the structure of your app, it may make sense to break it into a few small vulcanized bundles, instead of inlining everything into one file. To prevent specific imports from being inlined in your bundle, use the `excludes` option and pass it an array of file paths or regexes.

    gulp.task('vulcanize', function() {
      return gulp.src(app/elements/elements.html')
        .pipe(vulcanize({
          excludes: ['elements/x-foo.html']
        }))
        .pipe(gulp.dest('dist/elements'));
    });

This prevents the resources from being _inlined_ in the bundle, however, it leaves link tags in the bundle which attempt to import the resources. If you want to completely remove resources from a bundle, including their link tags, include an additional `stripExcludes` array with the same file paths/regexes.

    gulp.task('vulcanize', function() {
      return gulp.src(app/elements/elements.html')
        .pipe(vulcanize({
          excludes: [‘elements/x-foo.html’],
          stripExcludes: [elements/x-foo.html']
        }))
        .pipe(gulp.dest('dist/elements'));
    });


#### Example

Consider a {{site.project_title}} app composed of four HTML files: index.html, elements/elements.html, elements/x-foo.html, and elements/x-bar.html.

app/index.html:

{% raw %}
    <!doctype html>
    <html>
      <head>
        <script src="bower_components/webcomponentsjs/webcomponents-lite.js"></script>
        <link rel="import" href="elements/elements.html">
      </head>
      <body>
       ...
      </body>
    </html>
{% endraw %}

app/elements/elements.html:

{% raw %}
    <link rel="import" href="x-foo.html">
    <link rel="import" href="x-bar.html">
{% endraw %}

 app/elements/x-foo.html:

{% raw %}
    <link rel="import" href="../bower_components/polymer/polymer.html">
    <dom-module id="x-foo">
      <template>
        <p>Hello from x-foo!</p>
      </template>
      <script>
        Polymer({
          is: 'x-foo'
        });
      </script>
    </dom-module>
{% endraw %}

 app/elements/x-bar.html:

{% raw %}
    <link rel="import" href="../bower_components/polymer/polymer.html">
    <dom-module id="x-bar">
      <template>
        <p>Hello from x-bar!</p>
      </template>
      <script>
        Polymer({
          is: 'x-bar'
        });
      </script>
    </dom-module>
{% endraw %}

Without any concatenation in place, loading this application results in at least 5 network requests. Let's bring that number down. Running Vulcanize on elements/elements.html, and specifying elements.vulcanized.html as the output:

    vulcanize elements.html -o elements.vulcanized.html

This results in a elements.vulcanized.html that looks a little like this:

{% raw %}
    <!-- all the code for polymer.html -->
    <dom-module id="x-foo" assetpath="/">
      <template>
        <p>Hello from x-foo!</p>
      </template>
      <script>
        Polymer({is: 'x-foo'});
      </script>
    </dom-module>
    <dom-module id="x-bar" assetpath="/">
      <template>
        <p>Hello from x-bar!</p>
      </template>
      <script>
        Polymer({is: 'x-bar'});
      </script>
    </dom-module>
{% endraw %}

## Content Security Policy

[Content Security Policy](http://en.wikipedia.org/wiki/Content_Security_Policy) (CSP) is a JavaScript security model that aims to prevent XSS and other attacks. In so doing, it prohibits the use of inline scripts.

To use {{site.project_title}} in a CSP environment that doesn't support inline scripts, you can use the Crisper project. Crisper removes all scripts from the HTML Imports and places their contents into an output JavaScript file. This is useful in amongst other things, using {{site.project_title}} in a Chrome App.

Like Vulcanize, Crisper can be used either from the command line, or as a gulp plugin.

### Command Line

To install Crisper, run the following command:

    npm install -g crisper

Crisper can work directly with the piped output from vulcanize, as shown below:

    vulcanize elements/elements.html --inline-script | crisper --html elements/elements.vulcanized.html --js elements/elements.vulcanized.js

It may seem a little strange to call `vulcanize` with `--inline-script` then pass it through `crisper` to separate out the JavaScript. However, if any of your elements use external scripts, this flag ensures that both inline and external scripts are extracted and concatenated into `elements.vulcanized.js`.
{: .alert .alert-info }

### Gulp

Similarly, you can pipe the output from the Vulcanize task directly to the Crisper task in Gulp.

Run the following command:

    npm install -g gulp-crisper

Then add it to your Gulpfile:

    var gulp = require('gulp');
    var vulcanize = require('gulp-vulcanize');
    var crisper = require('gulp-crisper');

    gulp.task('vulcanize', function() {
      return gulp.src('app/elements/elements.html')
        .pipe(vulcanize())
        .pipe(crisper())
        .pipe(gulp.dest('dist/elements'));
      });

    gulp.task('default', ['vulcanize']);


## FAQ

### Is concatenation a good idea?

This depends on how large your application is. Excessive requests are often far worse than filesize. Hypothetically, let's say you have 20 HTML files/imports of 0.5MB each. Out of which only 2 (1MB) are required on the first page. You might want to vulcanize just those two requests into a critical bundle, then load the others in a separate, deferred bundle using Polymer’s importHref method.

For example:

    Polymer.Base.importHref('/elements/less-important-stuff.html',
      // onsuccess callback
      function() {
        console.log('yay! our app is ready!');
      },
      // onerror callback
      function(err) {
        console.log('uh oh, something failed', err);
      },
      // use `async` on import
    true);

Some of the things that you should think about before combining a large number of imports are when you combine 100 files together, you're eliminating 99 requests. But concatenation does have some drawbacks:

* The single file takes much longer to download, and potentially blocks loading of an important page.

* The browser needs to parse and render additional code that might not be required yet.

The short answer is "don't guess it, test it". There are always trade-offs when it comes to concatenation, but tools like [WebPageTest](http://webpagetest.org) can be useful for determining what your true bottlenecks are.
