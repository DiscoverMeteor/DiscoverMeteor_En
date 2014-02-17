---
title: Creating Posts
slug: creating-posts
date: 0007/01/01
number: 7
points: 10
photoUrl: http://www.flickr.com/photos/markezell/9688179085
photoAuthor: Mark Ezell
contents: Learn how to submit a post client-side.|Implement a simple security check.|Restrict access to the post submit form.|Learn to use a server-side Method for added security.
---

We've seen how easy it is to create posts via the console, using the `Posts.insert` database call, but we can't expect our users to open the console to create a new post. 

Eventually, we'll need to build some kind of user interface to let our users post new stories to our app. 

### Building The New Post Page

We begin by defining a route for our new page:

~~~js
Router.configure({
  layoutTemplate: 'layout',
  loadingTemplate: 'loading',
  waitOn: function() { return Meteor.subscribe('posts'); }
});

Router.map(function() {
  this.route('postsList', {path: '/'});
  
  this.route('postPage', {
    path: '/posts/:_id',
    data: function() { return Posts.findOne(this.params._id); }
  });
  
  this.route('postSubmit', {
    path: '/submit'
  });
});
~~~
<%= caption "lib/router.js" %>
<%= highlight "13~15" %>

We're using the router's `data` function to set the `postPage` template's data context. Remember that whatever we put into the data context will be available as `this` inside the template helpers. 

### Adding A Link To The Header

With that route defined, we can now add a link to our submit page in our header:

~~~html
<template name="header">
  <header class="navbar">
    <div class="navbar-inner">
      <a class="btn btn-navbar" data-toggle="collapse" data-target=".nav-collapse">
        <span class="icon-bar"></span>
        <span class="icon-bar"></span>
        <span class="icon-bar"></span>
      </a>
      <a class="brand" href="{{pathFor 'postsList'}}">Microscope</a>
      <div class="nav-collapse collapse">
        <ul class="nav">
          <li><a href="{{pathFor 'postSubmit'}}">New</a></li>
        </ul>
        <ul class="nav pull-right">
          <li>{{loginButtons}}</li>
        </ul>
      </div>
    </div>
  </header>
</template>
~~~
<%= caption "client/views/includes/header.html" %>
<%= highlight "11~16" %>

Setting up our route means that if a user browses to the `/submit` URL, Meteor will display the `postSubmit` template. So let's write that template:

~~~html
<template name="postSubmit">
  <form class="main">
    <div class="control-group">
        <label class="control-label" for="url">URL</label>
        <div class="controls">
            <input name="url" type="text" value="" placeholder="Your URL"/>
        </div>
    </div>

    <div class="control-group">
        <label class="control-label" for="title">Title</label>
        <div class="controls">
            <input name="title" type="text" value="" placeholder="Name your post"/>
        </div>
    </div>

    <div class="control-group">
        <label class="control-label" for="message">Message</label>
        <div class="controls">
            <textarea name="message" type="text" value=""/>
        </div>
    </div> 

    <div class="control-group">
        <div class="controls">
            <input type="submit" value="Submit" class="btn btn-primary"/>
        </div>
    </div>
  </form>
</template>

~~~
<%= caption "client/views/posts/post_submit.html" %>

Note: that’s a lot of markup, but it simply comes from using Twitter Bootstrap. While only the form elements are essential, the extra markup will help  make our app look a little bit nicer. It should now look similar to this:

<%= screenshot "7-1", "The post submit form" %>

