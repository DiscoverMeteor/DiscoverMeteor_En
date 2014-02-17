---
title: Latency Compensation
slug: latency-compensation
date: 0007/01/02
number: 7.5
points: 10
sidebar: true
photoUrl: http://www.flickr.com/photos/ikewinski/9473352049/
photoAuthor: Mike Lewinski
contents: Understand latency compensation.|Slow your app down and see what's going on.|Learn how Meteor Methods call each other.
---

In the last chapter, we introduced a new concept in the Meteor world: **Methods**. 

<%= diagram "latency1", "Without latency compensation", "pull-right" %>

A Meteor Method is a way of executing a series of commands on the server in a structured way. In our example, we used a Method because we wanted to make sure that new posts were tagged with their author's name and id as well as the current server time. 

However, if Meteor executed Methods in the most basic way, we'd have a problem. Consider the following sequence of events (note: the timestamps are random values picked for illustrative purpose only):

- *+0ms:* The user clicks a submit button and the browser fires a Method call.
- *+200ms:* The server makes changes to the Mongo database.
- *+500ms:* The client receives these changes, and updates the UI to reflect them.

If this were the way Meteor operated, then there'd be a short lag between performing such actions and seeing the results (that lag being more or less noticeable depending on how close you were to the server). We can't have that in a modern web application!

### Latency Compensation

<%= diagram "latency2", "With latency compensation", "pull-right" %>

To avoid this problem, Meteor introduces a concept called **Latency Compensation**. When we defined our `post` Method, we placed it within a file in the `collections/` directory. This means it is available to both the server *and the client* -- and it will run on both at the same time!

When you make a Method call, the client sends off the call to the server, but also simultaneously *simulates* the action of the Method on its client collections. So our workflow now becomes:

- *+0ms:* The user clicks a submit button and the browser fires a Method call.
- *+0ms:*  The client simulates the action of the Method call on the client collections and changes the UI to reflect this
- *+200ms:*  The server makes changes to the Mongo database.
- *+500ms:*  The client receives those changes and undoes its simulated changes, replacing them with the server's changes (which are generally the same). The UI changes to reflect this.

This results in the user seeing the changes instantly. When the server's response returns a few moments later, there may or may not be noticeable changes as the server's canonical documents come down the wire. One thing to learn from this is that we should try to make sure we simulate the real documents as closely as we can.

### Observing Latency Compensation

We can make a little change to the `post` Method call to see this in action. To do so, we'll be doing some advanced coding with the `futures` npm package to delay the insertion of objects in our Method.

We'll use `isSimulation` to ask Meteor if the Method is currently being invoked as a stub. A [stub](http://docs.meteor.com/#methods_header) is the Method simulation that Meteor runs on the client in parallel, while the "real" Method is being run on the server.

So we'll ask Meteor if the code is being executed on the client. If so, we'll add the string `(client)` at the end of our post's title. If not, we'll add the string `(server)`:

~~~js
Meteor.methods({
  post: function(postAttributes) {
    // […]
    
    // pick out the whitelisted keys
    var post = _.extend(_.pick(postAttributes, 'url', 'message'), {
      title: postAttributes.title + (this.isSimulation ? '(client)' : '(server)'),
      userId: user._id, 
      author: user.username, 
      submitted: new Date().getTime()
    });
    
    // wait for 5 seconds
    if (! this.isSimulation) {
      var Future = Npm.require('fibers/future');
      var future = new Future();
      Meteor.setTimeout(function() {
        future.return();
      }, 5 * 1000);
      future.wait();
    }
    
    var postId = Posts.insert(post);
    
    return postId;
  }
});
~~~
<%= caption "collections/posts.js" %>
<%= highlight "6, 7, 13~22" %>

Note: in case you're wondering, the `this` in `this.isSimulation` is a [Method invocation object](http://docs.meteor.com/#meteor_methods) that provides access to various useful variables.

Exactly how [Futures](https://npmjs.org/package/future) work is outside of the scope of this book, but we've basically told Meteor to wait for 5 seconds before doing the insert on the server collection. 

We'll also make a submit redirect directly to the post list:

~~~js
Template.postSubmit.events({
  'submit form': function(event) {
    event.preventDefault();
    
    var post = {
      url: $(event.target).find('[name=url]').val(),
      title: $(event.target).find('[name=title]').val(),
      message: $(event.target).find('[name=message]').val()
    }
    
    Meteor.call('post', post, function(error, id) {
      if (error)
        return alert(error.reason);
    });
    Router.go('postsList');
  }
});
~~~
<%= caption "client/views/posts/post_submit.js" %>
<%= highlight "15" %>

<%= scommit "7-5-1", "Demonstrate the order that posts appear using a sleep." %>

If we create a post now, we see latency compensation clearly. First, a post is inserted with `(client)` in the title (the first post in the list, linking to GitHub):

<%= screenshot "s5-1", "Our post as first stored in the client collection" %>

Then, five seconds later, it is cleanly replaced with the real document that was inserted by the server:

<%= screenshot "s5-2", "Our post once the client receives the update from the server collection" %>

### Client Collection Methods

You might think that Methods are complicated after this, but in fact they can be quite simple. We've actually seen three very simple Methods already: the collection mutation Methods, `insert`, `update` and `remove`.

When you define a server collection called `'posts'`, you are implicitly defining three Methods: `posts/insert`, `posts/update` and `posts/delete`. In other words, when you call `Posts.insert()` on your client collection, you are calling a latency compensated Method that does two things: 

1. Checks to see if we can make the mutation by calling `allow` and `deny` callbacks (this doesn't need to happen in the simulation however).
2. Actually makes the modification to the underlying data store.
  
### Methods Calling Methods

If you are keeping up, you might have just realized that our `post` Method is calling another Method (`posts/insert`) when we insert our post. How does this work?
 
When the simulation (client-side version of the Method) is being run, we run `insert`'s simulation (so we insert into our client collection), but we *do not* call the real, server-side `insert`, as we expect that the *server-side* version of `post` will do this.

Consequently, when the server-side `post` Method calls `insert` there's no need to worry about simulation, and the insertion goes ahead smoothly.