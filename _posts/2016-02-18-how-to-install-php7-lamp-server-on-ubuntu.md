---
layout: post
title:  "How to install PHP7 LAMP server on Ubuntu"
date:   2016-02-18 22:26:24 +0200
category: [tutorial, linux]
slug: "how-to-install-php7-lamp-server-on-ubuntu"
key: "php7-lamp"
tags: [php, linux]
description: >
  A tutorial on how to set up a LAMP stack with PHP7 on Ubuntu
---

First of all, add a custom `ppa` from which we will receive PHP7 packages:

    sudo add-apt-repository ppa:ondrej/php
    sudo apt-get update

Make sure you are making a clean install. We recommend following this guide if you're newly installing a LAMP stack.

We can now fetch some packages:

    sudo apt-get install apache2 php7.0 php7.0-fpm libapache2-mod-fastcgi

You may also opt in for some additional packages like:

    sudo apt-get install php7.0-mcrypt php7.0-gd


Then, we have to 'hook up' our PHP handler to Apache:

    sudo nano /etc/apache2/conf-available/php-fpm.conf

And paste this in:

    <IfModule mod_fastcgi.c>
      AddHandler php.fcgi .php
      Action php.fcgi /php.fcgi
      Alias /php.fcgi /usr/lib/cgi-bin/php.fcgi
      FastCgiExternalServer /usr/lib/cgi-bin/php.fcgi -socket /run/php/php7.0-fpm.sock -pass-header Authorization -idle-timeout 3600
      <Directory /usr/lib/cgi-bin>
        Require all granted
      </Directory>
    </IfModule>

To enable this `.conf` file, ubuntu apache has convenient shortcut:

    sudo a2enconf php-fpm.conf

Which just creates a shortcut to `/etc/apache/conf-available/php-fpm.con` in `/etc/apache/conf-enabled/php-fpm.conf`

Also, don't forget to enable these modules:

    sudo a2enmod rewrite actions

This enabled `mod_rewrite` and `mod_actions`.

## How to set up apache permission

Now, remember how every time someone sets up a LAMP stack or tries to install a WordPress plugin
the permission are always wrong?

Well, let me explain you how to set up permissions in an easy way.

Let's say that you are a user called `user1`. This may be your user on your local
development machine of an FTP user which will be upload WordPress, WordPress plugins, photos, etc.

In both of these use case we want to be able to create or upload new files and have them with the correct permissions
so that apache can execute them.

When we upload or create a file, the owner will naturally be `user1`. The group will be `user1`.

What we want is for both `user1` and `www-data` (apache) to be able to read and write the file.

To do this, we need a new group `www`:

    sudo groupadd www
    sudo usermod -a -G www www-data
    sudo usermod -a -G www user1

We created a new group and added `www-data` and `user1` to `www` group.

Now, here comes the magic bit:

You need to use this command when setting permissions:

    sudo chmod 2755 html/

Notice the number `2` at the beginning? This is called the **sticky bit**. It tells that any sub-files and sub-folders
in this folder should inherit the group.

This is what solves our permission problem. Basically if `www-data` or `user1` creates a file inside this folder,
that file will automatically has `www` group. This way, all users from this group will be able to read the file,
and be able to write the file if it has group write permission.

When `user1` creates a new file, the initial permissions for that file are `664` or `775` if it is a folder
(folders are `7` and `5` instead of `6` and `4` because they have **executable** permission, which allows users to
 navigate the folder `cd folder/`).

Apache, however, creates new files with `644` and `755` permissions. This may be bad for us, for example if we want to
delete cache files created by `apache` we won't be able to. To make `apache` have `664` and `775` permissions, you
need to change it's umask:

    sudo nano /etc/apache2/envvars

and paste this at the end of the file:

    # umask 002 to create files with 0664 and folders with 0775
    umask 002

Save it. Now for reasons unknown it may not take effect just by running `sudo apache2ctl restart`. You can try though.
If you're trying to install LAMP on local PC, I recommend restarting your PC. If you're running VPS... Well then try
different methods of restarting apache or restart the server.

Some insist that you have to stop it and start it (instead of using restart):

    sudo apache2ctl stop
    sudo apache2ctl start

## How to set up apache hosts

Now that we have that set up, we will need to set up websites.
If you're trying to run a single website, you'll need to edit
`000-default.conf` site configuration file:

    sudo su
    cd /etc/apache2/sites-available
    nano 000-default.conf

Inside this file, uncomment and write your website URL if you're running a VPS and a real domain that already points
to this server:

    ServerName mywebsite.com

Otherwise, the server will only respond if you enter the server IP in the browser. If you you're on local
development environment, then `127.0.0.1` and `localhost` will point to your local server.

Otherwise, if you're trying to set up multiple websites you'll need to copy the `.conf` file,
rename it and customize it for each website:

    sudo su
    cd /etc/apache2/sites-available
    cp 000-default.conf mysite.conf
    gedit mysite.conf

Paste this inside `<VirtualHost>`:

    ServerName mydevsite.dev # This may be a real domain .com or local domain .dev for development

    # Rhis specifies directives for the directory where our website files reside
    <Directory "/var/www/html">
      # This allows having .htaccess file in the website folder which overrides the directory for that website only
      Require all granted
      AllowOverride All

      Options +MultiViews # This allows entering URLs for directories with ending slashes '/' and without ''
                          # May be useful if you're running a Jekyll website, for example
    </Directory>

If you added a new `.conf` file for a new newsitem, you need to enable it:

    sudo a2ensite mysite
    sudo apache2ctl restart

If you only modified an existing configuration, you still need to reload the configuration:

    sudo apache2ctl restart

## MySQL

To install **MySql**, we have to install `mysql-server` package:

    sudo apt-get install mysql-server

Once you have that installed, it is recommended to run MySQL set up scripts:

    sudo mysql_install_db
    sudo mysql_secure_installation

First script sets up and initializes system table that MySQL will be using internally to manage users and other things,
second script allows you to secure installation by disallowing remove root login and removes test tables.

To have MySQL working with PHP, we need one more package:

    sudo apt-get install php7.0-mysql

That's it. You're now good to go. Additionally, you may opt-in for `phpMyAdmin` package, which will
help you work wil MySQL via web interface.

## phpMyAdmin setup

You have a couple of options install phpMyAdmin on your server.
phpMyAdmin is basically just a PHP app that connects and manages your
MySQL database.

You may install it using package manager:

  sudo apt-get install phpmyadmin

Benefits of installing phpMyAdmin this way:

  - package can be easily updated using `apt-get`
  - phpMyAdmin is installed in a secure location on the system
  - package creates custom apache directives, making it more secure

Another way to install phpMyAdmin is just downloading it from web:

  [phpmyadmin.net](https://www.phpmyadmin.net/)

You will a `.zip` which you can extract wherever you want.
If you're running a VPS, you can extract it to the root folder of your website,
making it accessible via `http://youwebsite.com/phpmyadmin`.
This is a great option if you want to manage your database along with
your website.

If you're setting up a local developments environment, a good way to go about it
would be to make a `pma/` folder in your **vhosts** directory (wherever you keep your website files),
add `127.0.0.1 pma` to `/etc/hosts/` and make a new website directive in
`/etc/apache2/sites-available/pma.conf`

This would make phpMyAdmin accessible via `http://pma/`,
which is pretty neat, I think.

P.S. You may need to install `php7.0-mbstring` package if you get an
error while trying to access phpMyAdmin. Some of these packages are required
to make phpMyAdmin work with translations.
