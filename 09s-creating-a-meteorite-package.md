---
title: Creating a Meteorite Package
slug: creating-a-meteorite-package
date: 0009/01/02
number: 9.5
points: 10
sidebar: true
photoUrl: http://www.flickr.com/photos/rxb/7779426142/
photoAuthor: Richard
contents: Write a local in-app package.|Write some tests for your package.|Release your package on Atmosphere.
---

We've built a re-usable pattern with our errors work, so why not package it up into a smart package and share it with the rest of the Meteor community?

First we need to create some structure for our package to reside in. We put the package in a directory named `packages/errors/`. This creates a custom package that's automatically used. (You might have noticed that Meteorite installs packages via symlinks in the `packages/` directory).

Second, we'll create `package.js` in that folder, the file that informs Meteor of how the package should be used, and the symbols that it exports.

~~~js
Package.describe({
  summary: "A pattern to display application errors to the user"
});

Package.on_use(function (api, where) {
  api.use(['minimongo', 'mongo-livedata', 'templating'], 'client');

  api.add_files(['errors.js', 'errors_list.html', 'errors_list.js'], 'client');
  
  if (api.export) 
    api.export('Errors');
});
~~~
<%= caption "packages/errors/package.js" %>

Let's add three files to the package. We can pull these files from Microscope without much change except for some proper namespacing and a slightly cleaner API:

~~~js
Errors = {
  // Local (client-only) collection
  collection: new Meteor.Collection(null),
  
  throw: function(message) {
    Errors.collection.insert({message: message, seen: false})
  },
  clearSeen: function() {
    Errors.collection.remove({seen: true});
  }
};

~~~
<%= caption "packages/errors/errors.js" %>

