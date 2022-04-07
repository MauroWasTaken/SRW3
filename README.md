# SRW3 DAW

## What we use

| Tool             | Version               |
|------------------|-----------------------|
| Ubuntu server    | 20.04.3               |
| Openssh          | 1:8.2p1-4ubuntu0.4    |

## Working from mutiple PC's

```shell
sudo nano /etc/netplan/99_config.yaml
```

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    ens32:
      addresses:
        - 10.229.42.29/20
      gateway4: 10.229.32.1
      nameservers:
        addresses: [ 10.229.28.22, 10.229.28.2 ]
```

```shell
# netplan apply
```

Pass the VM network to `bridge`, and restart.

## Working from mutiple PC's (NAT)

### Access in ssh

```shell
sudo nano /etc/netplan/99_config.yaml
```

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    ens32:
      addresses:
        - 192.168.218.4/24
      gateway4: 192.168.218.2
      nameservers:
        addresses: [ 10.229.28.22, 10.229.28.2 ]
```

- Go to the menu -> Edit -> Virtual Network Editor
- Select the row with "VMNET8"
 - Click on the button "NAT Settings"
 - Button "Add"
 - Here you can add the port on your host that you want to accept your SSH - use SSH if needed. So for example enter 2022 for the host port.
- Add the virtual machine IP address
- Add virtual machine port 22

```shell
ssh my_cpnv@192.168.218.4 -p 2022
```

### Access in http
- Go to the menu -> Edit -> Virtual Network Editor
- Select the row with "VMNET8"
- Click on the button "NAT Settings"
- Button "Add"
- Here you can add the port on your host that you want to accept your SSH - use SSH if needed. So for example enter **1984** for the host port.
- Add the virtual machine IP address
- Add virtual machine port **80**


## LAMP

### Stack installation

Link : <https://doc.ubuntu-fr.org/lamp#methode_recommandeeinstallation_des_paquets>

```shell
sudo apt install -y apache2 php libapache2-mod-php mariadb-server php-mysql
```

To test a service or command. :arrow_down:

```shell
sudo service [service_name] status
OR
[command_name] -v
```

### MariaDB

```shell
sudo mysql_secure_installation
```

Add a new root password and use default choices.

### PHP

Install php modules

```bash
sudo apt install -y php-gd php-curl php-zip php-dom php-xml php-simplexml php-mbstring php-intl php-bcmath php-gmp php-imagick
```

## Nextcloud

From the tutorial : <https://www.linuxbabe.com/ubuntu/install-nextcloud-ubuntu-20-04-apache-lamp-stack>

### Download and install NextCloud

Download the latest Nextcloud zip file

```bash
cd /tmp
wget https://download.nextcloud.com/server/releases/latest.zip
```

Unzip the downloaded file and move the content to `/var/www/html`

```bash
sudo apt install unzip
unzip latest.zip
sudo mv nextcloud /var/www/html
```

Change the owner of the nextcloud directory so that the web server can write to it.

```shell
sudo chown www-data:www-data /var/www/html/nextcloud/ -R
```

### Nextcloud database

```sql
sudo mariadb
CREATE DATABASE nextclouddb;
CREATE USER 'nextclouduser'@'localhost' IDENTIFIED BY 'password';
GRANT ALL ON nextclouddb.* TO 'nextclouduser'@'localhost' IDENTIFIED BY 'password';
FLUSH PRIVILEGES;
```

### Create a Database and User

Login to a mysql session

```bash
sudo mysql
```

Provide your root password when asked then create a database and user with the following command. _Where 'password' is your user password._

```bash
CREATE DATABASE nextclouddb;
CREATE USER 'nextclouduser'@'localhost' IDENTIFIED BY 'password';

GRANT ALL ON nextclouddb.* TO 'nextcloud_user'@'localhost' IDENTIFIED BY 'password';

FLUSH PRIVILEGES;

EXIT;
```

### Graphic installation

Open your browser and go to the page `http://[server_ip]/nextcloud/index.php`

For the new account, create a user name with a password.

For the database fill in all the fields with the corresponding values in the NextCloud database.

