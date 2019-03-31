# Setup of my NextCloud running in a Linux Container (LXC)

Prerequisite: Debian 9 Stretch running inside your LXC environment. **All commands must be run with root permission!**
&nbsp;  
&nbsp;

# Part 1: Install LAMB Stack on Debian 9

## Step 1: Update Software Packages
```shell
apt-get  update
apt-get  upgrade
```

## Step 2: Install Apache Web Server
```shell
install apache2 apache2-utils -y
```

Autostart Apache at boot.
```shell
systemctl enable apache2
```

Set `www-data` (Apache user) as the owner of web root directory.
```shell
chown www-data:www-data /var/www/html/ -R
```

## Step 3: Install MariaDB Database Server
```shell
apt-get install mariadb-server mariadb-client -y
```

Autostart MariaDB at boot.
```shell
systemctl enable mariadb
```

Run the post installation security script.
```shell
sudo mysql_secure_installation
```

## Step 4: Install PHP7
```shell
apt-get install php7.0 libapache2-mod-php7.0 php7.0-mysql php-common php7.0-cli php7.0-common php7.0-json php7.0-opcache php7.0-readline -y
```

Enable the Apache php7.0 module and restart Apache Web server.
```shell
a2enmod php7.0
systemctl restart apache2
```
&nbsp;  
&nbsp;

# Part 2: Install NextCloud

## Step 1: Download NextCloud
```shell
wget current_version_of_nextcloud.zip
```

Extract it to `/var/www/`
```shell
unzip current_version_of_nextcloud.zip -d /var/www/
```

Make `www-data` (Apache user) the owner of `/var/www/nextcloud/`.
```shell
chown www-data:www-data /var/www/nextcloud -R
```

## Step 2: Create a Database and User in MariaDB

Login to MariaDB.
```shell
mariadb -u root
```
Create a database for NextCloud.
```shell
create database nextcloud;
```

Create a seperate user.
```shell
grant all privileges on nextcloud.* to dbusername@localhost identified by 'password';
```

Flush MariaDB privileges and exit.
```shell
flush privileges;
exit;
```

## Step 3: Enable Binary Logging in MariaDB

Edit MariaDB configuration file.
```shell
vi /etc/mysql/mariadb.conf.d/50-server.cnf
```

Add the following three lines in `[mysqld]` section.
```Shell
log-bin        = /var/log/mysql/mariadb-bin
log-bin-index  = /var/log/mysql/mariadb-bin.index
binlog_format  = mixed
```

Restart MariaDB.
```shell
systemctl restart mariadb
```

## Step 4: Create an Apache Virtual Host File for Nextcloud

Create a `nextcloud.conf` file in `/etc/apache2/sites-available` directory.
```shell
vi /etc/apache2/sites-available/nextcloud.conf
```

Add this:
```shell
<VirtualHost *:443> 
 DocumentRoot "/var/www/nextcloud" 
 ServerName server.domain
 
<Directory /var/www/nextcloud/> 
 Options +FollowSymlinks 
 AllowOverride All 
 
 <IfModule mod_dav.c> 
 Dav off 
 </IfModule> 
 
 SetEnv HOME /var/www/nextcloud 
 SetEnv HTTP_HOME /var/www/nextcloud 
 Satisfy Any 
 
</Directory> 
 
</VirtualHost>
```

Enable this virtual host using the command below.
```shell
a2ensite nextcloud
```

Enable some Apache modules.
```shell
a2enmod rewrite headers env dir mime setenvif ssl
```

Install needed PHP modules.
```shell
apt-get install php7.0-common php7.0-mysql php7.0-gd php7.0-json php7.0-curl php7.0-zip php7.0-xml php7.0-mbstring -y
```

Restart Apache.
```shell
systemctl restart apache2
```

## Step 5: Enabling HTTPS

Create a directory to save both the certificate and the key.
```shell
mkdir /etc/apache2/ssl
```

Create the certificate and the key.
```shell
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/apache2/ssl/YOURFILENAME.key -out /etc/apache2/ssl/YOURFILENAME.crt
```

Configure Apache.
```shell
vi /etc/apache2/sites-available/000-default.conf
```

Change to this:
```
<VirtualHost *:80>
        ServerName 192.168.1.3
        Redirect "/" "https://server.domain/"
</VirtualHost>

<VirtualHost *:443>
        # The ServerName directive sets the request scheme, hostname and port that
        # the server uses to identify itself. This is used when creating
        # redirection URLs. In the context of virtual hosts, the ServerName
        # specifies what hostname must appear in the request's Host: header to
        # match this virtual host. For the default virtual host (this file) this
        # value is not decisive as it is used as a last resort host regardless.
        # However, you must set it for any further virtual host explicitly.
        #ServerName www.example.com

        ServerAdmin webmaster@localhost
        DocumentRoot /var/www/html

        # Available loglevels: trace8, ..., trace1, debug, info, notice, warn,
        # error, crit, alert, emerg.
        # It is also possible to configure the loglevel for particular
        # modules, e.g.
	    # error, crit, alert, emerg.
        # It is also possible to configure the loglevel for particular
        # modules, e.g.
        #LogLevel info ssl:warn
        SSLCertificateFile /etc/apache2/ssl/FILENAME.crt
        SSLCertificateKeyFile /etc/apache2/ssl/FILENAME.key
        
        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined

        # For most configuration files from conf-available/, which are
        # enabled or disabled at a global level, it is possible to
        # include a line for only one particular virtual host. For example the
        # following line enables the CGI configuration for this host only
        # after it has been globally disabled with "a2disconf".
        #Include conf-available/serve-cgi-bin.conf
</VirtualHost>
```

Restart Apache.
```shell
systemctl restart apache2
```

## Step 6: Configure the `/etc/hosts` file

```shell
127.0.0.1       localhost
192.168.1.3     server.domain
```