~~~html
<template name="meteorErrors">
  {{#each errors}}
    {{> meteorError}}
  {{/each}}
</template>

<template name="meteorError">
  <div class="alert alert-error">
    <button type="button" class="close" data-dismiss="alert">&times;</button>
    {{message}}
  </div>
</template>
~~~
<%= caption "packages/errors/errors_list.html" %>

~~~js
Template.meteorErrors.helpers({
  errors: function() {
    return Errors.collection.find();
  }
});

Template.meteorError.rendered = function() {
  var error = this.data;
  Meteor.defer(function() {
    Errors.collection.update(error._id, {$set: {seen: true}});
  });
};
~~~
<%= caption "packages/errors/errors_list.js" %>

### Testing the package out with Microscope

We will now test things locally with Microscope to ensure our changed code works. To link the package into our project, we run `meteor add errors`. Then, we need to delete the existing files that have been made redundant by the new package:

~~~bash
$ rm client/helpers/errors.js
$ rm client/views/includes/errors.html
$ rm client/views/includes/errors.js
~~~
<%= caption "removing old files on the bash console" %>

One other thing we need to do is to make some minor updates to use the correct API:

~~~js
Router.before(function() { Errors.clearSeen(); });
~~~
<%= caption "lib/router.js" %>

~~~html
  {{> header}}
  {{> meteorErrors}}
~~~
<%= caption "client/views/application/layout.html" %>

~~~js
Meteor.call('post', post, function(error, id) {
  if (error) {
    // display the error to the user
    Errors.throw(error.reason);

~~~
<%= caption "client/views/posts/post_submit.js" %>

~~~js
Posts.update(currentPostId, {$set: postProperties}, function(error) {
  if (error) {
    // display the error to the user
    Errors.throw(error.reason);
~~~
<%= caption "client/views/posts/post_edit.js" %>

<%= scommit "9-5-1", "Created basic errors package and linked it in." %>

Once these changes have been made, we should get our original pre-package behaviour back.

### Writing tests

The first step in developing a package is testing it against an application, but the next is to write a test suite that properly tests the package's behaviour. Meteor itself comes with Tinytest (a built in package tester), which makes it easy to run such tests and maintain peace of mind when sharing our package with others.

Let's create a test file that uses Tinytest to run some tests against the errors codebase:

~~~js
Tinytest.add("Errors collection works", function(test) {
  test.equal(Errors.collection.find({}).count(), 0);
  
  Errors.throw('A new error!');
  test.equal(Errors.collection.find({}).count(), 1);
  
  Errors.collection.remove({});
});

Tinytest.addAsync("Errors template works", function(test, done) {  
  Errors.throw('A new error!');
  test.equal(Errors.collection.find({seen: false}).count(), 1);
  
  // render the template
  OnscreenDiv(Spark.render(function() {
    return Template.meteorErrors();
  }));
  
  // wait a few milliseconds
  Meteor.setTimeout(function() {
    test.equal(Errors.collection.find({seen: false}).count(), 0);
    test.equal(Errors.collection.find({}).count(), 1);
    Errors.clearSeen();
    
    test.equal(Errors.collection.find({seen: true}).count(), 0);
    done();
  }, 500);
});
~~~
<%= caption "packages/errors/errors_tests.js" %>


In these tests we're checking the basic `Meteor.Errors` functions work, as well as double checking that the `rendered` code in the template is still functioning.

We won't cover the specifics of writing Meteor package tests here (as the API is not yet finalized and highly in flux), but hopefully it's fairly self explanatory how it works.

To tell Meteor how to run the tests in `package.js`, use the following code:

~~~js
Package.on_test(function(api) {
  api.use('errors', 'client');
  api.use(['tinytest', 'test-helpers'], 'client');  
  
  api.add_files('errors_tests.js', 'client');
});
~~~
<%= caption "packages/errors/package.js" %>

<%= scommit "9-5-2", "Added tests to the package." %>

Then we can run the tests with:

~~~bash
$ meteor test-packages errors
~~~
<%= caption "Terminal" %>

<%= screenshot "s7-1", "Passing all tests" %>

### Releasing the package

Now, we want to release the package and make it available to the world. We do this by putting it on Atmosphere.

First, we need to add a `smart.json`, to tell Meteorite and Atmosphere the important details about the package:

~~~json
{
  "name": "errors",
  "description": "A pattern to display application errors to the user",
  "homepage": "https://github.com/tmeasday/meteor-errors",
  "author": "Tom Coleman <tom@thesnail.org>",
  "version": "0.1.0",
  "git": "https://github.com/tmeasday/meteor-errors.git",
  "packages": {
  }
}
~~~
<%= caption "packages/errors/smart.json" %>

<%= scommit "9-5-3", "Added a smart.json" %>

We put in some basic metadata to provide information about the package, including what it does, the git location where we're going to host it, and an initial version number. If our package was relying on other Atmosphere packages, we could also use a `"packages"` section to outline its dependencies.

Once all this is in place, releasing is easy. We'll need to create a git repository, push to a remote git server somewhere, and link to that location in our `smart.json`.

The process for doing this for [GitHub](http://github.com) is to first create a new repository, then follow the standard practice to get the package's code within that repository. Then, we use the `mrt release` command to publish it:

~~~bash
$ git init
$ git add -A
$ git commit -m "Created Errors Package"
$ git remote add origin https://github.com/tmeasday/meteor-errors.git
$ git push origin master
$ mrt release .
Done!
~~~
<%= caption "Terminal (run from within `packages/errors`)" %>

Note: package names have to be unique. If you are following along word-for-word and use the same package name, there will be a conflict and it won't work. In the future though Atmosphere will be name-spaced by author, so you can expect this to change.

Second Note: You'll need to login at http://atmosphere.meteor.com and create a username and password which you'll enter on the command line when you call `mrt release .`.

Now that the package is released, we can now delete it from the project and then add it back in directly using Meteorite:

~~~bash
$ rm -r packages/errors
$ mrt add errors
~~~
<%= caption "Terminal (run from the top level of the app)" %>

<%= scommit "9-5-4", "Removed package from development tree." %>

Now we should see Meteorite download our package for the very first time. Well done!