---
layout: post
title: "LEMP WordPress Next Steps - Certbot and Caching"
date: 2019-10-10 12:00
categories: Software
tags: [wordpress, selinux, nginx, mariadb, redis]
---

Part 1: [Setting up LEMP and Wordpress with SELinux on CentOS 7]({% post_url 2019-10-08-setting-up-lemp-and-wordpress-with-selinux-on-centos7 %})

Part 3: [Send Emails from WordPress with Google GMail (OAuth)]({% post_url 2019-10-12-send-emails-from-wordpress-with-google-oauth %})

After completing the first part of this series ([Setting up LEMP and WordPress with SELinux on CentOS 7]({% post_url 2019-10-08-setting-up-lemp-and-wordpress-with-selinux-on-centos7 %})), we are now ready to optimize our installation. The only step I would consider mandatory is TLS certificates with certbot. The caching stuff is nice to have, but really no reason not to set it up.


## Free TLS with LetsEncrypt And Certbot

The [LetsEncrypt](https://letsencrypt.org/) service provides free TLS certificates for you site. The certificates expire in 90 days, but the `certbot` tool makes renewing them a breeze.

### Install the tool

```console
yum install certbot-nginx -y
```

### Create the certificate

Run the command to provision certificates. In this case, I'm running this on the same machine that is hosting the website. This command will complete the challenges necessary to prove you own the domain as well as save the certificate files and update your nginx configuration. Include both the domain with `www` as well as without - this will create a SAN on the certificate.

```console
certbot --nginx -d my-site.com -d www.my-site-com
```

* Provide an email for expiration notices
* Type A to agree to terms
* Type Y or N to share your email with EFF
* Type 2 to setup redirection (force HTTP to HTTPS)

At this point, your nginx configuration (`/etc/nginx/conf.d/my_site.conf`) will be updated to create a 301 redirection to HTTPS. In addition, you will see some additional lines added for `ssl_certificate` , `ssl_certificate_key`, etc. All of these changes will be indicated with "managed by Certbot" comments.

Navigate to your website (e.g. my-site.com) and you should be redirected to HTTPS. The browser should show the site as secure and provide the TLS certificate information if you click the lock icon in the address bar.


### Setup Automatic Renewal

Your site will be good until the certificate expires 90 days later. Luckily, you can easily renew the certificate with the `certbot renew` command. This command checks all certificates (based on information found in `/etc/letsencrypt`) and renews any which will expire in 30 days or less. We are going to use cron to run the command daily to check if the certificate is ready for renewal. Add the following line to your crontab to run the command each day at 2am. Results are appended to the log.

```
0 2 * * * /usr/bin/certbot renew >> /var/log/certbot_renew.log
```


## Configure Redis Object Cache

Redis provides an in memory key/value cache to offload calls to the database. Building WordPress pages involves lots of calls to the database – getting theme data and config, post data, comment data, etc. Each of these calls is expensive in terms of render time. A Redis cache allows this data to be stored in memory.

You can use a dedicated Redis server or the Redis service from DigitalOcean. However, for this article, I’m just going to run Redis on the Droplet alongside nginx, PHP and MariaDB. As seen in the first part, the CentOS 7 packages are quite a bit out of date (`3.2.12-2` vs `5.0.6-1`).

Redis could be installed from source, but I’m going to make use of the Remi Repo.

```console
yum -y install http://rpms.remirepo.net/enterprise/remi-release-7.rpm

yum --enablerepo=remi install redis -y

systemctl enable redis --now
```

Install the necessary PHP module for Redis and restart

```console
yum install php-redis -y

systemctl restart php-fpm
```

If you are planning on hosting multiple WordPress sites on the Droplet making use of the same Redis cache, you need to add a cache key to your `wp-config.php` file. This is so the keys in the cache can be differentiated between WordPress sites. Add the following line to your `wp-config.php` file (remember to set `my_site` accordingly):

```
define( 'WP_CACHE_KEY_SALT', 'my_site' );
```

Next, install the [Redis Object Cache plugin](https://wordpress.org/plugins/redis-cache/) for WordPress and activate it. Everything seems to be working, the cache shows as enabled. However, if you look closely, the status will show **Not Connected**. What to blame? Yep, SELinux.

If you tail the audit log with `tail -f /var/log/audit/audit.log` and try to enable the cache, you should get something similar to the following:

```
type=AVC msg=audit(1570563750.741:1714): avc:  denied  { name_connect } for  pid=12572 comm="php-fpm" dest=6379 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:redis_port_t:s0 tclass=tcp_socket permissive=0

type=SYSCALL msg=audit(1570563750.741:1714): arch=c000003e syscall=42 success=no exit=-13 a0=6 a1=7fd567865200 a2=10 a3=5d9ce6a6 items=0 ppid=12570 pid=12572 auid=4294967295 uid=997 gid=994 euid=997 suid=997 fsuid=997 egid=994 sgid=994 fsgid=994 tty=(none) ses=4294967295 comm="php-fpm" exe="/usr/sbin/php-fpm" subj=system_u:system_r:httpd_t:s0 key=(null)

type=PROCTITLE msg=audit(1570563750.741:1714): proctitle=7068702D66706D3A20706F6F6C20777777
```

What to make of this? The `php-fpm` process is getting denied trying to open a TCP socket to the Redis port (6379). Let’s see what ports nginx can use:

```console
semanage port -l | grep http_port_t

http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
```

As you can see, the port `6379` is not in that list. We can add it with the following command (modify the `http_port_t` context to include 6379/tcp):

```console
semanage port -m -t http_port_t -p tcp 6379
```

Now if you watch `redis-cli monitor` as you browse the blog, you should see the operations happening against Redis.


## Configure fast-cgi Page Cache

In the previous section, we enabled Redis (in-memory key/value store) for database objects. However, each time a page is requested, PHP is executed in order to render the page. What if we could store this rendered page in memory as well and bypass PHP all together when it is viewed a second time. Enter fast-cgi caching.

There are some things to keep in mind. If you update a post, you want to refresh the cache. If a comment is made, you want to refresh the cache. You get the idea. Any time data is changed, you need to clear the cache.

As mentioned in the previous post, we are going to reorganize our nginx configuration. This allows us to separate out some functionality as well as making it easier to host multiple WordPress sites from a single server.

### Create the Cache

First, lets create a cache to store the rendered pages. Create a new file (`/etc/nginx/conf.d/caching.conf`) and add the following content:

```
fastcgi_cache_path /var/run/nginx-cache levels=1:2 keys_zone=WORDPRESS:100m inactive=60m max_size=300m use_temp_path=off;
fastcgi_cache_key "$scheme$request_method$host$request_uri";
```

* Creates a cache located at `/var/run/nginx-cache`. On CentOS, `/var/run` is on tempfs so keep in mind the amount of memory available on your server.
* The `levels=1:2` controls the directory structure create within the cache directory. Leave this as is.
* The `keys_zone=WORDPRESS` names the cache which is referenced below in the WordPress snippet. The `100m`` sizes the cache at 100m.
* The `inactive=60m` sets the time to live for cached objects. After 60 minutes, the cached object will be removed regardless if its still fresh or not.
* The `max_size=300m` sets the maximum size the cache can grow to. Once the size is reached, the least recently used data is removed to make room for new data.
* The `use_temp_path=off` option says to bypass temporary files and write the data directly into the cache.
* Finally, the `fastcgi_cache_key` line determines how data in the cache is keyed.


### Create Snippet to be used by all WordPress Sites

Next, lets create a snippet for WordPress that can be reused. Create a new file at `/etc/nginx/snippets/wordpress.conf`. The comments in the file should explain what each section is doing. Notice that `fastcgi_cache WORDPRESS;` matches the cache we crated above.

```
# Disable logging for favicon and robots.txt
location = /favicon.ico {
    log_not_found off;
    access_log off;
}

location = /robots.txt {
    allow all;
    log_not_found off;
    access_log off;
    try_files $uri /index.php?$args;
}

# Deny all attempts to access hidden files such as .htaccess, .htpasswd, .DS_Store (Mac).
location ~ /\. {
    deny all;
}


# Deny access to any files with a .php extension in the uploads directory
location ~* /(?:uploads|files)/.*\.php$ {
    deny all;
}

###########
# Caching # 
###########

# set a flag to skip cache (based on rules below)
set $skip_cache 0;

# POST requests and should always go to PHP
if ($request_method = POST) {
  set $no_cache 1;
}

# Requests with query strings should always go to PHP
if ($query_string != "") {
  set $skip_cache 1;
}

# Don't cache uris containing the following segments
if ($request_uri ~* "(/wp-admin/|/xmlrpc.php|/wp-(app|cron|login|register|mail).php|wp-.*.php|/feed/|index.php|wp-comments-popup.php|wp-links-opml.php|wp-locations.php|sitemap(_index)?.xml|[a-z0-9_-]+-sitemap([0-9]+)?.xml)") {
  set $skip_cache 1;
}

# Don't use the cache for logged in users or recent commenters
if ($http_cookie ~* "comment_author|wordpress_[a-f0-9]+|wp-postpass|wordpress_no_cache|wordpress_logged_in") {
  set $skip_cache 1;
}

# WordPress single site rules.
location / {
    try_files $uri $uri/ /index.php?$args;
}

# Add trailing slash to */wp-admin requests.
rewrite /wp-admin$ $scheme://$host$uri/ permanent;

# Directives to send expires headers and turn off 404 error logging.
location ~* ^.+\.(eot|otf|woff|woff2|ttf|rss|atom|zip|tgz|gz|rar|bz2|doc|xls|exe|ppt|tar|mid|midi|wav|bmp|rtf)$ {
    access_log off; log_not_found off; expires max;
}

# Media: images, icons, video, audio send expires headers.
location ~* \.(?:jpg|jpeg|gif|png|ico|cur|gz|svg|svgz|mp4|ogg|ogv|webm)$ {
  expires 1M;
  access_log off;
  add_header Cache-Control "public";
}

# CSS and Javascript send expires headers.
location ~* \.(?:css|js)$ {
  expires 1M;
  access_log off;
  add_header Cache-Control "public";
}

# HTML send expires headers.
location ~* \.(html)$ {
  expires 7d;
  access_log off;
  add_header Cache-Control "public";
}

# Browser caching of static assets.
location ~* \.(jpg|jpeg|png|gif|ico|css|js|pdf)$ {
  expires 7d;
  add_header Cache-Control "public, no-transform";
}

# Enable Gzip compression in NGNIX.
gzip on;
gzip_disable "msie6";
gzip_types text/plain text/css application/json application/javascript application/x-javascript text/xml application/xml application/xml+rss text/javascript image/svg+xml;

# Pass all .php files onto a php-fpm/php-fcgi server.
location ~ \.php$ {
    include fastcgi_params;
    fastcgi_pass unix:/run/php-fpm/www.sock;

    fastcgi_cache WORDPRESS;
    fastcgi_cache_valid 200 301 302 60m;
    fastcgi_cache_use_stale error timeout updating invalid_header http_500 http_503;
    fastcgi_cache_lock on;
    add_header X-FastCGI-Cache $upstream_cache_status;
    fastcgi_cache_bypass $skip_cache;
    fastcgi_no_cache $skip_cache;

    fastcgi_index index.php;
    fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
}

```

### Update Site Configuration

Finally we have to update the site configuration to make use of our new snippet. We are going to update the configuration file created in the previous part. Edit `/etc/nginx/conf.d/my_site.conf` to replace all the WordPress specific options with a simple `include snippets/wordpress.conf;` Your file should look something like the following:

```
server {
    server_name my-site.com www.my-site.com;

    root /var/www/html/my_site;
    index index.php;

    # log files
    access_log /var/log/nginx/my_site.access.log;
    error_log /var/log/nginx/my_site.error.log;

    include snippets/wordpress.conf;

    listen 443 ssl; # managed by Certbot
    ssl_certificate /etc/letsencrypt/live/my-site.com/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/my-site.com/privkey.pem; # managed by Certbot
    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot

}
server {
    if ($host = www.my-site.com) {
        return 301 https://$host$request_uri;
    } # managed by Certbot

    if ($host = my-site.com) {
        return 301 https://$host$request_uri;
    } # managed by Certbot


    listen 80;
    server_name test.diymakerhub.com;
    return 404; # managed by Certbot
}
```

### Install NGINX Helper Plugin

Remember all those scenarios when we needed to refresh the cache? There’s a plugin for that. Install the [NGINX Helper plugin](https://wordpress.org/plugins/nginx-helper/) and activate it.

Head over to the settings page for the plugin (Settings > NGINX Helper) and make the following changes:

* Check the `Enable Purge` box
* Leave `nginx Fastcgi cache` checked, do **not** select Redis here
* If `Purge Method` section is not showing up, select `Redis` under `Caching Method` and then flip it back to `nginx Fastcgi cache`
* Under `Purge Method`, select `Delete local server cache files`
* Check all the options under `Purging Conditions`
* Click the [Save All Changes] button

### Testing the Cache

You can test the cache with the developer tools of your browser by looking for the `X-FastCGI-Cache` header. This header was set in the snippet we created above. Go ahead and clear the cache (big red button) on the NGINX Helper plugin settings page. Logout of your WordPress site. In Chrome, open developer tools and select the **Network** tab. Navigate to your page and check the response headers. You should see `X-FastCGI-Cache: MISS`. Refresh the page and this should change to HIT.

If you login and do the same check, you should see `X-FastCGI-Cache: BYPASS`


## Conclusion

If you’ve made it this far, you now have a TLS with HTTP -> HTTPS redirection setup, certbot automatically renewing certificates, and both a Redis object cache and fastcgi page cache setup and running. With this, you should get some good mileage out of the $5 DigitalOcean Droplet (or other small sized VPS).