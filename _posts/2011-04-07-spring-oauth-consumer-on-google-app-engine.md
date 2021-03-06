---
layout: default
title: Using Spring's CoreOAuthConsumerSupport on Google App Engine
---

## {{ page.title }}

Adam Duke \| 07 April 2011

**The Problem**

After [fixing CoreOAuthConsumerSupport](/2011/04/06/spring-oauth-consumer-bug-fix.html) and developing on the local [Google App Engine](http://appengine.google.com) environment I deployed my app to the production environment and promptly discovered a problem. It turns out that Spring's CoreOAuthConsumerSupport has dependencies on java.net.Proxy and java.net.ProxySelector, both which are restricted classes in the app engine environment. During my applications startup spring attempts to instantiate a singleton bean that I extended from CoreOAuthConsumerSupport, which causes an exception to be thrown regarding the restricted classes.

**The Fix**

In this case the fix is pretty easy because the CoreOAuthConsumerSupport class isn't highly dependent on the functionality of the restricted classes. In addition to using the getAuthorizationHeader method from [my last post](/2011/04/06/spring-oauth-consumer-bug-fix.html), I had to remove the proxySelector instance variable along with it's getter/setter, the selectProxy method, and modify the openConnection method not to call selectProxy. Instead of calling

<script src="https://gist.github.com/950315.js"></script>

it simply calls

<script src="https://gist.github.com/950317.js"></script>

instead.

The source for the fixed class is available on github at [adamvduke/spring_ext](https://github.com/adamvduke/spring_ext)
