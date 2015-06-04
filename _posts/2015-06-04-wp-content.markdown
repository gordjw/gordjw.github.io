---
layout: post
title:  "moving wp-content"
date:   2015-06-04 11:02:25
categories: wordpress
tags: 
image: /assets/article_images/2014-11-30-mediator_features/night-track.JPG
---
# Moving the wp-content folder

If you've ever run a WordPress installation that uses any kind of deployment tool, you'll know how annoying it is that wp-content sits inside your webroot.

I use plain old <code>svn up</code> combined with svn:ignore, and that works well enough. But with 4 installations and more on the way, patching them gets pretty tedious, and you've got to remember to get rid of the wp-content directory that comes with the latest WordPress core update.

If only there was some way that wp-content, and all of that non-upgrade-friendly user content, could be moved outside of the WordPress core.

## Roots, rocks and sage

The guys at Roots have created Bedrock, which seems pretty well thought out.

Bedrock separates wp-content and core WordPress, it has support for multiple environments, and some kind of deployinator, among a whole bunch of other stuff.

It's overkill for what I need though, and one drawback is that wp-config.php sits inside your webroot. It has to, because of the way their folder structure is, and because of WordPress' requirements for the location of wp-config.php. Sure, you can block access to wp-config with nginx rules, but I'd rather not risk it.

## D.I.Y.

Inspired by Bedrock, but wanting to cut back to the bare minimum, and to get rid of wp-config.php from the webroot, I set out to build my own mousetrap.

Your directory structure will end up looking like:

{% highlight bash %}
/data/www/web

/data/www/wp-content

/data/www/wp-config.php
{% endhighlight %}

The first part's pretty easy, move wp-content out of the WordPress webroot.

{% highlight bash %}
$ mv /data/www/web/wp-content /data/www/
{% endhighlight %}

If you refresh your WP site now, it'll all be broken, although the backend will largely still work.

Next, tell WordPress where your wp-content folder is now.

{% highlight bash %}
$ vi /data/www/wp-config.php
define('WP_CONTENT_DIR', '/data/www/wp-content');
{% endhighlight %}

Your themes should start to come back, but you'll notice on the frontend that no styles, scripts or images will load.
That's because <code>WP_CONTENT_DIR</code> only works for PHP files. WP still doesn't know where the client side assets are.

We need to also set <code>WP_CONTENT_URL</code>.

{% highlight php %}
if (isset($_SERVER['HTTP_X_FORWARDED_PROTO']) && $_SERVER['HTTP_X_FORWARDED_PROTO'] == 'https')
    $_SERVER['HTTPS'] = 'on';


// is_ssl doesn't always work, e.g. behind load balancers
if((isset($_SERVER['HTTPS']) && $_SERVER['HTTPS'] == 'on') || is_ssl())
  define('WP_CONTENT_URL', 'https://' . $_SERVER['HTTP_HOST'] . '/wp-content');
else
  define('WP_CONTENT_URL', 'http://' . $_SERVER['HTTP_HOST'] . '/wp-content');
{% endhighlight %}

We do a couple of SSL checks to make sure we use the right scheme, and then we tell WP that it should be looking for client side assets in /wp-content.

First snag!

I use nginx, so the config given is for that, but Apache config will use the same principles.

The problem is that the webroot is /data/www/web

Previously, when wp-content lived in /data/www/web/wp-content, this was fine!

But we've moved it out of the webroot now, to have some separation of concerns, so nginx won't be able to find anything.

Solution: add a location block and an alias to tell nginx where we moved wp-content to.

Put this in the server block of your nginx config.

{% highlight bash %}
  location ^~ /wp-content/ {
    alias /data/www/wp-content/;
    break;
  }
{% endhighlight %}

That's it! Pretty simple stuff.

We're now able to update WordPress core without overwriting any user content. In fact, you could delete the whole web folder, redownload WordPress, and it would be fine. I'd recommend using git instead of doing that, but you could if you wanted.

wp-config is still outside the webroot, so nobody will be able to access it through nginx. Your super secret DB password is safe!

This config will help you scale easily too. You could host the WP core on a fileshare, and mount it on all your webservers. Then, you'd just need to update the copy on the fileshare once, and voila, all your sites are updated instantly and without any disruption.


If you want to look at the whole config files, here you go:

- https://gist.github.com/gordjw/3db7bf513fbd51843ce6
- https://gist.github.com/gordjw/eedcdbf3e556aa729f35
