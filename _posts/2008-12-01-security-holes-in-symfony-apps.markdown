---
layout: post
title:  "Security holes in symfony apps"
date:   2008-12-01 21:52:00
categories: symfony
---
<p>symfony 1.2 was just released, and came along with a brand new advent calendar. <a href="http://www.symfony-project.org/jobeet/1_2/en/01">The first day</a> mentions two ways to set up your web server to host a symfony project. <a href="http://www.symfony-project.org/jobeet/1_2/en/01#Web Server Setup: The ugly Way">The ugly way</a> seems to be very... ugly and crazy, putting your app directly in your webserver's root folder allows anyone to access all the files. But this security hole can be easily found in 'real' apps, as well as a few others...</p>
<p>
This article will try to enumerate some of the mistakes that are often done when a symfony application is put in production environment.
</p>

<h2>Development environment is still present</h2>
<p>These files, ending with _dev.php, are very useful when you are coding: they provide a lot of information on the configuration, the SQL queries that were executed, your PHP version etc. But on the other hand, they should never be accessible to your visitors (I mean, bad ones, you know, those who type in the darkness). They can show your kernel version, the symfony plugins which are installed, the escaping method you are using (or <i>not</i> using...), etc., which can lead to other vulnerabilities.<p>

<h2>The project root is the webserver's root folder</h2>
<p>This can also give a lot of information to an attacker, especially your database connection settings. Just search for 'web/frontend.php' in your favorite search engine, and you will have a bunch of apps which are potentially in this case. Then you just have to browse config/databases.yml and you get what you want. Just consider that if the application is unsecure, the webserver which is running it is also unsecure, and you can have a remote access to MySQL on port 3306, which will show you a lot of interesting things...</p>

<h2>The backend is not protected</h2>
<p>
Harder to find, but not impossible. Even if this page is not indexed by search engines, it's quite easy to see if a web app is powered by symfony, and the majority of these web apps have a backend which is called 'backend' (if not, just try 'admin' :)). So it's very important to restrict access to this backend, as it can contain sensitive information (user accounts, etc.). Imagine that you didn't make a recent backup of your database, and that someone deletes all its content...
</p>

<h2>How to fix it?</h2>
<p>
It's not difficult to avoid these problems. In fact, the solutions are very well explained in the official symfony book, but nonetheless they are often not applied.</p>
<ul>
 <li>Check that *_dev.php files are not uploaded (see the rsync_excludes.txt file for example)</li>
 <li>If you can, put the whole project (except the web/ folder) outside the web document root</li>
 <li>Protect your backend with sfGuard, or at least a htaccess/htpasswd file :) (and do not user admin/admin or something like this)</li>
</ul>
</p>