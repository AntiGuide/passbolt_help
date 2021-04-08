---
title: Install Passbolt CE on FreeNAS
card_title: FreeNAS
card_teaser: Step by step guide to install passbolt CE on FreeNAS
card_position: 7
date: 2021-02-25 00:00:00 Z
description: How to install passbolt CE on FreeNAS.
icon: fa-server
categories: [hosting,install,ce]
sidebar: hosting
layout: default
slug: freenas
permalink: /:categories/:slug.html
---

{% include layout/row_start.html %}
{% include layout/col_start.html column="7" %}

## Introduction

This tutorial was written for FreeNAS with jails. There might be minor differences to a pure FreeBSD install but you should mostly be able to follow the same guide.

We will assume that you install this on your local NAS and that the NAS is not exposed to the internet. If you want to do that you will have to get SSL certificates (refer to [Let's Encrypt](https://letsencrypt.org/)) and read into securing your Apache server etc.

## Preparation

If you want to use the shell on your local machine instead of the FreeNAS web shell you can ssh in like this
```shell
ssh root@Your-FreeNAS-IP
iocage console Jail-Name
```

You will need an editor for a few config files. You can use the preinstalled vi or use
```shell
pkg install nano
```
to be able to use the exact commands from this guide.

## Installation steps

### 1. Create a web server matching the system requirements.

Use these commands to install Apache, PHP, MariaDB, Git, GnuPG and the needed PHP extensions:

```shell
pkg install apache24 php73 mariadb103-server php73-composer2 git gnupg mod_php73 php73-session php73-pecl-gnupg php73-intl php73-mbstring php73-simplexml php73-gd php73-pecl-imagick-im7 php73-mysqli php73-pdo php73-pdo_mysql php73-xsl php73-phar php73-posix php73-xml php73-zlib php73-ctype php73-curl php73-json php73-ldap
```

Activate a PHP config preset:

```shell
cp /usr/local/etc/php.ini-production /usr/local/etc/php.ini
```

Edit the php.ini

```shell
nano /usr/local/etc/php.ini
```

Add:

```shell
extension=gnupg
```

Create directories and generate a self signed certificate:

```shell
mkdir /usr/local/etc/ssl/private /usr/local/etc/ssl/certs
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /usr/local/etc/ssl/private/selfsigned.key -out /usr/local/etc/ssl/certs/selfsigned.crt
```

Edit the Apache config:

```shell
nano /usr/local/etc/apache24/httpd.conf
```

Uncomment and set the ServerName to Your-Jail-IP:80

Uncomment
```shell
LoadModule socache_shmcb_module libexec/apache24/mod_socache_shmcb.so
LoadModule ssl_module libexec/apache24/mod_ssl.so
LoadModule rewrite_module libexec/apache24/mod_rewrite.so
Include etc/apache24/extra/httpd-ssl.conf
```

And set the AllowOverride option to All
```shell
<Directory "/usr/local/www/apache24/data">
...
AllowOverride All
...
</Directory>
```

Setup SSL for your Apache server:

```shell
nano /usr/local/etc/apache24/extra/httpd-ssl.conf
```

Set the following options:

```shell
ServerName Your-Jail-IP:443
ServerAdmin Your-EMail-Address
SSLCertificateFile "/usr/local/etc/ssl/certs/selfsigned.crt"
SSLCertificateKeyFile "/usr/local/etc/ssl/private/selfsigned.key"
```

Enable and start the Apache server:

```shell
sysrc apache24_enable="yes"
service apache24 start
```

Enable, start and install the MariaDB server:

```shell
sysrc mysql_enable=yes
service mysql-server start
mysql_secure_installation
```

Enable PHP usage for Apache by creating a config:

```shell
nano /usr/local/etc/apache24/Includes/php.conf
```

```shell
<IfModule dir_module>
    DirectoryIndex index.php index.html
    <FilesMatch "\.php$">
        SetHandler application/x-httpd-php
    </FilesMatch>
    <FilesMatch "\.phps$">
        SetHandler application/x-httpd-php-source
    </FilesMatch>
</IfModule>
```

### 2. Create an empty database

Connect to your MariaDB server and create new database. The password is the one you set during mysql_secure_installation.

```shell
mysql -u root -p -e "CREATE DATABASE passbolt CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"
```

### 3. Clone the repository

Cloning the code using git will allow you to keep the source under version control and facilitate
subsequent updates.

```shell
cd /usr/local/www/apache24/data
rm index.html
git clone https://github.com/passbolt/passbolt_api.git
mv passbolt_api passbolt
```

### 4. Generate an OpenPGP key

Passbolt API uses an OpenPGP key for the server in order to authenticate and sign the outgoing JSON requests.
For improved compatibility we recommend that you use the same GnuPG version for generating the keys and for the 
php module. 

{% include hosting/install/warning-gpg-key-generation.html %}

```shell
gpg --pinentry-mode loopback --full-gen-key
```

After creating the key make sure you note down the fingerprint, it will be requested later in the install process.
You can get the server key fingerprint as follow:

```shell
gpg --list-keys | grep -i -B 2 'SERVER_KEY@EMAIL.TEST'
```

Copy the public and private keys to the passbolt config location:

```shell
gpg --armor --export-secret-keys SERVER_KEY@EMAIL.TEST > /usr/local/www/apache24/data/passbolt/config/gpg/serverkey_private.asc
gpg --armor --export SERVER_KEY@EMAIL.TEST > /usr/local/www/apache24/data/passbolt/config/gpg/serverkey.asc
```

### 5. Prepare the www user and initialize the gpg keyring

To set the home directory and a shell of the www user execute:

```shell
pw usermod -d /usr/local/www -s /bin/csh -n www
chown -R www:www /usr/local/www/
```

Import the key and show it:

```shell
su -l www -c "gpg --import /usr/local/www/apache24/data/passbolt/config/gpg/serverkey_private.asc"
su -l www -c "gpg --list-keys"
```

Save the fingerprint for the step 7.

### 6. Install the dependencies

The project dependencies such as the plugin to manage the images, emails, etc. are not included anymore
in the code on the official repository. Fret not, composer will manage this for us.

```shell
cd /usr/local/www/apache24/data/passbolt
composer install --no-dev
```

### 7. Create a passbolt configuration file

Copy the standard passbolt config file and edit it:

```shell
cp config/passbolt.default.php config/passbolt.php
nano config/passbolt.php
```

Set the following options under 'App':

```shell
'fullBaseUrl' => 'https://Your-Jail-IP',
'base' => '/passbolt'
```

Set the following options under 'Datasources':

```shell
'username' => 'root',
'password' => 'Your-MariaDB-Password',
```

Set the following options under 'passbolt' => 'gpg' => 'serverkey':

```shell
'fingerprint' => 'Your-GPG-Fingerprint',
```

If you didn't save your fingerprint in step 5 go back there now and list your keys.

Also configure your E-Mail Account. Refer to the data you find about you mail provider when searching for configuring Thunderbird etc. You want to look for the SMTP Server (for sendign mails)

```shell
'host' => 'smtp.gmail.com',
'port' => 587,
'username' => 'Your-EMail@gmail.com',
'password' => 'Your-Password',
'tls' => true,
'from' => ['Your-EMail@gmail.com' => 'Passbolt'],
```

You can also set your configuration using environment variables.
Check `config/default.php` to get the names of the environment variables.

### 8. Run the install script

Make sure you run the installation script as the web server user:

```shell
su -l www -c "/usr/local/www/apache24/data/passbolt/bin/cake passbolt install"
```

Optionally you can also run the health check to see if everything is fine.

```shell
su -l www -c "/usr/local/www/apache24/data/passbolt/bin/cake passbolt healthcheck"
```

### 9. Setup the emails

For passbolt to be able to send emails, you must first configure properly the “EmailTransport” section in the 
`config/passbolt.php` file to match your provider SMTP details as layed out in step 1.

Emails are placed in a queue that needs to be processed by the following shell.
```bash
/usr/local/www/apache24/data/passbolt/bin/cake EmailQueue.sender
```

In order to have your emails sent automatically, you can add a cron call to the script so the emails 
will be sent every minute. Run the following command to edit the crontab for the www user:
```bash
crontab -u www -e
```

You can add a cron call to the script so the emails will be sent every minute. 
Add the following line to you crontab:
```bash
 * * * * * /var/www/passbolt/bin/cron >> /var/log/passbolt.log
```

If the log file does not yet exist, you can create it with the following command:
```bash
touch /var/log/passbolt.log && chown www:www /var/log/passbolt.log
```

And you are done!


### Troubleshooting

Here are some frequently asked questions related to passbolt installation:
{% include faq/list-by-tag.html tag='troubleshoot' %}

Feel free to ask for help on the [community forum](https://community.passbolt.com/c/installation-issues).

{% include date/updated.html %}

{% include layout/col_end.html %}
{% include layout/col_start.html column="4 last push1" %}


{% include aside/ce-install-community-forum-cta.md %}

{% include aside/ce-stay-up-to-date.md %}

{% include aside/ce-install-pro-cta.html %}

{% include layout/row_end.html %}
