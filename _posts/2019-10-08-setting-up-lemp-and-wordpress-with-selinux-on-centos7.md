---
layout: post
title: "Setting up LEMP and Wordpress with SELinux on CentOS 7"
date: 2019-10-08 12:00
categories: Software
tags: [wordpress, selinux, nginx, mariadb]
---

Part 2: [LEMP WordPress Next Steps - Certbot and Caching]({% post_url 2019-10-10-LEMP-wordpress-next-steps-certbot-and-caching %})

Part 3: [Send Emails from WordPress with Google GMail (OAuth)]({% post_url 2019-10-12-send-emails-from-wordpress-with-google-oauth %})

I recently went through the process of setting up a blog for my wife. She had used blogspot in the past, but wanted something a bit more customizable. We considered hosted WordPress, but I figured for $5 a month I could do better myself and have more control. Plus, where is the fun in a hosted solution, you can always learn more by doing it yourself.

I’ve used shared hosts in the past and had issues with performance. You also don’t have complete control. For the past 10 months, I’ve been using AWS at work and initially looked at setting something up in AWS. However, when its you paying the bill and not the company, all those nice features and services (managed databases, ALBs, and even EC2s) really start to add up. The decision came down to AWS Lightsail or a DigitalOcean droplet. Technically, the smallest Lightsail option was cheaper, but ultimately I wanted experience with another provider and ended up going with a $5/month Droplet from [DigitalOcean](https://m.do.co/c/dc21a836fd4f).

## Provisioning a DigitalOcean Droplet

Creating an account and setting up a droplet is fairly straight forward. If you don’t already have a DigitalOcean account, create one. You can use my [referral link](https://m.do.co/c/dc21a836fd4f) to get a bonus credit. At the time of writing, I am using a `CentOS 7.6` image running on a 1 GB / 1 CPU droplet. The droplet can always be scaled up in the future if necessary.


## Setting up DNS

I’m not going to go into much detail here. Register a domain and point the nameservers at DigitalOcean. In your DigitalOcean account, add the domain and create two `A` records. The first with a hostname of `@` which points to your Droplet. The second is with a hostname of `www` that also points to your Droplet. Let the DNS propagate as you continue with the following steps.


## SELinux - Don't Disable It

SELinux stands for Security Enhanced Linux. In a nutshell, it expands the standard owner/group permission controls with the idea of contexts. For example, your nginx web server shouldn’t be modifying files in `/etc` or listening on random ports. Each file, process and so on is assigned a context. You explicitly allow the desired behavior by changing context, SELinux booleans, or writing custom policies.

Many of the articles and guides I came across advocated to simply disable SELinux. While SELinux can make things more difficult, the additional protections and security are worth the effort. Rather than disabling it because its hard, why not take the time to learn something new and grow your skill set? Any changes for things to operate with SELinux enforcing will be noted below in the
following steps.

SELinux could be an article (or more) all in itself. But some useful commands to know at this point are `ls -Z` and `ps -Z`. These commands will allow you to see the context of files and processes.


## Configuring yum-cron for automatic updates

After logging into your server for the first time, you should make sure everything is updated.

```console
yum update -y
```

Unless you are going to login and run this command every day (you aren’t), you need to setup some way to automatically apply updates. In this case, I am going to use yum-cron and enable automatic updates for security updates only. This won’t break things like PHP version or MariaDB version, but still apply security updates.

```console
yum install yum-cron -y
```

Customize the configuration in `/etc/yum/yum-cron.conf`. The following lines should be changed:

```console
update_cmd = security

apply_updates = yes
```

Enable and start the service and ensure its running. The status should show as green (active).

```console
systemctl enable yum-cron --now

systemctl status yum-cron
```

You can check the logs and see what packages were updated with the following two commands:

```console
cat /var/log/cron | grep yum-daily

cat /var/log/yum.log
```

## Installing NGINX

I am installingthe latest version of nginx from their repo. Create the file `/etc/yum.repos.d/nginx.repo` and add the following block:

```ini
[nginx]
name=nginx repo
baseurl=http://nginx.org/packages/mainline/centos/7/$basearch/
gpgcheck=0
enabled=1
```

Install the package and enable/start:

```console
yum install nginx -y

systemctl enable --now nginx
```

At this point, you should be able to see the nginx welcome page by hitting the droplet’s IP address in your web browser.

By default, nginx will only allow upload (body) size of 1MB. We are going to increase this to allow larger files to be uploaded.

**Note**: A similar change is needed in PHP and documented below in the
PHP section.

Edit the file `/etc/nginx/nginx.conf` and update the line
`client_max_body_size`. I’ve set this value to `20m`


## Install firewalld

Is firewalld necessary? No. But its another one of those things that you should be doing. It's a tool that’s useful to learn and helps keep things secured. You could configure the firewall at the DigitalOcean droplet level (or security group in AWS), but we are going to use the `firewalld` package in this case.

```console
yum install firewalld -y

systemctl enable firewalld --now
```

And just like that, you’ve broken your site and can no longer access it. Try hitting your Droplet’s IP address in your browser again. Firewalld in itself could be an entire post, but for now just enough to get you back up and running. You need to let the firewall know you are hosting a website, that you want to allow HTTP and HTTPS.

```console
firewall-cmd --add-service http --permanent

firewall-cmd --add-service https --permanent

firewall-cmd --reload
```

Hit your Droplet’s IP in the browser again and you should see the nginx welcome page. Success! We’ve fixed what we broke.


## Installing MariaDB

Similar to nginx, we are going to install the latest version of MariaDB from their repo. Create the file `/etc/yum.repos.d/mariadb.repo` and add the following block:

```ini
[mariadb]
name = mariadb
baseurl = http://yum.mariadb.org/10.4/centos7-amd64
gpgkey=https://yum.mariadb.org/RPM-GPG-KEY-MariaDB
gpgcheck=1
```

Install the server and client and start/enable the server

```console
yum install MariaDB-server MariaDB-client -y

systemctl enable mariadb --now
```

Secure the installation using the provided tool.

```console
mysql_secure_installation

Enter current password for root (enter for none): <press enter>
Switch to unix_socket authentication [Y/n] y
Change the root password? [Y/n] y
Remove anonymous users? [Y/n] y
Disallow root login remotely? [Y/n] y
Remove test database and access to it? [Y/n] y
Reload privilege tables now? [Y/n] y
```


## Installing PHP

The version of PHP provided in CentOS repos (even EPEL) is quite old. I’m going to use REMI REPO to install `PHP 7.3`

```console
yum install -h epel-release yum-utils

yum install -y http://rpms.remirepo.net/enterprise/remi-release-7.rpm

yum-config-manager --enable remi-php73

yum install php php-common php-opcache php-mcrypt php-cli php-gd php-curl php-mysqlnd

yum install php-fpm

php -v
```

Customize the configuration for php-fpm at `/etc/php-fpm.d/www.conf`. The lines should be updated to match the following:

```
user = nginx
group = nginx

listen = /run/php-fpm/www.sock

listen.owner = nginx(be sure to uncomment)
listen.group = nginx(be sure to uncomment)
```

Change group on `/var/lib/php` from `apache` to `nginx`

```console
chown -R root:nginx /var/lib/php
```

Disable PHP’s `cgi.fix_path` option. Edit the `/etc/php.ini` file and add the following line under the section discussing `cgi.fix_path`

```
cgi.fix_pathinfo = 0
```

By default, PHP will only allow you to upload files 2mb and smaller. We are going to increase this so larger images and files can be uploaded. You can set this value to whatever you think is appropriate. Edit `/etc/php.ini` and look for the line setting `upload_max_filesize`. I’ve set this to `20M` to match the nginx update we made above.

Start and enable the process

```console
systemctl enable php-fpm --now
```


## Installing WordPress

The end is in sight! At this point we have a secure LEMP setup and just need to add in WordPress.

### Setup the Database

The first step is to setup the database and user. Customize the `my_` values to suit your needs. For example, `my_wp` is the name of the database you are creating. Be sure to customize` my_user` and `my_password` as well. These values will be provided to the WordPress installation wizard in the following step.

This will create a database named `my_wp` within the database server. The user `my_user` (with password `my_password`) will have full access to the `my_wp` database.

```console
mysql -u root

CREATE DATABASE my_wp CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;
Query OK, 1 row affected (0.001 sec)

GRANT ALL ON my_wp.* TO 'my_user'@'localhost' IDENTIFIED BY 'my_password';
Query OK, 0 rows affected (0.002 sec)

FLUSH PRIVILEGES;
Query OK, 0 rows affected (0.001 sec)

exit
```

### Setup Directories and Download WordPress

Create the directory for your website (customize `my_site` to fit your needs)

```console
mkdir -p /var/www/html/my_site
```

Download latest version of wordpress

```console
curl -o /tmp/wordpress.tar.gz https://wordpress.org/latest.tar.gz
```

Extract WordPress to your site directory (the `--strip-components` option removes the wordpress directory and installs it into the root of your site directory).

```console
tar -xzvf /tmp/wordpress.tar.gz -C /var/www/html/my_site/ --strip-components=1
```

Set ownership on the files we just extracted

```console
chown -R nginx: /var/www/html/my_site/
```

Now for SELinux… During the WordPress installation wizard, it will attempt to create the `wp-config.php` file in `/var/www/html/my_site`. However, the SELinux context for that path is `httpd_sys_content_t` which does not allow for writes. We will temporarily change the context for this directory during installation.

```console
chcon -t httpd_sys_rw_content_t /var/www/html/my_site/
```


### Setup NGINX Configuration

The following configuration is a simple starting point. In the next part of this series, I’ll discuss setting up `certbot` for free TLS certificates utilizing Lets Encrypt, REDIS cache, fast-cgi page cache and more. The configuration and structure of these files will change.

Create a new file `/etc/nginx/conf.d/my_site.conf` and insert the following block (customized for your specific case)

```
server {
    listen 80;
    server_name my-site.com www.my-site.com;

    root /var/www/html/my_site;
    index index.php;

    # log files
    access_log /var/log/nginx/my_site.access.log;
    error_log /var/log/nginx/my_site.error.log;

    location = /favicon.ico {
        log_not_found off;
        access_log off;
    }

    location = /robots.txt {
        allow all;
        log_not_found off;
        access_log off;
    }

    location / {
        try_files $uri $uri/ /index.php?$args;
    }

    location ~ \.php$ {
        try_files $uri =404;
        fastcgi_pass unix:/run/php-fpm/www.sock;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }

    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg)$ {
        expires max;
        log_not_found off;
    }
}
```

Access your site (e.g. my-site.com) and complete the WordPress wizard. Use the database name and user/pass created earlier.


### Restore the SELinux Context

After completing the wizard, restore the SELinux context that we changed previously. The `restorecon` command will use the default context set for the file based on its path.

```console
restorecon -v /var/www/html/my_site/
```

## Conclusion

Yeah, this article got a bit long. But at this point you should have a relatively secure WordPress installation running on a DigitalOcean Droplet. The performance at this point should blow away a shared host.

In the next part of this series, I’ll discuss some next steps you should take. This includes things like TLS certs (for HTTPS) using LetsEncrypt and certbot, setting up a REDIS object cache, and setting up fast-cgi page cache.