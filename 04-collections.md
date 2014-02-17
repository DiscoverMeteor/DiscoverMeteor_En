---
title: Collections
slug: collections
date: 0004/01/01
number: 4
points: 10
photoUrl: http://www.flickr.com/photos/73449134@N04/8270793784/
photoAuthor: Mike Lewinski
contents: Learn about Meteor's core feature, realtime collections.|Understand how Meteor's data synchronization works.|Integrate collections with our templates.|Turn our basic prototype into a functioning realtime application!
---

In chapter one, we spoke about the core feature of Meteor, the automatic synchronisation of data between client and server. 

In this chapter, we'll take a closer look at how that works, and observe the operation of the key piece of technology that enables this, the Meteor Collection.

We are building a social news app, so the first thing we want to do is make a list of links that people have posted. We'll call each of these items a "post."

Naturally, we need to store these posts somewhere. Meteor comes bundled with a Mongo database which runs on your server and is your *persistent* data store.

So, although a user's browser may contain some kind of state (for instance which page they are on, or the comment they are currently typing), the server, and specifically Mongo, contains the permanent, canonical data source. By *canonical*, we mean that it is the same for all users: each user might be on a different page, but the master list of posts is the same for all.

This data is stored in Meteor in the **Collection**. A collection is a special data structure that, through publications and subscriptions, takes care of synchronising real-time data to and from each connected user's browser and into the Mongo database. Let's see how.

We want our posts to be permanent and shared between users, so we'll start by creating a collection called `Posts` to store them in. If you haven't done so already create a `collections/` folder at the root of your app, and then a `posts.js` file inside it. Then add:

~~~js
Posts = new Meteor.Collection('posts');
~~~
<%= caption "collections/posts.js" %>

<%= commit "4-1", "Added a posts collection" %>

Code inside folders that are not `client/` or `server/` will run in *both* contexts. So the `Posts` collection is available to both client and server. However, what the collection does in each environment is very different.

<% note do %>

### To Var Or Not To Var?

In Meteor, the `var` keyword limits the scope of an object to the current file. We want to make the `Posts` collection available to our whole app, which is why we're omitting that keyword here. 

<% end %>

On the server, the collection has the job of talking to the Mongo database, and reading and writing any changes. In this sense, it can be compared to a standard database library. On the client however, the collection is a *secure* copy of a *subset* of the real, canonical collection. The client-side collection is constantly and (mostly) transparently kept up to date with that subset in real-time.

<% note do %>

### Console vs Console vs Console

In this chapter, we'll start making use of the **browser console**, which is not to be confused with the **terminal** or the **Mongo shell**. Here's a quick primer on each of them.

#### Terminal

<%= screenshot "terminal" %>

- Called from your operating system.
- **Server-side** `console.log()` calls output here. 
- Prompt: `$`.
- Also known as: Shell, Bash

#### Browser Console

<%= screenshot "browser-console" %>

- Called from inside the browser, executes JavaScript code.
- **Client-side** `console.log()` calls output here. 
- Prompt: `❯`.
- Also known as: JavaScript Console, DevTools Console

#### Mongo Shell

<%= screenshot "mongo-shell" %>

- Called from the Terminal with `meteor mongo` or `mrt mongo`.
- Gives you direct access to your app's database. 
- Prompt: `>`.
- Also know as: Mongo Console

Note that in each case, you're not supposed to type the prompt character (`$`, `❯`, or `>`) as part of the command. And you can assume that any line *not* beginning with the prompt is the output of the preceding command. 

<% end %>

### Server-Side Collections

On the server, the collection acts as an API into your Mongo database. In your server-side code, this allows you to write Mongo commands like `Posts.insert()` or `Posts.update()`, and they will make changes to the `posts` collection stored inside Mongo.

To look inside the Mongo database, open up a second terminal window (while `meteor` is still running in your first), and go to your app's directory. Then, run the command `meteor mongo` to initiate a Mongo shell, into which you can type standard Mongo commands (and as usual, you can quit it with the `ctrl+c` keyboard shortcut). For example, let's insert a new post:

~~~bash
> db.posts.insert({title: "A new post"});

> db.posts.find();
{ "_id": ObjectId(".."), "title" : "A new post"};
~~~
<%= caption "The Mongo Shell" %>

<% note do %>

### Mongo on Meteor.com

Note that when hosting your app on *.meteor.com, you can also access your deployed app's Mongo shell with `meteor mongo myApp`. 

And while we're at it, you can also get your app's logs by typing `meteor logs myApp`.

<% end %>

Mongo's syntax is familiar, as it uses a JavaScript interface. We won't be doing any further data manipulation in the Mongo shell, but we might take a peek inside from time to time just to make sure what's in there.

