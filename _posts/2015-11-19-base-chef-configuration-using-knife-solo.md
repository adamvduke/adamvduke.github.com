---
layout: post
title: Base Chef configuration with Chef and Knife Solo
---

## {{ page.title }}

Adam Duke \| 19 Nov 2015

In the [last post]({% post_url 2015-11-16-getting-started-with-chef-solo-and-knife-solo-on-ubuntu %}), we created a vanilla 'kitchen' and bootstrapped a single server with Chef. To follow up on that, we're going to:

<ol>
  <li>Extract our runlist into a cookbook that we can share across machines</li>
  <li>Secure the machine a bit by adding a firewall</li>
  <li>Add a user for ourselves</li>
  <li>Give our user `sudo` access</li>
  <li>A few other tweaks that we can share across all the nodes in our kitchen.</li>
</ol>

### Extracting a cookbook

We can generate the skeleton of a cookbook using knife. In the root of your kitchen run the following, but substitute your name and email:

{% highlight bash %}
$ bundle exec knife cookbook create webapp-kitchen-base -C 'Adam Duke' -m 'adam.v.duke@gmail.com' -I mit -o site-cookbooks/
{% endhighlight %}

This will create a cookbook skeleton in site-cookbooks/webapp-kitchen-base. We won't go into all the directories that are generated, but the directory/file structure should be similar to the output from `tree` below. For our purposes recipes/default.rb and metadata.rb are the most important.

{% highlight bash %}
$ tree
├── CHANGELOG.md
├── LICENSE
├── README.md
├── attributes
├── definitions
├── files
│   └── default
├── libraries
├── metadata.rb
├── providers
├── recipes
│   └── default.rb
├── resources
└── templates
    └── default
{% endhighlight %}

Replace `apt::default` with `webapp-kitchen-base::default` in the node's runlist, add the line `depends 'apt'` to the webapp-kitchen-base/metadata.rb file, and add the line `include_recipe 'apt::default'` to the webapp-kitchen-base/recipes/default.rb file. Running chef should produce the same output as before.

{% highlight bash %}
$ bundle exec knife solo cook root@$KNIFE_SOLO_TARGET
{% endhighlight %}

### Adding a firewall

Let's limit the open ports on our machine to only ssh by default. Add the `firewall` cookbook to your Berksfile and a corresponding `depends 'firewall'` to the webapp-kitchen-base/metadata.rb file. Add the following two lines to webapp-kitchen-base/recipes/default.rb.

{% highlight ruby %}
node.set['firewall']['allow_ssh'] = true
include_recipe 'firewall::default'
{% endhighlight %}

If we run Chef now, the firewall cookbook's default recipe will create a firewall rule to only allow traffic to port 22.

{% highlight bash %}
$ bundle exec knife solo cook root@$KNIFE_SOLO_TARGET
{% endhighlight %}

### Adding a non-root user

Now traffic is limited to only ssh, but we're still logging in as root all the time. Let's add a non-root user for the times we actually need to log in to the machine. Add the `user` cookbook to your Berksfile and a corresponding `depends 'user'` to the webapp-kitchen-base/metadata.rb file. We can either use the `user_account` resource provided by the user cookbook directly, or use the `data_bag` recipe. For now, we'll use the data_bag recipe because it seems a little cleaner. Add the following two lines to webapp-kitchen-base/recipes/default.rb, again substituting your username.

{% highlight ruby %}
node.set['users'] = ['adamd']
include_recipe 'user::data_bag'
{% endhighlight %}

Create the data bag for your user by adding a file to data_bags/users/<user-name>.json with content similar to the following:

{% highlight json %}
{
  "id": "adamd",
  "comment": "Adam Duke",
  "home": "/home/adamd",
  "groups": [
    "admin"
  ],
  "ssh_keys": [
    "ssh-rsa AAA............o6l"
  ]
}
{% endhighlight %}

Run Chef again and you will see the `user::data_bag` recipe create your user.

{% highlight bash %}
$ bundle exec knife solo cook root@$KNIFE_SOLO_TARGET
{% endhighlight %}