This is a simple form. We don't need to worry about an action for it, as we'll be intercepting submit events on the form and updating data via JavaScript. (It doesn't make sense to provide a non-JS fallback when you consider that a Meteor app is completely non-functional with JavaScript disabled).

### Creating Posts

Let's bind an event handler to the form `submit` event. It's best to use the `submit` event (rather than say a `click` event on the button), as that will cover all possible ways of submitting (such as hitting enter in URL field for instance).

~~~js
Template.postSubmit.events({
  'submit form': function(e) {
    e.preventDefault();
    
    var post = {
      url: $(e.target).find('[name=url]').val(),
      title: $(e.target).find('[name=title]').val(),
      message: $(e.target).find('[name=message]').val()
    }
    
    post._id = Posts.insert(post);
    Router.go('postPage', post);
  }
});
~~~
<%= caption "client/views/posts/post_submit.js" %>

<%= commit "7-1", "Added a submit post page and linked to it in the header." %>

This function uses [jQuery](http://jquery.com) to parse out the values of our various form fields, and populate a new post object from the results. We need to ensure we `preventDefault` on the `event` argument to our handler to make sure the browser doesn't go ahead and try to submit the form. 

Finally, we can route to our new post's page. The `insert()` function on a collection returns the generated `id` for the object that has been inserted into the database, which the Router's `go()` function will use to construct a URL for us to browse to.

The net result is the user hits submit, a post is created, and the user is instantly taken to the discussion page for that new post.

### Adding Some Security

Creating posts is all very well, but we don't want to let any random visitor do it: we want them to have to be logged in to do so. Of course, we can start by hiding the new post form from logged out users. Still, a user could conceivably create a post in the browser console without being logged in, and we can't have that.

Thankfully data security is baked right into Meteor collections; it's just that it's turned off by default when you create a new project. This enables you to get started easily and start building out your app while leaving the boring stuff for later.

Our app no longer needs these training wheels, so let's take them off! We'll remove the `insecure` package:

~~~bash
$ meteor remove insecure
~~~
<%= caption "Terminal" %>

After doing so, you'll notice that the post form no longer works. This is because without the `insecure` package, client-side inserts into the posts collection _are no longer allowed_. We need to either give some explicit rules telling Meteor when it's OK for a client to insert posts, or else do our post insertions server-side.

### Allowing Post Inserts

To begin with, we'll show how to allow client-side post inserts in order to get our form working again. As it turns out, we'll eventually settle on a different technique, but for now, the following will get things working again easily enough:

~~~js
Posts = new Meteor.Collection('posts');

Posts.allow({
  insert: function(userId, doc) {
    // only allow posting if you are logged in
    return !! userId;
  }
});
~~~
<%= caption "collections/posts.js" %>
<%= highlight "3~8" %>

<%= commit "7-2", "Removed insecure, and allowed certain writes to posts." %>

We call `Posts.allow`, which tells Meteor "this is a set of circumstances under which clients are allowed to do things to the `Posts` collection". In this case, we are saying "clients are allowed to insert posts as long as they have a `userId`". 

The `userId` of the user doing the modification is passed to the `allow` and `deny` calls (or returns `null` if no user is logged in), which is almost always useful. And as user accounts are tied into the core of Meteor, we can rely on `userId` always being correct. 

We've managed to ensure that you need to be logged in to create a post. Try logging out and creating a post; you should see this in your console:

<%= screenshot "7-2", "Insert failed: Access denied " %>

However, we still have to deal with a couple of issues:

 - Logged out users can still reach the create post form.
 - The post is not tied to the user in any way (and there's no code on the server to enforce this).
 - Multiple posts can be created that point to the same URL.

Let's fix these problems.

### Securing Access To The New Post Form

Let's start by preventing logged out users from seeing the post submit form. We'll do that at the router level, by defining a *route hook*. 

A hook intercepts the routing process and potentially changes the action that the router takes. You can think of it as a security guard that checks your credentials before letting you in (or turning you away).

What we need to do is check if the user is logged in, and if they're not render the `accessDenied` template instead of the expected `postSubmit` template (we then stop the router from doing anything else). So let's modify router.js like so:

~~~js
Router.configure({
  layoutTemplate: 'layout'
});

Router.map(function() {
  this.route('postsList', {path: '/'});
  
  this.route('postPage', {
    path: '/posts/:_id',
    data: function() { return Posts.findOne(this.params._id); }
  });
  
  this.route('postSubmit', {
    path: '/submit'
  });
});

var requireLogin = function() {
  if (! Meteor.user()) {
    this.render('accessDenied');
    this.stop();
  }
}

Router.before(requireLogin, {only: 'postSubmit'});
~~~
<%= caption "lib/router.js" %>
<%= highlight "18~25" %>

We also create the template for the access denied page:

~~~html
<template name="accessDenied">
  <div class="alert alert-error">You can't get here! Please log in.</div>
</template>
~~~
<%= caption "client/views/includes/access_denied.html" %>

<%= commit "7-3", "Denied access to new posts page when not logged in." %>

If you now head to http://localhost:3000/submit/ without being logged in, you should see this:

<%= screenshot "7-3", "The access denied template" %>

The nice thing about routing hooks is that they are _reactive_. This means we can be declarative and we don't need to think about callbacks, or similar, when the user logs in. When the log-in state of the user changes, the Router's page template instantly changes from `accessDenied` to `postSubmit` without us having to write any explicit code to handle it.

Log in, then try refreshing the page. You might sometimes see the access denied template flash up for a brief moment before the post submission page appears. The reason for this is that Meteor begins rendering templates as soon as possible, before it has talked to the server and checked if the user currently (stored in the browser's local storage) even exists.

To avoid this problem (which is a common class of problem that you'll see more of as you deal with the intricacies of latency between client and server), we'll just display a loading screen for the brief moment that we are waiting to see if the user has access or not. 

After all at this stage we don't know if the user has the correct log-in credentials, and we can't show either the `accessDenied` or the `postSubmit` template until we do.

So we modify our hook to use our loading template whilst `Meteor.loggingIn()` is true:

~~~js
Router.map(function() {
  this.route('postsList', {path: '/'});
  
  this.route('postPage', {
    path: '/posts/:_id',
    data: function() { return Posts.findOne(this.params._id); }
  });
  
  this.route('postSubmit', {
    path: '/submit'
  });
});

var requireLogin = function() {
  if (! Meteor.user()) {
    if (Meteor.loggingIn())
      this.render(this.loadingTemplate);
    else
      this.render('accessDenied');
    
    this.stop();
  }
}

Router.before(requireLogin, {only: 'postSubmit'});
~~~
<%= caption "lib/router.js" %>
<%= highlight "16~19" %>

<%= commit "7-4", "Show a loading screen while waiting to login." %>


### Hiding the Link

The easiest way to prevent users from trying to reach this page by mistake when they are logged out is to hide the link from them. We can do this pretty easily:

~~~html
<ul class="nav">
  {{#if currentUser}}<li><a href="{{pathFor 'postSubmit'}}">Submit Post</a></li>{{/if}}
</ul>
~~~
<%= caption "client/views/includes/header.html" %>

<%= commit "7-5", "Only show submit post link if logged in." %>

The `currentUser` helper is provided to us by the `accounts` package and is the handlebars equivalent of `Meteor.user()`. Since it's reactive, the link will appear or disappear as you log in and out of the app.

### Meteor Method: Better Abstraction and Security

We've managed to secure access to the new post page for logged out users, and deny such users from creating posts even if they cheat and use the console. Yet there are still a few more things we need to take care of:

- Timestamping the posts.
- Ensuring that the same URL can't be posted more than once.
- Adding details about the post author (ID, username, etc.).

You may be thinking we can do all of that in our `submit` event handler. Realistically, however, we would quickly run into a range of problems.

- For the timestamp, we'd have to rely on the user's computer's time being correct, which is not always going to be the case.
- Clients won't know about _all_ of the URLs ever posted to the site. They'll only know about the posts that they can currently see (we'll see how exactly this works later), so there's no way to enforce URL uniqueness client-side.
- Finally, although we _could_ add the user details client-side, we wouldn't be enforcing its accuracy, which could open our app up to exploitation by people using the browser console.

For all these reasons, it's better to keep our event handlers simple and, if we are doing more than the most basic inserts or updates to collections, use a **Method**.

A Meteor Method is a server-side function that is called client-side. We aren't totally unfamiliar with them -- in fact, behind the scenes, the `Collection`'s `insert`, `update` and `remove` functions are all Methods. Let's see how to create our own.

Let's go back to `post_submit.js`. Rather than inserting directly into the `Posts` collection, we'll call a Method named `post`:

~~~js
Template.postSubmit.events({
  'submit form': function(e) {
    e.preventDefault();
    
    var post = {
      url: $(e.target).find('[name=url]').val(),
      title: $(e.target).find('[name=title]').val(),
      message: $(e.target).find('[name=message]').val()
    }
    
    Meteor.call('post', post, function(error, id) {
      if (error)
        return alert(error.reason);
        
      Router.go('postPage', {_id: id});
    });
  }
});
~~~
<%= caption "client/views/posts/post_submit.js" %>


The `Meteor.call` function calls a Method named by its first argument. You can provide arguments to the call (in this case, the `post` object we constructed from the form), and finally attach a callback, which will execute when the server-side Method is done. Here we simply alert the user if there's a problem, or redirect the user to the freshly created post's discussion page if not.

We then define the Method in our `collections/posts.js` file. We'll remove the `allow()` block from `posts.js` since Meteor Methods bypass them anyway. Remember that Methods are executed on the server, so Meteor assumes they can be trusted. 

~~~js
Posts = new Meteor.Collection('posts');

Meteor.methods({
  post: function(postAttributes) {
    var user = Meteor.user(),
      postWithSameLink = Posts.findOne({url: postAttributes.url});
    
    // ensure the user is logged in
    if (!user)
      throw new Meteor.Error(401, "You need to login to post new stories");
    
    // ensure the post has a title
    if (!postAttributes.title)
      throw new Meteor.Error(422, 'Please fill in a headline');
    
    // check that there are no previous posts with the same link
    if (postAttributes.url && postWithSameLink) {
      throw new Meteor.Error(302, 
        'This link has already been posted', 
        postWithSameLink._id);
    }
    
    // pick out the whitelisted keys
    var post = _.extend(_.pick(postAttributes, 'url', 'title', 'message'), {
      userId: user._id, 
      author: user.username, 
      submitted: new Date().getTime()
    });
    
    var postId = Posts.insert(post);
    
    return postId;
  }
});
~~~
<%= caption "collections/posts.js" %>

<%= commit "7-6", "Use a method to submit the post." %>

This Method is a little complicated, but hopefully you can follow along.

First, we define our `user` variable and check if a post with the same link already exists. Then, we check to see that the user is logged in, throwing an error (which will eventually be `alert`-ed by the browser) if not. We also do some simple validation of the post object to make sure that our posts have titles. 

Next, if there's another post with the same URL, we throw a `302` error (which means redirect) telling the user that they should just go and look at that previously created post. 

Meteor's `Error` class takes three arguments. The first one (`error`) will be the `302` numeric code, the second one (`reason`) is a short human-readable explanation of the error, and the last one (`details`) can be any useful additional information.

In our case, we'll use this third argument to pass the ID of the post that we just found. Spoiler alert: we'll use this later on to redirect the user to the pre-existing post.

If all those checks pass, we grab the fields that we want to insert (to ensure a user calling this Method in browser console can't put spurious data into our database), and include some information about the submitting user -- as well as the current time -- into  the post. 

Finally, we insert the post, and return the new post's `id` to the user.

### Sorting Posts

Now that we have a submitted date on all our posts, it makes sense to ensure that they are sorted using this attribute. To do so, we can just use Mongo's `sort` operator, which expects an object consisting of the keys to sort by, and a sign indicating whether they are ascending or descending.

~~~js
Template.postsList.helpers({
  posts: function() {
    return Posts.find({}, {sort: {submitted: -1}});
  }
});
~~~
<%= caption "client/views/posts/posts_list.js" %>
<%= highlight "3" %>

<%= commit "7-7", "Sort posts by submitted timestamp." %>

It took a bit of work, but we finally have a user interface to let users securely enter content in our app! 

But any app that lets users create content also needs to give them a way to edit or delete it. That's what the Editing Posts chapter will be all about.