![](https://codimd.s3.shivering-isles.com/demo/uploads/44fa0f91ce1d47acab119b4f8.png)

Click on the install button.

When the installation is done, you can access to the dashboard to <http://[ipaddress]/nextcloud/index.php/apps/dashboard/>

### NextCloud error management

In the page `.../nextcloud/index.php/settings/admin/overview`, you can see some errors, we need the handle them.

#### PHP configuration

##### php.ini

```ini
memory_limit = 512M
upload_max_filesize = 500M
post_max_size = 500M
max_execution_time = 300
date.timezone = [your/timezone]
output_buffering = Off
```

`sudo apache2ctl restart`

#### .htaccess

if you encouter the following error:

```
Your data directory and files are probably accessible from the internet. 
The .htaccess file is not working. 
It is strongly recommended that you configure your web server so that the data directory is no longer accessible, or move the data directory outside the web server document root.
```

You need to do the following :

1. Open your apache2.conf file with a file editor
    
    ```bash
    sudo nano /etc/apache2/apache2.conf
    ```
    
1. Scroll untill you find the following
    ```
    <Directory /var/www/>
            Options Indexes FollowSymLinks
            AllowOverride None
            Require all granted
    </Directory>
    ```

1. Replace the AllowOverride parameter to "All"

    ```
    <Directory /var/www/>
            Options Indexes FollowSymLinks
            AllowOverride All
            Require all granted
    </Directory>
    ```

1. Restart the apache server.

    ```bash
    sudo apache2ctl restart
    ```

#### Rewrite rules

Add the following lines in the nextcloud .htaccess

```apache
RewriteRule ^\.well-known/carddav /nextcloud/remote.php/dav/ [R=301,L]
RewriteRule ^\.well-known/caldav /nextcloud/remote.php/dav/ [R=301,L]
RewriteRule ^\.well-known/webfinger /nextcloud/index.php/.well-known/webfinger [R=301,L]
RewriteRule ^\.well-known/nodeinfo /nextcloud/index.php/.well-known/nodeinfo [R=301,L]
```

```shell
sudo a2enmod rewrite
sudo systemctl restart apache2
```

#### HTTPS 

Install openssl (if it isn't already installed)

```bash
sudo apt install openssl
```

Issue the following command to generate a CSR and a key

```bash
openssl req -new -newkey rsa:2048 -nodes -keyout MYServer.cpnv.local.key -out MYServer.cpnv.local.csr
```

Answer the following prompts 

![](https://codimd.s3.shivering-isles.com/demo/uploads/5cb7409230e115350875c1617.png)

Send your request to your CA

If it's not enabled yet, enable the ssl mod

```shell
sudo a2enmod ssl
```

To use https with the server you need to create a nextcloud apache config.

**/etc/apache2/sites-available/nextcloud.conf**

```apache
<VirtualHost *:80>
    ServerName myserver.cpnv.local
    Redirect permanent / https://myserver.cpnv.local/
</VirtualHost>

<VirtualHost *:443>
    DocumentRoot /var/www/html/nextcloud/
    ServerName myserver.cpnv.local

    Alias /nextcloud "/var/www/html/nextcloud/"

    SSLEngine on

    SSLCertificateFile	/etc/ssl/certs/certnew-yannick.cer
    SSLCertificateKeyFile /etc/ssl/private/cert_daw.key

    <Directory /var/www/html/nextcloud/>
        Options +FollowSymlinks
        AllowOverride All
        Require all granted
        <IfModule mod_dav.c>
            Dav off
        </IfModule>
        SetEnv HOME /var/www/html/nextcloud
        SetEnv HTTP_HOME /var/www/html/nextcloud
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

```shell
sudo a2ensite nextcloud.conf
sudo a2dissite 000-default.conf
sudo systemctl restart apache2
```

There is 2 virtual host.

The first redirect all to the port 443 (https).

The second, that handle the port 443, is the used one.

#### Phone numbers

If you have the following error
```
Votre installation n’a pas de préfixe de région par défaut. C’est nécessaire pour valider les numéros de téléphone dans les paramètres du profil sans code pays. Pour autoriser les numéros sans code pays, veuillez ajouter "default_phone_region" avec le code ISO 3166-1 respectif de la région dans votre fichier de configuration.
```
Go to your website's config.php

```bash
sudo nano /var/www/html/nextcloud/config/config.php
```

Add the following before the **);** located at the end of the file

```
'default_phone_region' => 'CH',
```

Your config file should resemble the following

![](https://codimd.s3.shivering-isles.com/demo/uploads/5cb7409230e115350875c1626.png)

Save it and restart the apache server.

```bash
sudo service apache2 restart
```

#### Cache Memory

If you have the following error

```
Pas de mémoire caché configurée. Pour améliorer les performances, merci de configurer un memcache, si disponible.
```

Install redis-server and php-redis

```bash
sudo apt install redis-server php-redis
```

Go to your website's config.php

```bash
sudo nano /var/www/html/nextcloud/config/config.php
```

Add the following before the **);** located at the end of the file

```
'memcache.local' => '\OC\Memcache\Redis', /* contient les scripts php précompilés */
'filelocking.enabled' => 'true',
'memcache.distributed' => '\OC\Memcache\Redis',
'memcache.locking' => '\OC\Memcache\Redis',
'redis' =>
        array (
                'host' => 'localhost',
                'port' => 6379,
                'timeout' => 0,
                'dbindex' => 0,
                ),
```

Your config file should resemble the following

![](https://codimd.s3.shivering-isles.com/demo/uploads/5cb7409230e115350875c1625.png)

Save it and restart the apache server.

```bash
sudo service apache2 restart
```

#### SVG

If you have the following error

```
Le module php-imagick n’a aucun support SVG dans cette instance. Pour une meilleure compatibilité, il est recommandé de l’installer.
```

Install php-imagick and imagemagick by running the following command

```bash
sudo apt install php-imagick imagemagick
```

Restart the apache server.
```bash
sudo service apache2 restart
```
### Trust domains

If you working on multiple PC's you need to tell Nextcloud to trust the other PC ip.

Modify the file config.php of Nextcloud

```bash
sudo nano /var/www/html/nextcloud/config/config.php
```

Add the ip address.

```php
$CONFIG = array (
  'trusted_domains' =>
      array (
        0 => '192.168.153.137',
        1 => '10.229.33.134',
      ),
);
```

Restart the apache server.

```bash
sudo service apache2 restart
```

### Security

## fail2ban
Install the fail2ban package

```bash
sudo apt install fail2ban
```

To add the regex rules that filter users, you need to create the following file : **/etc/fail2ban/filter.d/nextcloud.conf**

Add the following to the file

```
[Definition]

_groupsre = (?:(?:,?\s*"\w+":(?:"[^"]+"|\w+))*)
failregex = ^\{%(_groupsre)s,?\s*"remoteAddr":"<HOST>"%(_groupsre)s,?\s*"message":"Login failed:
            ^\{%(_groupsre)s,?\s*"remoteAddr":"<HOST>"%(_groupsre)s,?\s*"message":"Trusted domain error.
datepattern = ,?\s*"time"\s*:\s*"%%Y-%%m-%%d[T ]%%H:%%M:%%S(%%z)?"
```

And the following to the file

To define what happens to the users that fail the filter, create the following file **/etc/fail2ban/jail.d/nextcloud.local**

```
[nextcloud]
backend = auto
enabled = true
port = 80,443
protocol = tcp
filter = nextcloud
maxretry = 3
bantime = 86400
findtime = 43200
logpath = /var/www/html/nextcloud/data/nextcloud.log
```

**Create the /etc/nextcloud folder and the create the /etc/nextcloud/nextcloud file**

Restart the fail2ban service

```bash
sudo service fail2ban restart
```

## Apache Errors

If you find "AH00558: apache2: Could not reliably determine the server's fully qualified domain name", go to your apache2 config file

```bash
sudo nano /etc/apache2/apache2.conf
```

Add the following

```bash
ServerName [yourServerName]
```

## Apache Security

In the apache2.conf file :arrow_down: 

### Disable the ServerSignature Directive

```
ServerSignature Off
```

### Set the ServerTokens Directive to Prod

```
ServerTokens Prod
```

### Disable Directory Listing

Add the following line to your directory

```
<Directory /your/website/directory>
    Options -Indexes
</Directory>
```

### Restrict Unwanted Services

Add the following line to your directory

```
<Directory /your/website/directory>
    Options -ExecCGI -FollowSymLinks -Includes
</Directory>
```

### Enable Logging

```
LogFormat "%h %l %u %t \"%r\" %>s %b \"%{Referer}i\" \"%{User-Agent}i\"" detailed
CustomLog logs/access.log detailed
sudo mkdir /etc/apache2/logs
```

### Disable ETag header

```
<IfModule mod_headers.c> 
    Header unset ETag 
</IfModule> 
FileETag None
```

### Disable HTTP TRACE / TRACK Methods

```
TraceEnable Off
```

### Clickjacking

```
Header always append X-Frame-Options SAMEORIGIN
```

### XSS-Protection

Go to the following file

```
sudo nano /etc/apache2/conf-enabled/security.conf
```

Add the following

```
Header always set X-XSS-Protection "1;  mode=block"
```

Add the following lines in the apache configuration

```apache
<VirtualHost *:443>

    <IfModule mod_headers.c>
          Header always set Strict-Transport-Security "max-age=15552000; includeSubDomains"
    </IfModule>

</VirtualHost>
```