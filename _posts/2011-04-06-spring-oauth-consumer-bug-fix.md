---
layout: default
title: Fixing Spring's CoreOAuthConsumerSupport
---

## {{ page.title }}

Adam Duke \| 06 April 2011

**The Background**

For reference the OAuth spec at [tools.ietf.org](http://tools.ietf.org/html/rfc5849) is what I'm basing my research/knowledge on.

Earlier this year, I started working on a Java based [Google App Engine](http://appengine.google.com) project. During the course of writing that application I came to the realization that I would need to be able to make signed OAuth requests to one or more OAuth protected resources. I came across a number of solutions, but ultimately started using the library provided by the [Codehaus](http://codehaus.org) spring-security project. The Codehaus project has since been "adopted into SpringSource as an official extension for Spring Security."

During my testing I tried two versions of the spring-security-oauth library with similar results. The maven dependencies are below:

<script src="https://gist.github.com/950320.js?file=snippet1.xml"></script>

<script src="https://gist.github.com/950320.js?file=snippet2.xml"></script>

**The Problem**

One of my project requirements is to make an http POST request where the body that has multiple application/x-www-form-urlencoded parameters. My first attempt was to use OAuthConsumerSupport's configureURLForProtectedAccess method, with a Map containing my extra parameters. This resulted in those parameters correctly being used in calculating the signature base string and subsequently the the oauth_signature parameter, however they were also being included in the Authorization header. From my understanding and research of the oauth protocol, the extra parameter/value pairs need to be included in the signature base string in order to sign the request, but they should be omitted from the Authorization header. The extra parameter/value pairs should then **ONLY** be included in the POST body.  See the following for examples. (newlines added for clarity)

Correct Signature Base String:

<script src="https://gist.github.com/950320.js?file=correct_sig_base_string.txt"></script>

Expected Authorization header:

<script src="https://gist.github.com/950320.js?file=expected_auth_header.txt"></script>

Actual Authorization header:

<script src="https://gist.github.com/950320.js?file=actual_auth_header.txt"></script>

The POST body in both cases was written as:

Post body:

<script src="https://gist.github.com/950320.js?file=post_body.txt"></script>

Twitter's OAuth provider does not seem to mind extra parameters included in the Authorization header. However, when attempting to use Google's OAuth provider available in App Engine, the provider throws an OAuthRequestException when it encounters a parameter that it doesn't expect. The google provider seems correct according to the OAuth 1a spec [here](http://tools.ietf.org/html/rfc5849#section-3.5.1).

Reading through the RFC, the extra parameters should be used in the signing of the request, but non "protocol parameters" should be included either in the POST body, or the url query string. My first attempt to fix the issue was to simply exclude parameters from the header in the CoreOAuthConsumerSupport's getAuthorizationHeader method, based on if the parameter name had the prefix "oauth_". However, it is possible that a provider can define it's own extra parameters that are allowed in the Authorization header. e.g. Google's [OAuthGetRequestToken](http://code.google.com/apis/accounts/docs/OAuth_ref.html#RequestToken) endpoint requires the extra parameters **scope** and accepts an optional parameter **xoauth_displayname**.

**The Fix**

In order to fix the issue I extended CoreOAuthConsumerSupport and overrode the getAuthorizationHeader method. My class adds an extra dependency to the class in the form of a Map bean. The map should contain a list of strings for each OAuth resource that is configured, keyed on that resource's id (the value returned by ProtectedResourceDetails.getId()). Each string in the list represents the name for an extra parameter that the resource supports in the Authorization header. e.g. Referencing the earlier example, if I configure my resource that connects to google with the name "google" the map would have an entry keyed on the String "google", and it's value would be a list of the strings "scope" and "xoauth_displayname".

The source for the extended class is available on github at [adamvduke/spring_ext](https://github.com/adamvduke/spring_ext)
