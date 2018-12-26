# Linux Server Configuration

A step by step linux server configuration to host Flask app like [Item catalog](https://github.com/ZizoNaser/item-catalog).

## Server information

url:`http://35.178.212.250/`
ssh: 2200

## Getting Started

These instructions will help you deploy your flask on linux server and secure it.

### Prerequisites

- Ubuntu18.04 LTS instance on [AWS LightSail](https://lightsail.aws.amazon.com/).
- Logged in using LightSail.

### Configration

#### Update

Update ubuntu packages using apt.

```bash
sudo apt update && sudo apt upgrade && sudo apt full-upgrade
```

#### Create new user

Create new user `grader`

```bash
sudo adduser gader
```

Add `grader` to the `sudo` group

```bash
sudo usermod -aG sudo grader
```

Switch to `grader` acount

```bash
su grader
```

#### Create ssh key

##### On the local machine

- Create ssh pair of keys using `ssh-keygen` enter name of the key e.g. `grader`

##### On the server

- Create new directory in the `grader` home directory

```bash
mkdir .ssh
```

- Change `.ssh` permissions

```bash
chmod 700 .ssh
```

- Get in the `.ssh` directory

```bash
cd .ssh
```

- Create file named `authorized_keys` and copy the content of the `grader.pub` into `authorized_keys`

```bash
nano authorized_keys
```

- Change `authorized_keys` permissions

```bash
chmod 644 authorized_keys
```

##### Edit the configuration file

First in [LightSail](https://lightsail.aws.amazon.com/)

```
Head to your instance - >
                     Networking ->
                            Firewall ->
                             Add another->
                              2200/tcp as custom port.
                              save
```

Then in `/etc/ssh/sshd_config`:

```bash
sudo nano /etc/ssh/sshd_config
```

###### Disable login using password

- Change `PasswordAuthentication` to `no`.

###### Disable root login

- Change `PermitRootLogin` to `no`.

###### Force login using Public key

- Change `PubkeyAuthentication` to `yes`.

###### Change the default port to 2200

- Change `port` to `2200`.

###### save the changes

```
ctl + o
ctl + x
```

###### Restat ssh

```bash
sudo service ssh restart
```

Your connection with the server will terminate. from your local machine terminal connect using the public key.

```bash
ssh grader@server-ip-address -p 2200 -i ~/.ssh/grader
```

#### Configure Firewall

Ubuntu 18.04 servers can use the UFW firewall to make sure only connections to certain services are allowed. We can set up a basic firewall very easily using this application.

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 2200/tcp
sudo ufw allow www
sudo ufw allow ntp
sudo ufw enable
```

#### Configure the timezone

We can reconfigure the timezone using `dpkg-reconfigure`

```bash
sudo dpkg-reconfigure tzdata
```

choose `none of the above` then `UTC`.

#### Install Apache

```bash
sudo apt install apache2
```

#### Install Python mod_wsgi package

```bash
sudo apt install libapache2-mod-wsgi
```

Enale mod_wsgi

```bash
sudo a2enmod wsgi
```

#### Install Apache

```bash
sudo apt install libpq-dev python-dev postgresql postgresql-contrib
```

#### Clone the flask app

we will use the [item catalog](https://github.com/ZizoNaser/item-catalog) in this example.

- Create a new directory for the project in `/var/www/`

```bash
sudo mkdir /var/www/catalog
```

- Change the owner of the new directory to `grader`

```bash
sudo chown -R grader:grader /var/www/catalog
```

- Change the working `cd /var/www/catalog`.

- Clone the app

```bash
git clone https://github.com/ZizoNaser/item-catalog.git && mv item-catalog catalog
```

- Create new branch

```bash
cd catalog && checkout -b deployment
```

- Change app file to `__init__.py`

```bash
mv FinalProject.py __init__.py
```

- Finally make `.git` inaccessible

```bash
cd /var/www/catalog/
sudo nano .htaccess
```

Add `RedirectMatch 404 /\.git` then save.

### Install requirement

```bash
sudo pip install Flask httplib2
sudo pip install oauth2client sqlalchemy sqlalchemy_utils
sudo pip install requests
sudo pip install psycopg2
```

### Create Database

- Switch to postgres user

```bash
su postgres
```

- Cahnge the working directory to the app file

```bash
cd /var/www/catalog/catalog
```

- Open PostgreSQL

```bash
psql
```

- Create new user `catalog`

```shell
CREATE USER catalog WITH PASSWORD 'password';
```

- Create new Database named `catalog`

```shell
CREATE DATABASE catalog WITH OWNER catalog;
```

- Connect to `catalog` databse

```shell
\c catalog
```

- Revoke rights

```shell
 REVOKE ALL ON SCHEMA public FROM public;
```

- Lock down the permissions only to user catalog

```shell
GRANT ALL ON SCHEMA public TO catalog;
```

- Exit PostgreSQL

```shell
\q
```

- Return to the `grader` user

```bash
exit
```

- Edit the app and database_setup files and change `create_engine` function's paramter to be

```python
create_engine('postgresql://catalog:password@localhost/catalog')
```

- Finally run `database_setup`

```bash
python database_setup.py
```

### Create wsgi file

- Cahnge the working directory `/var/www/catalog`

```bash
cd /var/www/catalog
```

- Create a `itemsCatalog.wsgi`

```bash
nano itemsCatalog.wsgi
```

- Add

```python
#!/usr/bin/python
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0,"/var/www/catalog/")

from catalog import app as application
application.secret_key = 'secret_key'
```

### Configure Apache2 file

```bash
sudo nano /etc/apache2/sites-available/000-default.conf
```

Change its content to:

```shell
<VirtualHost *:80>
                ServerName servername
                ServerAdmin email
                WSGIScriptAlias / /var/www/catalog/itemsCatalog.wsgi
                <Directory /var/www/catalog/catalog/>
                        Order allow,deny
                        Allow from all
                </Directory>
                Alias /static /var/www/catalog/catalog/static
                <Directory /var/www/catalog/catalog/static/>
                        Order allow,deny
                        Allow from all
                </Directory>
                ErrorLog ${APACHE_LOG_DIR}/error.log
                LogLevel warn
                CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

save then restart Apache2

```bash
sudo services apache2 restart
```

That's it enjoy.


## Resources

### DigitalOcean

- [Initial Server Setup with Ubuntu](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-16-04)
- [How To Serve Django Applications with Apache and mod_wsgi on Ubuntu](https://www.digitalocean.com/community/tutorials/how-to-serve-django-applications-with-apache-and-mod_wsgi-on-ubuntu-16-04)

- [How To Install the Apache Web Server on Ubuntu](https://www.digitalocean.com/community/tutorials/how-to-install-the-apache-web-server-on-ubuntu-18-04-quickstart)

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details
