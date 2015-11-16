---
layout: post
title: Getting started with Chef Solo and Knife Solo on Ubuntu
---

## {{ page.title }}

Adam Duke \| 16 Nov 2015


We launched [ZeroPush](https://zeropush.com) using Chef/Knife Solo and a kitchen based largely on [this post](http://teohm.com/blog/2013/04/17/chef-cookbooks-for-busy-ruby-developers). There were a lot of unanswered questions though. How do we safely store secrets? Should we really be running all this on one machine? How do we extract things to run on different machines? Over time we added things to the kitchen that made things easier, but why isn't there a canonical 'deploy my web app' kitchen out there? I've always wanted to go back and start over while applying what I've learned, to clean up the cruft. That's the goal of this series of posts.

Let's start by creating a totally vanilla kitchen.

{% highlight bash %}
$ mkdir webapp-kitchen && cd webapp-kitchen
$ cat > Gemfile << EOL
source 'https://rubygems.org'

gem 'berkshelf'
gem 'knife-solo'
EOL

$ bundle && bundle exec knife solo init .
{% endhighlight %}

Now that we have a kitchen, let's create a machine to run it on. Using Vagrant or your preferred cloud provider, provision a new machine using Ubuntu 14.04 and get the machine's ip. For the purpose of this post, I'm going to set that ip address as a shell variable so that if you're copying and pasting you only have to change it in one place.

{% highlight bash %}
$ KNIFE_SOLO_TARGET=<target ip address>
$ bundle exec knife solo prepare root@$KNIFE_SOLO_TARGET
{% endhighlight %}

The command above will create a file in the kitchen's nodes directory for our machine, and additionally install Chef on the machine. The generated node file should look something like:

{% highlight bash %}
$ cat nodes/$KNIFE_SOLO_TARGET.json
{
  "run_list": [
  ],
  "automatic": {
    "ipaddress": "<your-ip-here>"
  }
}
{% endhighlight %}


We can now actually run Chef on the machine by running:

{% highlight bash %}
$ bundle exec knife solo cook root@$KNIFE_SOLO_TARGET
{% endhighlight %}

That's progress, but doesn't really do much yet. Let's add a recipe to the node's runlist to see something actually happen. Edit your Berksfile to add the apt cookbook as a dependency and add the apt::default recipe to the node's runlist.

{% highlight bash %}
$ cat > Berksfile << EOL
source 'https://api.berkshelf.com'

cookbook 'apt'
EOL

$ cat > nodes/$KNIFE_SOLO_TARGET.json << EOL
{
  "run_list": [
    "apt::default"
  ],
  "automatic": {
    "ipaddress": "$KNIFE_SOLO_TARGET"
  }
}
EOL
{% endhighlight %}

Run Chef again.

{% highlight bash %}
bundle exec knife solo cook root@$KNIFE_SOLO_TARGET
{% endhighlight %}

At this point everything should be working. Running `knife solo cook`, against our machine, uploads the cookbooks to the server, generates some configuration files for Chef, and then starts the chef-client application using the newly generated configuration files and applies the `apt::default` recipe from the runlist.

In the [next post]({% post_url 2015-11-19-base-chef-configuration-using-knife-solo %}), we'll secure the machine a bit by adding a firewall, adding a user for ourselves, giving our user `sudo` access, and a few other tweaks that we can share across all the nodes in our kitchen.