### Client-Side Collections

Collections get more interesting client-side. When you declare `Posts = new Meteor.Collection('posts');` on the client, what you are creating is a _local, in-browser cache_ of the real Mongo collection. When we talk about a client-side collections being a "cache", we mean it in the sense that it contains a *subset* of your data, and offers very *quick* access to this data.

It's important to understand this point as it's fundamental to the way Meteor works. In general, a client side collection consists of a subset of all the documents stored in the Mongo collection (after all, we generally don't want to send our *whole* database to the client). 

Secondly, those documents are stored *in browser memory*, which means that accessing them is basically instantaneous. So there are no slow trips to the server or the database to fetch the data when you call `Posts.find()` on the client, as the data is already pre-loaded.

<% note do %>

### Introducing MiniMongo

Meteor's client-side Mongo implementation is called MiniMongo. It's not a perfect implementation yet, and you may encounter occasional Mongo features that don't work in MiniMongo. Nevertheless, all the features we cover in this book work similarly in both Mongo and MiniMongo.

<% end %>

### Client-Server Communication

The key piece of all this is how the client-side collection sychronizes its data with the server-side collection of the same name (`'posts'` in our case).

Rather than explaining this in detail, let's just watch what happens.

Start by opening up two browser windows, and accessing the browser console in each one. Then, open up the Mongo shell on the command line. At this point, we should see the single document we created earlier in all three contexts.

~~~bash
> db.posts.find();
{title: "A new post", _id: ObjectId("..")};
~~~
<%= caption "The Mongo Shell" %>

~~~js
❯ Posts.findOne();
{title: "A new post", _id: LocalCollection._ObjectID};
~~~
<%= caption "First browser console" %>

Let's create a new post. In one of the browser windows, run an insert command:

~~~js
❯ Posts.find().count();
1
❯ Posts.insert({title: "A second post"});
'xxx'
❯ Posts.find().count();
2
~~~
<%= caption "First browser console" %>

Unsurprisingly, the post made it into the local collection. Now let's check Mongo:

~~~bash
❯ db.posts.find();
{title: "A new post", _id: ObjectId("..")};
{title: "A second post", _id: 'yyy'};
~~~
<%= caption "The Mongo Shell" %>

As you can see, the post made it all the way back to the Mongo database, without us writing a single line of code to hook our client up to the server (well, strictly speaking, we did write a _single_ line of code: `new Meteor.Collection('posts')`). But that's not all!

Bring up the second browser window and enter this in the browser console:

~~~js
❯ Posts.find().count();
2
~~~
<%= caption "Second browser console" %>

The post is there too! Even though we never refreshed or even interacted with the second browser, and we certainly didn't write any code to push updates out. It all happened magically -- and instantly too, although this will become more obvious later.

What happened is that our server-side collection was informed by a client collection of a new post, and took on the task of distributing that post into the Mongo database and back out to all the other connected `post` collections. 

Fetching posts on the browser console isn't that useful. We will learn how to wire this data into our templates, and in the process turn our simple HTML prototype into a functioning realtime web application.

### Keeping it Real-time

Looking at the contents of our Collections on the browser console is one thing, but what we'd really like to do is display the data, and the changes to that data, on the screen. In doing so we'll turn our app from a simple web *page* displaying static data, to a realtime web *application* with dynamic, changing data.

Let's find out how.

### Populating the Database

The first thing we'll do is put some data into the database. We'll do so with a fixture file that loads a set of structured data into the `Posts` collection when the server first starts up. 

First, let's make sure there's nothing in the database. We'll use `meteor reset`, which erases your database and resets your project. Of course, you'll want to be very careful with this command once you start working on real-world projects. 

Stop the Meteor server (by pressing `ctrl-c`) and then, on the command line, run:

~~~bash
$ meteor reset
~~~

The reset command completely clears out the Mongo database. It's a useful command in development, where there's a strong possibility of our database falling into an inconsistent state. 

Now that the database is empty, we can add the following code that will load up three posts whenever the server starts and finds the `Posts` collection empty:

~~~js
if (Posts.find().count() === 0) {
  Posts.insert({
    title: 'Introducing Telescope',
    author: 'Sacha Greif',
    url: 'http://sachagreif.com/introducing-telescope/'
  });
  
  Posts.insert({
    title: 'Meteor',
    author: 'Tom Coleman',
    url: 'http://meteor.com'
  });
  
  Posts.insert({
    title: 'The Meteor Book',
    author: 'Tom Coleman',
    url: 'http://themeteorbook.com'
  });
}
~~~
<%= caption "server/fixtures.js" %>

<%= commit "4-2", "Added data to the posts collection." %>

We've placed this file in the `server/` directory, so it will never get loaded on any user's browser. The code will run immediately when the server starts, and make `insert` calls on the database to add three simple posts in our `Posts` collection. As we haven't built any data security yet, there's no real difference between doing this in a file run on the server or in the browser. 

Now run your server again with `meteor`, and these three posts will get loaded into the database.

### Wiring the data to our HTML with helpers

Now, if we open up a browser console, we see all three posts loaded up into MiniMongo:

~~~js
❯ Posts.find().fetch();
~~~
<%= caption "Browser console" %>

To get these posts into rendered HTML, we can use a template helper. In Chapter 3 we saw how Meteor allows us to bind a *data context* to our Handlebars templates to build HTML views of simple data structures. We can bind in our collection data in the exact same way. We'll just replace our static `postsData` JavaScript object by a dynamic collection.

Speaking of which, feel free to delete the `postsData` code at this point. Here's what `posts_list.js` should now look like:

~~~js
Template.postsList.helpers({
  posts: function() {
    return Posts.find();
  }
});
~~~
<%= caption "client/views/posts/posts_list.js" %>
<%= highlight "2~4" %>

<%= commit "4-3", "Wired collection into `postsList` template." %>

<% note do %>

### Find & Fetch

In Meteor, `find()` returns a *cursor*, which is a [reactive data source](http://docs.meteor.com/#find). When we want to log its contents, we can then use `fetch()` on that cursor to transform it into an array . 

Within an app, Meteor is smart enough to know how to iterate over cursors without having to explicitly convert them into arrays first. This is why you won't see `fetch()` that often in actual Meteor code (and why we didn't use it in the above example).

<% end %>

Now, rather than pulling a list of posts as a static array from a variable, we return a cursor to our `posts` helper. But what does this do? If we go back to our browser, we see:

<%= screenshot "4-3", "Using live data" %>

So we can clearly see that our `{{#each}}` helper has iterated over all of our `Posts`, and displayed them on the screen. The server-side collection pulled the posts from Mongo, passed them over the wire to our client-side collection, and our handlebars helper passed them into the template. 

Now, we'll take this one step further; let's add another post via the console:

~~~js
❯ Posts.insert({
  title: 'Meteor Docs', 
  author: 'Tom Coleman', 
  url: 'http://docs.meteor.com'
});
~~~
<%= caption "Browser console" %>

Look back at the browser -- you should see this:

<%= screenshot "4-4", "Adding posts via the console" %>

You have just seen reactivity in action for the first time. When we told handlebars to iterate over the `Posts.find()` cursor, it knew how to observe that cursor for changes, and patch the HTML in the simplest way to display the correct data on screen.

<% note do %>

### Inspecting DOM Changes

In this case, the simplest change possible was to add another `<div class="post">...</div>`. If you want to make sure this is really what happened, open the DOM inspector and select the `<div>` corresponding to one of the existing posts. 

Now, in the JavaScript console, insert another post. When you tab back to the inspector, you'll see an extra `<div>`, corresponding to the new post, but you will still have the *same* existing `<div>` selected. This is a useful way to tell when elements have been re-rendered and when they have been left alone.

<% end %>

### Connecting Collections: Publications and Subscriptions

So far, we've had the `autopublish` package enabled, which is not intended for production applications. As its name indicates, this package simply says that each collection should be shared in its entirety to each connected client. This isn't what we really want, so let's turn it off.

Open a new terminal window, and type:

~~~bash
$ meteor remove autopublish
~~~

This has an instant effect. If you look in your browser now, you'll see that all our posts have disappeared! This is because we were relying on `autopublish` to make sure our client-side collection of posts was a mirror of all the posts in the database.  

Eventually we'll need to make sure we're only transferring the posts that the user actually needs to see (taking into account things like pagination). But for now, we'll just setup `Posts` to be published in its entirety. 

To do so, we create a simple `publish()` function that returns a cursor referencing all posts:

~~~js
Meteor.publish('posts', function() {
  return Posts.find();
});
~~~
<%= caption "server/publications.js" %>

In the client, we need to *subscribe* to the publication. We'll just add the following line to `main.js`:

~~~js
Meteor.subscribe('posts');
~~~
<%= caption "client/main.js" %>

<%= commit "4-4", "Removed `autopublish` and set up a basic publication." %>

If we check the browser again, our posts are back. Phew! 

### Conclusion

So what have we achieved? Well, although we don't have a user interface yet, what we have now is a functional web application. We could deploy this application to the Internet, and (using the browser console) start posting new stories and see them appear in other user's browsers all over the world.
