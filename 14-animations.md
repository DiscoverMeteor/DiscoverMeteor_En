  ---
title: Animations
slug: animations
date: 0014/01/01
number: 14
points: 10
photoUrl: http://www.flickr.com/photos/ikewinski/8377615133/
photoAuthor: Mike Lewinski
contents: See what happens behind the scenes when Meteor swaps two DOM elements.|Learn how to animate the reordering of posts.|Learn how to animate the insertion of new posts.
---

We now have real-time voting, scoring, and ranking. However, this leads to a jarring, erratic user experience as posts jump around on the homepage. We'll use animations to smooth this over.

### Meteor & the DOM

Before we can start the fun part (making things move around), we need to understand how Meteor interacts with the DOM (Document Object Model -- the collection of HTML elements that make up a page's contents).

The crucial point to keep in mind is that elements *cannot be moved*. They can only be deleted and created (note that this is a limitation of the DOM itself, not of Meteor). So to give the illusion of elements A and B switching place, Meteor will actually delete element B and insert a brand new copy (B') before element A. 

This does make animation tricky, as you can't just animate B to move it to a new position, because B will be gone as soon as Meteor re-renders the page (which as we know happens instantly, thanks to reactivity). Instead, you have to animate the newly created B' as it moves from B's old position to its new position before A. 

To switch posts A and B (positioned in positions p1 and p2, respectively), we would go through the following steps:

1. Delete B
2. Create B' before A in the DOM
3. Move B' to p2
4. Move A to p1
5. Animate A to p2
6. Animate B' to p1

The following diagram explains these steps in more detail:

<%= diagram "animation_diagram", "Swtiching two posts", "pull-center" %>

Note that in steps 3 and 4 we're not *animating* A and B' to their positions but "teleporting" them there instantly. Since this is instantaneous, it will give the illusion that B was never deleted, and properly position both elements to be animated back to their new position.

Thankfully, Meteor takes care of steps 1 & 2 for us, so we only need to worry about steps 3 through 6. 

Moreover, in steps 5 and 6 all we're doing is moving the elements to their proper spot. So the only part we really need to worry about is steps 3 and 4, i.e. sending the elements to the animation's starting point. 

### Proper Timing

Up to now we've talked about *how* to animate our posts but not *when* to animate them. 

For steps 3 and 4, the answer is on Meteor's `rendered` template callback inside the `post_item.js` manager, which is fired any time a post's property (in our case, ranking) changes. 

Steps 5 and 6 are a bit trickier. Think of it this way: if you told a perfectly logical android to run north for 5 minutes, and then once that's done run south for 5 minutes, it would probably deduce that since it will end up in the same place, it might as well save its energy and not run at all. 

So if you want to ensure that your android runs during the entire 10 minutes, you have to *wait* until it's ran the first 5 minutes, and *then* tell it to come back.

The browser works in a similar way: if we just gave it both instructions simultaneously, the new coordinates would simply replace the old ones and nothing would happen. In other words, the browser needs to register the position changes as separate points in time, otherwise it won't be able to animate them. 

Meteor doesn't provide a `justAfterRendered` callback, but we can fake it using `Meteor.defer()`, which simply takes a function and defers its execution just enough to register as a different event. 

### CSS Positioning

To animate the posts being reordered around the page, we'll have to venture into CSS territory. A quick review of CSS positioning might be in order. 

Elements on a page use **static** positioning by default. Statically positioned elements just fit within the flow of the page, and their coordinates on the screen cannot be changed or animated. 

**Relative** positioning on the other hand means that the element also fits in the flow of the page, but can be positioned *relative to its original position*. 

**Absolute** positioning goes one step further and lets you give  the element specific x/y coordinates relative to the **document** or **the first absolute or relative-positioned parent element**. 

We'll use relative positioning to animate our posts. We've already taken care of the CSS for you, but if you needed to do it yourself all you would do is add this code to your stylesheet:

~~~css
.post{
  position:relative;
  transition:all 300ms 0ms ease-in;
}
~~~
<%= caption "client/stylesheets/style.css" %>

This makes steps 5 and 6 quite easy: all we need to do is set `top` to `0px` (its default value) and our posts will slide back to their "normal" position. 

This means our only challenge is figuring where to animate them *from* (steps 3 and 4) relative to their new position. In other words, how much to offset them. But that's not very hard either: the correct offset is simply a post's previous position minus its new one. 

<% note do %>

### Position:absolute

We could also use `position:absolute` with a relative parent to position our elements. But a big downside of absolutely positioned elements is that they're completely removed from the flow of the page, causing their parent container to collapse as if it were empty. 

This in turns means we'd need to artificially set the height of the container via JavaScript, instead of leaving the browser reflow elements naturally. Consequently, whenever possible it's best to stick with relative positioning. 

<% end %>

### Total Recall

We do have one more problem though. While element A persists in the DOM and can thus "remember" its previous position, element B experiences reincarnation and comes back to life as B' with its memory wiped clean. 

Thankfully Meteor comes to the rescue by giving us access to the  **template instance** object in the `rendered` callback. As the [Meteor documentation](http://docs.meteor.com/#template_rendered) states:

> In the body of the callback, `this` is a template instance object that is unique to this occurrence of the template and persists across re-renderings. 

So what we'll do is find out a post's current position in the page, and then store that position in the template instance object. This way, even when a post is deleted and recreated, we'll still be able to know where we're supposed to animate it from. 

Template instances also let us access collection data through the `data` property. This will come in handy to get a post's rank. 

### Ranking Posts

We've been talking about posts rank, but this "rank" does not actually exist as a post property, since it's just a consequence of the order in which posts are listed in our collection. If we want to be able to animate posts according to their rank, we'll have to somehow conjure up this property out of thin air. 

Note that we can't put this `rank` property in the database itself, since rank is a relative property that depends on how you sort posts (i.e. a post can be ranked first when sorting by date, but third when sorting by points).

We would ideally put that property in our `newPosts` and `topPosts` collections, but Meteor doesn't offer a convenient mechanism to do this yet. 

So instead, we'll insert `rank` at the last possible step, the `postList` template manager:

~~~js
Template.postsList.helpers({
  postsWithRank: function() {
    this.posts.rewind();
    return this.posts.map(function(post, index, cursor) {
      post._rank = index;
      return post;
    });
  }
});
~~~
<%= caption "/client/views/posts/posts_list.js" %>
<%= highlight "2~8" %>

Instead of simply returning the `Posts.find({}, {sort: {submitted: -1}, limit: postsHandle.limit()})` cursor like our previous `posts` helper, `postsWithRank` takes the cursor and adds the `_rank` property to each of its documents. 

Don't forget to update the `postsList` template as well:

~~~html
<template name="postsList">
  <div class="posts">
    {{#each postsWithRank}}
      {{> postItem}}
    {{/each}}
    
    {{#if hasMorePosts}}
      <a class="load-more" href="{{nextPath}}">Load more</a>
    {{/if}}
  </div>
</template>
~~~
<%= caption "/client/views/posts/posts_list.html" %>

<%= highlight "3" %>

<% note do %>

### Be Kind, Rewind

Meteor is one of the most forward-thinking and cutting-edge web frameworks around. But one of its features feels like a throwback to the days of VCRs and video cassette recording, namely the `rewind()` function. 

Whenever you use a cursor with `forEach()`, `map()`, or `fetch()`, you'll need to rewind the cursor afterwards before it's ready to be used again. 

And in some cases, it's better to be on the safe side and `rewind()` the cursor preventively rather than risk a bug. 

<% end %>

### Putting it together

We can now put everything together by using the `post_item.js` manager's `rendered` template callback for our animation logic:

~~~js
Template.postItem.helpers({
  //...
});

Template.postItem.rendered = function(){
  // animate post from previous position to new position
  var instance = this;
  var rank = instance.data._rank;
  var $this = $(this.firstNode);
  var postHeight = 80;
  var newPosition = rank * postHeight;
 
  // if element has a currentPosition (i.e. it's not the first ever render)
  if (typeof(instance.currentPosition) !== 'undefined') {
    var previousPosition = instance.currentPosition;
    // calculate difference between old position and new position and send element there
    var delta = previousPosition - newPosition;
    $this.css("top", delta + "px");
  }
  
  // let it draw in the old position, then..
  Meteor.defer(function() {
    instance.currentPosition = newPosition;
    // bring element back to its new original position
    $this.css("top",  "0px");
  }); 
};

Template.postItem.events({
  //...
});
~~~
<%= caption "/client/views/posts/post_item.js" %>
<%= highlight "5~27" %>


<%= commit "14-1", "Added post reordering animation." %>

It shouldn't be too hard to follow along if you refer back to our previous diagram.

Note that since we set the template instance's `currentPosition` property in the `defer` callback, this means that this property won't exist on the very first render of the template fragment. But this is not a problem since we're not interested in animating that first render anyway. 

Now open your site and start upvoting. You should now see posts gently moving up and down with ballet-like grace!

### Animating New Posts

Our posts are now reordering properly, but we don't really have a "new post" animation yet. Instead of having new posts simply pop up at the top of our list, let's fade them in. 

This is actually more complicated than it sounds. The problem is that Meteor's `rendered` callback actually gets triggered in two separate cases:

1. When a new template is inserted into the DOM
2. Whenever a template's underlying data changes

Only case 1 should be animated, unless you want your user interface lighting up like a christmas tree every time your data changes. 

So let's make sure we only animate posts when they're actually new, and not just when they're re-rendered because their data changed. We're already testing for the presence of an instance variable (which is only set after the first render), so we just have to go back to our `rendered` callback and add an `else` block:

~~~js
Template.postItem.helpers({
  //...
});

Template.postItem.rendered = function(){
  // animate post from previous position to new position
  var instance = this;
  var rank = instance.data._rank;
  var $this = $(this.firstNode);
  var postHeight = 80;
  var newPosition = rank * postHeight;
  
  // if element has a currentPosition (i.e. it's not the first ever render)
  if (typeof(instance.currentPosition) !== 'undefined') {
    var previousPosition = instance.currentPosition;
    // calculate difference between old position and new position and send element there
    var delta = previousPosition - newPosition;
    $this.css("top", delta + "px");
  } else {
    // it's the first ever render, so hide element
    $this.addClass("invisible");
  }
  
  // let it draw in the old position, then..
  Meteor.defer(function() {
    instance.currentPosition = newPosition;
    // bring element back to its new original position
    $this.css("top",  "0px").removeClass("invisible");
  }); 
};

Template.postItem.events({
  //...
});
~~~
<%= caption "/client/views/posts/post_item.js" %>
<%= highlight "19~22,28" %>

<%= commit "14-2", "Fade items in when they are drawn." %>

Note that the `removeClass("invisible")` we added in the `defer()` function will run for every rendering. But it will only  do something if the `.invisible` class is actually present on the element, which will only be true the first time it's rendered. 

<% note do %>

### CSS & JavaScript

You might have noticed that we're using an `.invisible` CSS class to trigger the animation instead of animating the CSS `opacity` property directly like we did for `top`. This is because for `top`, we needed to animate the property to a specific value that depended on the instance data. 

On the other hand, here we only want to show and hide an element independently of its data. Since it's a good idea to keep your CSS out of your JavaScript as much as possible, we'll only add and remove the class here, and specify the details of the animation over in our stylesheet. 

<% end %>

We should finally have the animation behavior we wanted. Load up your app and give it a try! And you can also play around with the `.post` and `.post.invisible` classes to see if you can come up with other transitions. Hint: [CSS easing functions](http://matthewlein.com/ceaser/) are a good place to start!