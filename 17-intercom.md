---
title: Implementing Intercom
slug: intercom
number: 17
extra: true
date: 0017/01/01
points: 100
published: true
photoUrl: http://www.flickr.com/photos/ikewinski/8267236829/
photoAuthor: Mike Lewinski
contents: Implement Intercom in a Meteor app.
---

In this chapter, we'll look at integrating [Intercom](http://intercom.io) into your Meteor app. Intercom bills itself as "a smarter way to do lifecycle marketing, customer development, newsletters, support", and is in effect a CRM (Customer Relationship Manager) for your app.

What this means is that Intercom provides you with a dashboard listing all your users, and lets you message each of them, either automatically or manually. But the best part is that you can pick how your users will actually receive that message: either in their email inbox, or as a notification within your web app the next time they log on.

<%= screenshot "15-1", "The Intercom dashboard." %>

The reason why Intercom is a great tool to integrate with any web app, is that even if you don't spring for the $50/month messaging plan, you can still benefit from the user dashboard for free. This gives you a great overview of recently connected users, along with meta-data such as their location or subscription date, and even custom data from your app!

<% note do %>

### The Intercom Package

The code in this chapter is a simplified version of the [Intercom package](https://github.com/percolatestudio/meteor-intercom) created by Tom Coleman.

Compared with what we'll be covering here, the package handles a few other edge cases, like users logging out or re-sending data.

<% end %>

### Signing Up

As soon as you click "Start Using Intercom", Intercom will provide you with its code snippet without even asking for an email or password. 

This code snippet is made of two distinct `<script>` tags: an app-specific block where you'll configure Intercom's settings, and a second generic part that will actually load in the Intercom script. 

A lot of the time, Meteor makes traditionally complex operations very easy to do. But the other side of the coin is that it can also make seemingly simple things more complicated!

Especially when it comes to integrating any kind of third-party code snippets, you'll often find that you can't just blindly follow the instructions provided, and that the process will be a little more involved. So although Intercom tells us to paste this code before the `</body>` tag, we'll disregard that and instead create a local package for it. 

### Creating a Package

First, create a new `/packages` root directory, and inside it an `intercom` sub-directory. 

Then copy-paste your Intercom code snippet into a new `intercom.js` file inside that:

We'll get rid of the configuration block, since we're going to set all those properties from within our app, and we'll also get rid of the `<script>` tags since we're already inside a JavaScript file. 

~~~js
(function(){var w=window;var ic=w.Intercom;if(typeof ic==="function"){ic('reattach_activator');ic('update',intercomSettings);}else{var d=document;var i=function(){i.c(arguments)};i.q=[];i.c=function(args){i.q.push(args)};w.Intercom=i;function l(){var s=d.createElement('script');s.type='text/javascript';s.async=true;s.src='https://static.intercomcdn.com/intercom.v1.js';var x=d.getElementsByTagName('script')[0];x.parentNode.insertBefore(s,x);}if(w.attachEvent){w.attachEvent('onload',l);}else{w.addEventListener('load',l,false);}};})()
~~~
<%= caption "packages/intercom/intercom_loader.js" %>

Now, we need to tell Meteor when and how to load our brand new package. First, create a new `package.js` file in the same directory and add a couple lines of code:

~~~js
Package.describe({
  summary: "Intercom package"
});

Package.on_use(function (api) {
  api.add_files('intercom_loader.js', 'client');
});
~~~
<%= caption "packages/intercom/package.js" %>

The [Package API](https://coderwall.com/p/ork35q)'s `add_files()` follows a simple pattern where you choose in which environment to add one or more files. It also takes arrays as arguments, so you can also write something like `api.add_files(['file1.js', 'file2.js'], ['client', 'server')`.

That's it for our (very simple) package. You can now let Meteor know we'd like to use it by adding it to your app with:

~~~bash
meteor add intercom
~~~

Now let's finish our integration by implementing the settings block. At this stage we just want to make sure that everything is working so we won't even bother with custom data yet. We'll simply take the dummy code provided by Intercom and stick it in our `main.js` file:

~~~js
window.intercomSettings = {
  // TODO: The current logged in user's email address.
  email: "john.doe@example.com",
  // TODO: The current logged in user's sign-up date as a Unix timestamp.
  created_at: 1234567890,
  app_id: "9b593f5995da6d2bd8f418754c0411d55201258a"
};
~~~
<%= caption "client/main.js" %>

<%= screenshot "15-2", "Intercom confirmation message." %>

If you did everything right, you should get a prompt in the console suggesting you get rid of the dummy data, and a message over on the Intercom site confirming that Intercom is now receiving your data. You will then be taken to a sign-up screen where you can finish the sign-up process. 

<%= commit "17-1", "Added Intercom code snippet." %>

### Sending Custom Data

Another reason why we picked Intercom as our first integration example is that they have a great installation wizard that takes you through each step of the process. So let's get started by bringing up the Intercom dashboard and clicking on the red progress bar in the top right corner. 

The next step is "send more data about your users", so that's what we'll do. First, we're still sending out our dummy user email and sign-up date so let's take care of that. That's where we hit our first snag: we're not actually collecting either one!

Collecting emails is simple enough. We'll change our Meteor app's sign-up method from `USERNAME_ONLY` to `USERNAME_AND_EMAIL`:

~~~js
Accounts.ui.config({
  passwordSignupFields: 'USERNAME_AND_EMAIL'
});
~~~
<%= caption "client/helpers/config.js" %>

To avoid any inconsistencies with previously created user profiles (which won't have an email), now would be a good time to run `meteor reset` to purge the user collection. Once you've done that, just create a new account filling in both a username and an email. 

If you type `Meteor.user()` in the browser console, you'll be able to drill down into the user object and see that a user's main email is accessible at `Meteor.user().emails[0].address`. But what about a user account's creation date?

Although that date is stored in the database (feel free to check with the Mongo console to make sure), that date is not being published by default by Meteor Accounts. So what this means is that we'll need to manually set up a publication of the `Meteor.users` collection to tell Meteor to publish this extra property.

First, set up this publication in `publications.js`:

~~~js
Meteor.publish('currentUser', function() {
  return Meteor.users.find(this.userId, {fields: {createdAt: 1}});
});
~~~
<%= caption "server/publications.js" %>

Then, we'll subscribe to it all the time in our router::

~~~js
Router.configure({
  layoutTemplate: 'layout',
  loadingTemplate: 'loading',
  waitOn: function() { 
    return [
      Meteor.subscribe('currentUser'),
      Meteor.subscribe('notifications')
    ]
  }
});
~~~
<%= highlight "6" %>
<%= caption "lib/router.js" %>

Note that we only need to publish that extra field (and not the whole user object). Meteor is then smart enough to add that new property to the user object that is already being published by default.

We can then go back to our Intercom snippet and update it accordingly. Intercom takes a 10-digit UNIX timestamp, while Meteor provides a 13-digit one, so we'll divide by 1000 and round up the result. And to test out Intercom's custom data capabilities, we'll also throw in a user's username as a custom user property.

We'll also wrap the whole thing into an `if` statement to check that the is logged in before attempting to access their info, and an `autorun` block to make sure we check again if the user's logged-in-ness changes (and that `autorun` block will also come in handy later on). 

Finally, we'll use Intercom's `boot` method to manually trigger Intercom's initialization. Most third-party services offer such methods, as their regular code snippets generally don't play nice with single-page apps. But you sometimes have to dig a little deeper in [their documentation](http://docs.intercom.io/#IntercomJS) to find them.

~~~js
Deps.autorun(function(){
  if (Meteor.user() && !Meteor.loggingIn()) {
    var intercomSettings = {
      email: Meteor.user().emails[0].address,
      created_at: Math.round(Meteor.user().createdAt/1000),
      user_name: Meteor.user().username,
      app_id: "9b593f5995da6d2bd8f418754c0411d55201258a"
    };
    Intercom('boot', intercomSettings);
  }
});
~~~
<%= caption "client/main.js" %>

<%= commit "17-2", "Sending custom user data." %>

### Enabling Secure Mode

To prevent just anybody from using your app ID to make fake requests and impersonate other users, Intercom's secure mode uses a secret server-side only key to hash each email. 

Of course our Meteor code runs in the client and so running 'secret server-side' code is out of the question. So we'll need to ensure that we attach the hash to the user when they are first created.

First, delete all the existing users in the database with a `meteor reset`.

First, we'll add a simple `IntercomHash` function to our package, that takes a user and uses a SHA hash to create the hash, as per intercom's specs:

~~~js
var crypto = Npm.require('crypto');

IntercomHash = function(user, secret) {
  var secret = new Buffer(secret, 'utf8')
  return crypto.createHmac('sha256', secret)
    .update(user._id).digest('hex');
}
~~~
<%= caption "packages/intercom/intercom_server.js" %>

Note the use of `Npm.require`, Meteor's method of using NPM packages (we don't need to add a `Npm.depends` in our `package.js` here because `crypto` is a built in node package - it doesn't actually come from NPM at all!).

Now we just need to add that function to our intercom package, and export it so we can use it in the server `IntercomHash`.

~~~js
Package.describe({
  summary: "Intercom package"
});

Package.on_use(function (api) {
  api.add_files('intercom_loader.js', 'client');
  api.add_files('intercom_server.js', 'server');
  
  api.export('IntercomHash', 'server');
});
~~~
<%= highlight "7,9" %>
<%= caption "packages/intercom/package.js" %>

Armed with our super-secret cryptographic hashing algorithm, we can now write our user creation hook. What we are aiming to do is ensure that users have a secret `intercomHash` field that we can send to Intercom. 

First, create a new `accounts.js` file inside the `server` directory. Don't forget to replace `123456789` with the secure string provided by Intercom!

~~~js
Accounts.onCreateUser(function(options, user) {
  user.intercomHash = IntercomHash(user, '12345678');

  if (options.profile)
    user.profile = options.profile;

  return user;
});
~~~
<%= caption "server/accounts.js" %>

The `onCreateUser` function is a built-in Meteor function that allows you to run custom code to create users.

Now, we make sure we are publishing that hash (only to the logged in user!) as well:

~~~js
//...

Meteor.publish('currentUser', function() {
  return Meteor.users.find(this.userId, {fields: {createdAt: 1, intercomHash: 1}});
});
~~~
<%= caption "server/publications.js" %>
<%= highlight "3~5" %>

Finally, we send it up to intercom:

~~~js
Deps.autorun(function(){
  if (Meteor.user() && !Meteor.loggingIn()) {
    var intercomSettings = {
      email: Meteor.user().emails[0].address,
      created_at: Math.round(Meteor.user().createdAt/1000),
      user_name: Meteor.user().username,
      user_id: Meteor.user()._id,
      user_hash: Meteor.user().intercomHash,
      app_id: "k20iexvc"
    };
    Intercom('boot', intercomSettings);
  }
});
~~~
<%= caption "client/main.js" %>
<%= highlight "7~8" %>

<%= commit "17-3", "Enabling secure mode." %>

<% note do %>
### Where did my users go?!?

You might be concerned that we deleted all the existing users as the first step of the process. If this was an app that was out there in production, we of course couldn't do that!

The correct way to deal with this is with a *data migration* -- we run through the database and add the hash to all the existing users. In fact, we'll show you exactly how to do that in the next sidebar.

<% end %>

### Installing the Inbox

Only one step left! If we want our users to be able to receive and reply to messages, we'll need to provide them with a user interface element that brings up their Intercom inbox. 

Intercom gives us the choice of a default inbox button that lives in the bottom right corner of your app, or a custom trigger that you can apply to an element of your choice. We want to keep our UI clean, so let's go with the custom element. 

First, let's add a "Support" link to our global navigation:

~~~html
<template name="header">
  <header class="navbar">
    <div class="navbar-inner">
      <a class="btn btn-navbar" data-toggle="collapse" data-target=".nav-collapse">
        <span class="icon-bar"></span>
        <span class="icon-bar"></span>
        <span class="icon-bar"></span>
      </a>
      <a class="brand" href="{{postsListPath}}">Microscope</a>
      <div class="nav-collapse collapse">
        <ul class="nav">
          <li class="{{activeRouteClass 'home' 'newPosts'}}">
            <a href="{{homePath}}">New</a>
          </li>
          <li class="{{activeRouteClass  'bestPosts'}}">
            <a href="{{bestPostsPath}}">Best</a>
          </li>
          {{#if currentUser}}
            <li>
              <a href="{{postSubmitPath}}">Submit Post</a>
            </li>
            <li class="dropdown">
              {{> notifications}}
            </li>
            <li>
              <a id="Intercom" href="mailto:k20iexvc@incoming.intercom.io">Support</a>
            </li>
          {{/if}}
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
<%= highlight "25~27" %>

Then all we need to do is tell the Intercom settings code about our new `#Intercom` element:

~~~js
var intercomSettings = {
  email: Meteor.user().emails[0].address,
  created_at: Math.round(Meteor.user().createdAt/1000),
  user_name: Meteor.user().username,
  user_id: Meteor.user()._id,
  user_hash: Meteor.user().intercomHash,
  widget: {
    activator: '#Intercom',
    use_counter: true
  },
  app_id: "k20iexvc"
};
~~~
<%= caption "client/main.js" %>
<%= highlight "7~10" %>

<%= screenshot "15-1", "The Intercom inbox." %>

If we stopped here, you'd soon notice a weird behavior. Sometimes clicking the "Support" link would bring up the Intercom inbox like it's supposed to, but at other seemingly random times it would default back to trigger the `mailto:` link! What's going on?

When Intercom first boots up, it looks for an `#Intercom` element and tells its `click` event to open that inbox. The only problem is, as soon as the containing template gets redrawn (which tends to happen a lot with Meteor apps), that element will be regenerated fresh without any memory of Intercom's click handler. This is another common gotcha when dealing with third-party code. Since Meteor code is usually reactive, it's easy to forget that other JavaScript code rarely is. 

To prevent this all we need to do is wrap our `Support` link in a `{{#constant}}` block helper, which will tell Meteor not to redraw that specific template bit:

~~~html
{{#if currentUser}}
  <li class="{{activeRouteClass 'postSubmit'}}">
    <a href="{{postSubmitPath}}">Submit Post</a>
  </li>
  <li class="dropdown">
    {{> notifications}}
  </li>
  <li>
    {{#constant}}
      <a id="Intercom" href="mailto:k20iexvc@incoming.intercom.io">Support</a>
    {{/constant}}
  </li>
{{/if}}
~~~
<%= caption "client/views/includes/header.html" %>

<%= commit "17-4", "Displaying the inbox link." %>

### Wrapping Up

You can now complete your sign-up process by adding a photo, entering your credit card information (there's a 14-day free trial), and sending your users a few messages.

It probably would also make sense to use the <a href="https://github.com/percolatestudio/meteor-intercom">community Intercom package</a>, which deals with a few more edge cases, like users logging out, and reactive intercom data.

In any case, whether you end up using Intercom or not, we're willing to bet that the patterns introduced in this chapter will prove themselves to be quite useful!

