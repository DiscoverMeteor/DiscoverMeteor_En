---
title: Meteor Vocabulary
slug: meteor-vocabulary
date: 0014/01/02
number: 14.5
points: 10
sidebar: true
photoUrl: http://www.flickr.com/photos/ikewinski/8110681595/
photoAuthor: Mike Lewinski
contents: Review a few common Meteor terms. 
---


In this book, you'll hear some words which may be new, or at least used in a new way in a Meteor context. We'll use this chapter to define them.

#### Collection

A Meteor Collection is the data store that automatically synchronizes between client and server. Collections have a name (such as `posts`), and usually exist both on client and server. Although they behave differently, they have a common API based on Mongo's API.

#### MiniMongo

The client-side collection is an in-memory data store offering a Mongo-like API. The library that supports this behaviour is called "MiniMongo", to indicate it's a smaller version of Mongo that runs completely in memory.
 
#### Document

Mongo is a document-based data-store, so the objects that come out of collections are called "documents". They are plain JavaScript objects (although they can't contain functions) with a single special property, the `_id`, which Meteor uses to track their properties over DDP.

#### DDP

DDP is Meteor's Distributed Data Protocol, the wire protocol used to synchronize collections and make Method calls. DDP is intended as a generic protocol, which takes the place of HTTP for realtime applications that are data heavy.

#### Client

When we talk about the Client, we are referring to code running in the users *web browser*, whether that is a traditional browser like Firefox or Safari, or something as complex as a UIWebView in a native iPhone application.

#### Computation

A computation is a block of code that runs every time one of the reactive data sources that it depends on changes. If you have a reactive data source (for example, a Session variable) and would like to respond reactively to it, you'll need set up a computation for it. 

#### Server

The Meteor server is a HTTP and DDP server run via node.js. It consists of the all the Meteor libraries as well your server-side JavaScript code. When you start your Meteor server, it connects to a Mongo database (which it starts itself in development).

#### Method

A Meteor Method is a remote procedure call from the client to the server, with some special logic to keep track of collection changes and allow Latency Compensation.

#### Latency Compensation

Is a technique to allow simulation of Method calls on the client, to avoid lagginess while waiting for the server to respond.

#### Template

A template is a method of generating HTML in JavaScript. By default, Meteor supports Handlebars, a logic-less templating system, although there are plans to support more in the future.

#### Template Data Context

When a template renders, it refers to a JavaScript object that provides specific data for this particular rendering. Usually such objects are plain-old-JavaScript-objects (POJOs), often documents from a collection, although they can be more complicated and have functions available on them.

#### Helpers

When a template needs to render something more complex than a document property it can call a helper, a function that is used to aid rendering.

#### Session

The Session in Meteor refers to a client-side reactive data source that's used by your application to track the state that the user's in. 

#### Publication

A publication is a named set of data that is customized for each user that subscribes to it. You set up a publication on the server.

#### Subscription

A subscription is a connection to a publication for a specific client. The subscription is code that runs in the browser that talks to a publication on the server and keeps the data in sync.

#### Cursor

A cursor is the result of running a query on a Mongo collection. On the client side, a cursor isn't just an array of results, but a *reactive* object that can be observed as objects in the relevant collection are added, removed and updated.

#### Package

A Meteor package can consist of 
  1. JavaScript code to run on the server.
  2. JavaScript code to run on the client.
  3. Instructions on how to process resources (such as SASS to CSS).
  4. Resources to be processed.

A package is like a super-powered library. Meteor comes with an extensive set of core packages. There's also [Atmosphere](http://atmosphere.meteor.com), which is a collection of community supplied third party packages.

#### Deps

Deps is Meteor's reactivity system. Deps is used behind the scenes to keep HTML automatically sync with the underlying data model.
