---
title: Integrating Search
slug: search
date: 0018/01/01
number: 18
extra: true
points: 1000
published: false
photoUrl: http://www.flickr.com/photos/ikewinski/8377615133/
photoAutor: Mike Lewinski
contents: See what happens behind the scenes when Meteor swaps two DOM elements.|Learn how to animate the reordering of posts.|Learn how to animate the insertion of new posts.
---


setParameter = textSearchEnabled=true

db.runCommand({getParameter: 1, textSearchEnabled: 1})
