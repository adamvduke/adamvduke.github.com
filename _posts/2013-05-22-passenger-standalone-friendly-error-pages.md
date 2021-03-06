---
layout: post
title: Turning off Friendly Error Pages in Passenger Standalone
---

## {{ page.title }}

Adam Duke \| 22 May 2013

We recently decided to try our hand at running our own servers rather than using Heroku for our rails hosting. [Teohm](https://github.com/teohm) put together a [fantastic walkthrough](http://teohm.github.io/blog/2013/04/17/chef-cookbooks-for-busy-ruby-developers) using Chef and Knife Solo to get up and running.

After a learning a bit about Chef/Knife and building a kitchen with a few tweaks for our specific setup, we had [ZeroPush](https://zeropush.com) running on a [Digital Ocean](http://digitalocean.com) machine fronted by nginx and passenger.

When we were setting up our deploy scripts, we had an instance where the app failed to boot and we were confronted with passenger's "friendly error page".

![passenger friendly error](/blog_content/passenger/passenger-friendly-error.png)

During development the friendly error page is far more welcome than digging through a bunch of log files to find out what went wrong. However, when we're talking about your production site and potentially sensitive data getting returned to the browser, it's time to start looking for a way to turn the error pages off.

The actual fix for turning the error pages off is trivial. They can be disabled by adding the following to your nginx config:

```passenger_friendly_error_pages off;```

Our issue was that we are using passenger standalone as setup by teohm's recipes, and the nginx config for passenger standalone gets generated during application startup. I looked through the passenger documentation as thoroughly as I could, but I couldn't find a way to make passenger standalone startup and generate the desired nginx config. I started looking at the passenger source to see how difficult it would be to add a command line option that would get the desired result, and it turned out to be fairly easy.

The nginx config that passenger generates comes from an erb template within the passenger gem. I needed to pass a new command line option to disable the error pages, look for that option's presence when compiling the erb template, and conditionally include the passenger configuration.

I decided to add the command line option --no-friendly-error-pages, in order to preserve the default behavior of serving the error pages. During command line parsing all of the options are stored in a hash, if the --no-friendly-error-pages option is passed, the default value in the hash is overridden. The final step is to make the modification to the erb template. I've made a [pull request](https://github.com/FooBarWidget/passenger/pull/77) with my changes. The changed lines are all included below.


{% highlight ruby %}
# start_command.rb
DEFAULT_OPTIONS = {
		:address       => '0.0.0.0',
		:port          => 3000,
		:env           => ENV['RAILS_ENV'] || ENV['RACK_ENV'] || 'development',
		:max_pool_size => 6,
		:min_instances => 1,
		:spawn_method  => Kernel.respond_to?(:fork) ? 'smart' : 'direct',
		:nginx_version => PREFERRED_NGINX_VERSION,
		:friendly_error_pages => true # This was added
	}.freeze

# This is the new option
opts.on("--no-friendly-error-pages", wrap_desc("Disable passenger_friendly_error_pages")) do
  @options[:friendly_error_pages] = false
end

# config.erb
<% unless @options[:friendly_error_pages] %>passenger_friendly_error_pages off;<% end %>
{% endhighlight %}
