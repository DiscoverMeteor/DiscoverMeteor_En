---
title: Integrating MailChimp
slug: mailchimp
date: 0019/01/01
number: 19
extra: true
points: 1000
published: false
photoUrl: http://www.flickr.com/photos/ikewinski/8377615133/
photoAutor: Mike Lewinski
contents: See what happens behind the scenes when Meteor swaps two DOM elements.|Learn how to animate the reordering of posts.|Learn how to animate the insertion of new posts.
---


### Mailchimp

~~~js
Package.describe("Mailchimp API library");

Npm.depends({mailchimp: '1.0.3'});

Package.on_use(function (api) {
  api.add_files('mailchimp.js', 'server');
});
~~~
<%= caption "packages/mailchimp/package.js" %>

The mailchimp package's API _is_ synchronous, so we need to wrap it.

**This isn't very nice. Need to keep thinking about what the package we produce here looks like in the end**.

~~~js
var MailChimpAPI = Npm.require('mailchimp').MailChimpAPI;

var API_KEY = 'a20d3e317c6bfe0a08088887c71c05cd-us7';
var LIST_ID = '8af64c21ed';

var Future = Npm.require('fibers/future');
var MailChimpAPIObject = new MailChimpAPI(API_KEY, { version : '1.3', secure : false });

MailChimp = {
  listSubscribe: function(options) {
    var future = new Future();
    MailChimpAPIObject.listSubscribe(options, function(err, res) {
      if (err) {
        future.throw(err);
      } else {
        future.return(res);
      }
    });
    
    return future.wait();
  }
}

// XXX: temporary
Meteor.methods({
  subscribe: function(email) {
    MailChimp.listSubscribe({
      id: LIST_ID,
      email_address: email
    });
  }
})
~~~
<%= caption "packages/mailchimp/mailchimp.js" %>

We've added a temporary `subscribe` method which we'll use to test that our integration is working: SS.

### Signing users up to mailchimp when they sign up to Microscope

To do this, we'll use `accounts`' built in `validateNewUser` callback. Although there is a `onCreateUser` callback (which you might think is the right thing to use here), you can only have *one* such callback, and it has responsibility to construct the user properly. 

Considering we just want to do an external task every time a user is created, a "validation" that always returns true is more appropriate.

XXX: it'd be even better to use this: https://github.com/matb33/meteor-collection-hooks
but I haven't yet vetted it for quality and it might be dodgy.

XXX: actually it's not great as if the user creation fails (e.g. username taken), the validator will run anyway. Probably just use `onCreateUser` for now then..

~~~js
var MAILCHIMP_LIST_ID = '8af64c21ed';

// sign users up to mailchimp when they are created
Accounts.validateNewUser(function(user) {
  // XXX: what to do here?
  if (! user.emails)
    return;
  
  var email = user.emails[0].address;
  
  MailChimp.listSubscribe({
    id: MAILCHIMP_LIST_ID,
    email_address: email
  });
  
  return user;
});
~~~
<%= caption "server/mailchimp.js" %>

XXX: also removed temporary method


<% note do %>

### Asyncro-what?

Why do we go to all this effort to wrap the MailChimp API in essentially the same API with some extra fibers attached?

The reason is that one of the core design decisions made in Meteor was to use syncronous APIs. This is a fairly controversial decision when you consider that the vast majority of existing node packages have asyncronous APIs and thus need to be wrapped in a similar fashion to become 'meteoric'. 

However, there is a reason for this tradeoff--syncronous APIs are easier to deal with. Most programmers agree that when your program is stepping through a series of instructions such as:

1. read data from the database
2. check a file on disk
3. write something back into the database

That it's much easier to write such code in an imperative way (much like the list is written), rather than via a "callback soup".

When you are writing code that needs to flexibly respond to many different event simulatenously in a controlled fashion, then a callback style approach can be very powerful (and to be fair, many complex node applications do this), but when your code is performing the "drudgery" of server side web application logic, this power tends to get in your way.

<% end %>