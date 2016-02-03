---
layout: post
title: iOS Push Notifications using Jabber and Ejabberd
---

## {{ page.title }}

Adam Duke \| 23 January 2013

#### Problem

I recently had the fortune to work on an iOS application that included a feature to allow users to chat with each other. The decision was made to use ejabberd as the backend for the chat, with a native, SMS like, interface for the front end.

One of the challenges we had to overcome was sending push notifications to offline users when they received an incoming chat message. At the time, it didn't seem like there was a great solution to get messages to offline users directly from ejabberd, so we started doing a bit of research. The solution we came up with requires a some integration, but only between 60-70 lines of code.

#### Hypothesis

Because we were concurrently developing a rails app to provide the API for the iOS app, we decided the easiest solution would be to forward the offline messages from ejabberd to the rails app and integrate [ZeroPush](https://zeropush.com) to handle the communication with Apple's push notification service.

We decided to see if we could get ejabberd to forward the messages for offline users. In my research, I came across a module for ejabberd called [mod_offline_prowl](http://www.unsleeping.com/2010/07/31/prowl-module-for-ejabberd) that was originally intended to forward offline messages from ejabberd to [Prowl](http://www.prowlapp.com). After digging into the mod_offline_prowl code, I saw that it could be adapted to do what I needed. There were two things I needed it to do a bit differently.

1. Easily configure the url where the data was being sent.
 * Everything was regularly deployed to a staging environment and compiling ejabberd modules per environment seemed like a maintenance headache.
1. Format the POST body with the data needed by the rails app

#### Solution

The basic idea behind ejabberd modules is that there are lifecylce events that occur, such as a user coming online, a user going offline, or receiving a message, and a module can subscribe to a particular event. When an interesting event happens the modules that are subscribed to that event get run.

The original mod_offline_prowl uses <tt>gen_mod:get_module_opt</tt> to retrieve certain options from ejabberd.cfg, so it seemed reasonable to use a similar approach to retrieve a url and access token for our purposes.

Based on the applications requirements, I came up with a minimal piece of configuration that we could use to set the details per environment. We would need an auth token to avoid bogus messages from being forwarded, and a url where the messages should be forwarded.

{% highlight erlang %}
{mod_offline_post, [
    {auth_token, "your-secret-token-here"},
    {post_url, "http://localhost:3000/your/custom/url"}
]}
{% endhighlight%}

The module gets the properties from ejabberd.cfg using the following:

{% highlight erlang %}
Token = gen_mod:get_module_opt(To#jid.lserver, ?MODULE, auth_token, [] ),
PostUrl = gen_mod:get_module_opt(To#jid.lserver, ?MODULE, post_url, [] )
{% endhighlight%}

Each time a message is sent to an offline user, an http request to the <tt>PostUrl</tt> is made using the <tt>Token</tt>, and the contents of <tt>To</tt>, <tt>From</tt>, and <tt>Body</tt> fields. We implemented a controller action that accepts the incoming post and hands it off to [ZeroPush](https://zeropush.com). The message makes a bunch of hops between applications, but in reality there is very little code which means fewer bugs, and a lower cost of maintenance.

The full mod_offline_post module along with two others can be found [on my github profile](https://github.com/adamvduke/mod_interact).