### Allowing sudo

Now we can log in as a non-root user, but we don't have sudo privileges. We can fix that by allowing users of the group admin to use passwordless sudo. Add the `sudo` cookbook to your Berksfile and a corresponding `depends 'sudo'` to the webapp-kitchen-base/metadata.rb file. Add the following two lines to webapp-kitchen-base/recipes/default.rb.

{% highlight ruby %}
node.set['authorization']['sudo']['groups'] = ['admin']
node.set['authorization']['sudo']['passwordless'] = true
include_recipe 'sudo::default'
{% endhighlight %}

Run Chef again and you will see the contents of /etc/sudoers change to allow the admin group.

{% highlight bash %}
$ bundle exec knife solo cook root@$KNIFE_SOLO_TARGET
{% endhighlight %}

Let's try running chef on the machine as our user rather than root.

{% highlight bash %}
$ bundle exec knife solo cook $KNIFE_SOLO_TARGET
{% endhighlight %}

At this point I got the following an error:

{% highlight bash %}
Running Chef: sudo chef-solo -c ~/chef-solo/solo.rb -j ~/chef-solo/dna.json
[2015-11-15T16:58:16-05:00] WARN: Did not find config file: /home/adamd/chef-solo/solo.rb, using command line options.
[2015-11-15T16:58:16-05:00] FATAL: Cannot load configuration from /home/adamd/chef-solo/dna.json
ERROR: RuntimeError: chef-solo failed. See output above.
{% endhighlight %}

It turns out that there was a recent change to knife-solo that attempts to speed things up by setting some ssh options to keep open connections for uploading the kitchen. That causes issues when switching users because the open connection is tied to the previous user. See details on the issue [here](https://github.com/matschaffer/knife-solo/issues/444). The problem can be worked around by downgrading knife-solo or the control master option can also be turned off by setting `knife[:ssh_control_master] = "no"`, in webapp-kitchen/.chef/knife.rb and deleting any sockets that may be in $HOME/.chef/knife-solo-sockets.

You should now be able to run knife solo as your user successfully.

### Disable Root Login and Password Authentication

We now have a user that can act as root when necessary, and run Chef for us, so let's disable root logins by changing the sshd config and while we're there, let's also disable password logins. Add the `openssh` cookbook to your Berksfile and a corresponding `depends 'openssh'` to the webapp-kitchen-base/metadata.rb file. Add the following two lines to webapp-kitchen-base/recipes/default.rb.

{% highlight ruby %}
node.set['openssh']['server']['permit_root_login'] = 'no'
node.set['openssh']['server']['password_authentication'] = 'no'
include_recipe 'openssh::default'
{% endhighlight %}

Run Chef to have the updated recipes applied.

{% highlight bash %}
$ bundle exec knife solo cook $KNIFE_SOLO_TARGET
{% endhighlight %}

Try to login to the machine as root.

{% highlight bash %}
$ ssh root@$KNIFE_SOLO_TARGET
Permission denied (publickey).
{% endhighlight %}

### Time

Sometimes we write time dependent code, and it would be nice if that code behaved the same everywhere because the computer's clock was correct. Adding the ntp::default recipe to our machines will make sure that we stay in sync with the correct time. Additionally, setting the server to UTC time will make sure that we always have a consistent reference for comparing times. Add the `ntp` and `timezone` cookbooks to your Berksfile and corresponding `depends 'ntp'` and `depends 'timezone'` lines to the webapp-kitchen-base/metadata.rb file. Add the following lines to webapp-kitchen-base/recipes/default.rb.

{% highlight ruby%}
include_recipe 'ntp::default'
include_recipe 'timezone::default'
{% endhighlight %}

One last Chef run to apply the time settings and we can start installing software to actually run our apps.

{% highlight bash %}
$ bundle exec knife solo cook $KNIFE_SOLO_TARGET
{% endhighlight %}

In the next post we'll install and configure postgres, and subsequently the dependencies to run our first application, which in this case will be a simple rails application.
