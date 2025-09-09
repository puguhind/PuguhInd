---
layout: docs
title: Nginx Configuration
headline: How to Configure Nginx for Pico
description: Nginx can be very powerful, but it takes a little extra configuration with Pico - Let's break it down.
toc:
  getting-started: Getting Started
  general-server-configuration: General Server Configuration
  denying-access-to-picos-internal-files: Denying Access to Pico's Internal Files
  php-configuration: PHP Configuration
  url-rewriting: URL Rewriting
  example-configuration: Example Configuration
nav-url: /docs/
---

[Nginx](https://www.nginx.com/) has quickly become a solid contender in the realm of web servers.  While most of the web still uses Apache for serving web content, we acknowledge that a growing portion of our users may be interested in deploying Pico on Nginx instead of Apache.  Deploying Pico on Apache at the moment is relatively painless, with Pico's `.htaccess` file providing almost all the configuration necessary.

Unlike Apache, Nginx uses a ["Centralized" configuration](https://www.digitalocean.com/community/tutorials/apache-vs-nginx-practical-considerations#distributed-vs-centralized-configuration).  Apache scours every folder it hosts in search of `.htaccess` files.  These files hold extra configuration options, often deployed with hosted software to make their setup easy.  Pico is no exception to this, and in most cases, Pico's `.htaccess` provides all the settings required to get it up and running without user interaction.

This "Distributed" configuration can make it hard to understand what's really going on behind the scenes.  Apache's configuration can be spread out into as many folders as your webapps (or your own content) provide.

In comparison, Nginx's "Centralized" configuration is all located in one place.  On a Linux server, this configuration would usually be located in `/etc/nginx`, but this may vary by distro and OS.  While [configuration of Nginx](https://www.nginx.com/resources/wiki/start/) as a whole is out of the scope of this document, we hope to provide you with enough information to get over any hurdles you may encounter.

If you are migrating from Apache, [this article](https://www.digitalocean.com/community/tutorials/apache-vs-nginx-practical-considerations) (also linked above) and [this tutorial](https://www.digitalocean.com/community/tutorials/how-to-migrate-from-an-apache-web-server-to-nginx-on-an-ubuntu-vps) provide an excellent overview on the differences between the two and the changes you'll have to make.

## Getting Started

While the [example]({{ site.github.url }}/docs/#nginx) provided on the previous page is a good starting point, here we will provide a more in-depth look at Nginx configuration.

We've broken down the process of configuring Pico into three segments, in addition to [general server configuration](#general-server-configuration).  The three sets of rules we will be developing provide the following functions: [Denying access to Pico's internal files](#denying-access-to-picos-internal-files), [configuring PHP](#php-configuration), and [setting up Pico's URL rewriting](#url-rewriting).  Although it's arguably the most important function, we'll be configuring URL rewriting last due to the order that Nginx processes its config.  Having this said: The order of your rules do matter in Nginx, so make sure that the rules have the same order as we discuss them in this document.

## General Server Configuration

Nginx's server configuration is broken into blocks.  Each website has its own `server` block inside your Nginx config.  While configuration of Nginx sites, virtual hosts, and other aspects of this topic are outside the scope of this guide, we'll provide enough to at least get you started with Pico.

```
server {
	listen 80;

	server_name www.example.com example.com;
	root /var/www/html/pico;

	index index.php;
}
```

Let's break down this example.  The first line, `listen 80` tells Nginx to listen on port 80 for incoming connections.  This is the default port used by web traffic, and what you'll want to use 99% of the time.  `server_name` is where you specify what domain name or names match this configuration.  You'll likely want to include your domain both with and without the `www.` subdomain.  `root` lets you specify the Document Root for this site.  This is usually going to be where you've installed Pico, but ultimately it depends on your configuration.  `index index.php` tells Nginx that your site's index file will be called `index.php`.

We'd highly recommend you consider securing your website using HTTPS.  Using HTTPS will provide your users with extra protection, and prevent third parties from being able to snoop on their web traffic.  Unfortunately, configuring SSL is out of the scope of this document, but we'd be happy to point you in the right direction.  For an easy to set up SSL certificate, you can check out [Let's Encrypt](https://letsencrypt.org/) and [this tutorial](https://www.digitalocean.com/community/tutorials/how-to-secure-nginx-with-let-s-encrypt-on-ubuntu-14-04).  For more general information, see Nginx's own documentation about [configuring HTTPS servers](https://nginx.org/en/docs/http/configuring_https_servers.html).

Below we'll add a few more sections to this server configuration.  All of the following examples should be placed **inside** the server block, before the closing `}`.

## Denying Access to Pico's Internal Files

One of the important features provided by Pico's `.htaccess` file is the denial of access to some of Pico's internal files and directories.  There are two simple reasons for this, added security, and ease-of-use.  We don't want anyone snooping around and reading files they shouldn't be (like your `config/config.yml`), and we also don't really want people navigating to raw Markdown files.

You should always use the following rule in the Nginx config of your Pico site:

```
location ~ ^/((config|content|vendor|composer\.(json|lock|phar))(/|$)|(.+/)?\.(?!well-known(/|$))) {
	try_files /index.php$is_args$args =404;
}
```

This rule returns Pico's `404 Not Found` page if someone tries to access one of Pico's internal files like `config/config.yml`.  We recommend this as it's generally a good practice.  Users's don't need access to these files, so why allow it?

In order to keep configuration simple, the above example assumes that you've installed Pico to the Document Root of your site.  If you've rather installed Pico to a subfolder, you'll have to use the path starting from your site's Document Root to Pico's installation directory.  So, if your Document Root is `/var/www/html`, and you've installed Pico to `/var/www/html/pico`, your rule should match `^/pico/` instead of `^/` (lines `1` and `2`).

```
location ~ ^/pico/((config|content|vendor|composer\.(json|lock|phar))(/|$)|(.+/)?\.(?!well-known(/|$))) {
	try_files /pico/index.php$is_args$args =404;
}
```

## PHP Configuration

This is a topic outside the realm of this document.  Unlike Apache (which sends documents to PHP in most default configurations automatically), Nginx needs to be *told* to send a file to an external PHP processor.

Configuring PHP is a topic that will differ slightly depending on the OS you are using.  The examples we provide here apply to Ubuntu 14.04, but they also require external configuration of `php-fpm` or another PHP processor.

Your PHP configuration will look something like this:

```
location ~ \.php$ {
	try_files $uri =404;

	fastcgi_pass unix:/var/run/php5-fpm.sock;
	fastcgi_index index.php;
	include fastcgi_params;

	# Let Pico know about available URL rewriting
	fastcgi_param PICO_URL_REWRITING 1;
}
```

Please note that this is only provided as an **example**.  You should write your own PHP location block based on your personal system configuration.

This `location` rule tells Nginx to send all pages ending in `.php` to an external PHP processor called `php-fpm`.  Again, setting this up is outside the scope of this document.  There are many tutorials available online.  Here is one for [Ubuntu 14.04](https://www.digitalocean.com/community/tutorials/how-to-install-linux-nginx-mysql-php-lemp-stack-on-ubuntu-14-04#3-install-php-for-processing).

By default, `php-fpm` comes with a very insecure setting that can allow unauthorized code execution.  We've included a small `try_files` statement here that will protect you from this vulnerability.

The line `fastcgi_param PICO_URL_REWRITING 1;` informs Pico that URL rewriting is available, so you won't have to explicitly set `rewrite_url: true` in your `config/config.yml`.

## URL Rewriting

Setting up Pico's URL rewriting sometimes is a bit challenging in Nginx.  Depending on whether you've installed Pico to your site's Document Root or a subfolder, the rules differ.

```
location / {
	try_files $uri $uri/ /index.php$is_args$args;
}
```

This rule tells Nginx that whenever it looks up a URL within your site, it should first look for a file of that name, then a directory, and finally, if neither exist, it should rewrite the URL for Pico.  When the URL is rewritten for Pico, the request is passed to `index.php`.  Nginx will load `index.php` and pass the rendering along to your PHP processor.

If you've rather installed Pico to a subfolder, configuration is slightly more complicated, but should be easy if you follow along.  Rather than searching for just `/`, you'll have to use the path starting from your site's Document Root to Pico's installation directory.  So, if your Document Root is `/var/www/html`, and you've installed Pico to `/var/www/html/pico`, your rule should match `/pico/` instead of `/` (lines `1` and `2`).

```
location /pico/ {
    try_files $uri $uri/ /pico/index.php$is_args$args;
}
```

## Example Configuration

If we combine all the examples above, your configuration will look something like this:

```
server {
	listen 80;

	server_name www.example.com example.com;
	root /var/www/html/pico;

	index index.php;

	location ~ ^/((config|content|vendor|composer\.(json|lock|phar))(/|$)|(.+/)?\.(?!well-known(/|$))) {
		try_files /index.php$is_args$args =404;
	}

	location ~ \.php$ {
		try_files $uri =404;

		fastcgi_pass unix:/var/run/php5-fpm.sock;
		fastcgi_index index.php;
		include fastcgi_params;

		# Let Pico know about available URL rewriting
		fastcgi_param PICO_URL_REWRITING 1;
	}

	location / {
		try_files $uri $uri/ /index.php$is_args$args;
	}
}
```

Again, please note that this is only provided as an **example**.  You should not copy it, but only use it as a reference.  Your own configuration will depend very much on your own system configuration.  If you do copy this code, be sure to modify it according to your own requirements.
