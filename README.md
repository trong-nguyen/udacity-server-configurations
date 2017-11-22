# Linux Server Configurations
This is a project to demo my DevOps skills which I rely on to deliver developed web apps to the public. Though some specific tools are mentioned in the project, the underlying understanding could be applied to other tools (NginX, UWSGI, MySQL instead of Apache2, Mod_WSGI, PostgreSQL).

A detailed walk-through of server configuration for hosting of multiple web applications. The utmost important aspect is security for which user permissions, firewall and admin connection (SSH) are configured. A separate execution environment (user, database, environment variables) is dedicated to each application. Web server (Apache) configuration is also covered, particularly for running Python web apps. The stack for this demo project involves Amazon Lightsail, Linux, SSH services, Apache2 web server, Mod_WSGI, PostgreSQL, Linux Uncomplicated Firewall and Python VirtualEnv.

## Server Details

IP: 54.85.31.69

URL: http://54.85.31.69

ssh port: 2200

Connect to the ssh server:

`ssh -i /path/to/submited/chmod/400/private/key grader@54.85.31.69 -p 2200`

## Software installed

- Apache2
- mod_wsgi
- PostgreSQL
- python-pip
- virtualenv
- [Catalog App](https://github.com/trong-nguyen/udacity-catalog-site.git) and its dependencies:
```
bleach==2.0.0
certifi==2017.7.27.1
chardet==3.0.4
click==6.7
Flask==0.12.2
Flask-HTTPAuth==3.2.3
Flask-SQLAlchemy==2.2
html5lib==0.999999999
httplib2==0.10.3
idna==2.6
itsdangerous==0.24
Jinja2==2.9.6
MarkupSafe==1.0
oauth2client==4.1.2
packaging==16.8
passlib==1.7.1
psycopg2==2.7.3.1
pyasn1==0.3.4
pyasn1-modules==0.1.4
pyparsing==2.2.0
redis==2.10.6
requests==2.18.4
rsa==3.4.2
six==1.10.0
SQLAlchemy==1.1.14
urllib3==1.22
webencodings==0.5.1
```

## Configuration Steps

### Create a Linux instance using AWS Lightsail

- Connect using your own shell. Download the pem key from [Lightsail](https://lightsail.aws.amazon.com/ls/webapp/account/keys)
```shell
# chmod to 400
chmod 400 path/to/your/lightsail/key.pem

# connect
ssh -i path/to/your/lightsail/key.pem username@public_ip

# copy public SSH key to the Lightsail instance
local@ssh-keygen # create a public key on the local machine
# copy content of the .pub key to clipboard and echo it to the remote authorized_keys file
echo "rsa-....(public key content)" >> ~/.ssh/authorized_keys
```



### Initial configurations and network security

```shell
# update package lists
sudo apt-get update

# upgrade installed packages and dependencies
sudo apt-get dist-upgrade

# remove no longer required libraries
sudo apt autoremove

# open port 2200, get prepared for ssh port change (from 22 to 2200)
sudo ufw allow 2200
# enable and reload
sudo ufw enable
sudo ufw reload # (opt)
sudo ufw status # (opt) check status

# change ssh port 22 -> 2200 (look for a line starting with "Port 22")
sudo nano /etc/ssh/sshd_config
sudo service sshd restart # restart ssh service

# one last step: on the Lightsail web panel, under networking, add a custom port to be opened at 2200
```

- Setup firewall to enable ports
```shell
sudo ufw allow in 2200
sudo ufw allow in 80
sudo ufw allow in 123
sudo ufw reload # take effect after reloading
```

- Disable remote login as root
```shell
sudo nano /etc/ssh/sshd_config
# and change line to "PermitRootLogin no"
```

- Disable password-based authentication for ssh connections
```shell
sudo nano /etc/ssh/sshd_config
# And change a line to `PasswordAuthentication no`
# Then restart ssh service
sudo service sshd restart
```

### Setup `grader` user

```shell
sudo adduser grader
# sudo passwd -e grader # (opt) expire grader password immediately, enforce password change - Udacity
su - grader # login to grader

# add sudo permission to grader
# either edit the /etc/sudoers
sudo visudo # and copy the line
    root ALL=(ALL:ALL) ALL
    to grader ALL=(ALL:ALL) ALL

# or (recommended by Ubuntu) create a new file in the /etc/sudoers.d/ such as /etc/sudoers.d/grader-user with the following line
    grader ALL=(ALL) NOPASSWD:ALL

# generate ssh key and follow instructions
ssh-keygen

# create authorized_keys file
touch ~/.ssh/authorized_keys

# copy the public key to the trusted list, this will enable anyone with the right private key login as grader
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
```

### Deploy app

- Set timezone to UTC
```shell
sudo timedatectl set-timezone UTC
```

- Install Apache
```shell
sudo apt-get install apache2 libapache2-mod-wsgi
```
- PostgreSQL
```shell
sudo apt-get install postgresql postgresql-contrib

sudo dpkg-reconfigure locales # https://askubuntu.com/a/694172

# switch to #postgres user (created during PostgreSQL install)
sudo su - postgres
# create PostgreSQL user "catalog" and database "catalog"
postgres$createuser catalog
postgres$createdb catalog
postgres$psql
psql#ALTER DATABASE catalog OWNER TO catalog; # hand over ownership of catalog-the database to catalog-the user.

# create an actual catalog linux user, our app will run as this user
sudo adduser catalog

sudo apt-get install python-pip
sudo apt-get install virtualenv

# switch to catalog user
su - catalog # enter password
mkdir catalog-app
cd catalog-app
git clone https://github.com/trong-nguyen/udacity-catalog-site.git
virtualenv init
. init/bin/activate
pip install udacity-catalog-site/requirements.txt

python setup.py # create database and data
```

- Setup environment
```shell
sudo apt-get install libapache2-mod-wsgi python-dev
sudo a2enmod wsgi
```

- Configure Apache2 and WSGI
```shell

# config content as in wsgi_config.txt file in repo (explained below)
sudo nano /etc/apache2/sites-available/catalog.conf

# add the site to serving list
sudo a2ensite catalog
sudo service apache2 reload
```

### Configure Mod_WSGI to serve our app

- Put this in /etc/apache/sites-available/[APP_NAME].conf
```
# wsgi_config.txt content
<VirtualHost *:80>
    ServerName [SERVERNAME]
    ServerAdmin [CONTACT_EMAIL]
    WSGIDaemonProcess [DAEMON_PROCESS_NAME] user=[USERNAME] group=[GROUPNAME] home=[HOME_DIRECTORY]
    WSGIProcessGroup [DAEMON_PROCESS_NAME]
    WSGIScriptAlias / [PATH_TO_THE_WSGI_FILE]
    <Directory [PATH_TO_THE_APP_DIRECTORY_IN_www]>
        Order allow,deny
        Allow from all
    </Directory>
    ErrorLog ${APACHE_LOG_DIR}/error.log
    LogLevel warn
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

- where SERVERNAME is the address (URL) of the site. It could be:
    - localhost (reached from within)
    - 127.0.0.1
    - 53.24.19.12
    - realwebsite.com
    - somename.com
    - api.somename.com

- That will be the identifier that Apache bases on to direct the requests to correct process. For example:
    - request `somename.com` routed to `52.1.42.22` through Apache will go to VirtualHost with SERVERNAME of `somename.com`
    - request `api.somename.com` routed to `52.1.42.22` through Apache will go to VirtualHost with SERVERNAME of `api.somename.com`
    - request `52.1.42.22` routed to `52.1.42.22` through Apache will go to VirtualHost with SERVERNAME of `52.1.42.22`

- **Important!** DAEMON_PROCESS_NAME is used to identify the group of daemon processes to serve a particular SERVERNAME. It must be used to demand correct serving processes (and permissions). Example:
```shell
sudo adduser apprunner
sudo -u postgres createuser apprunner # execute the command createuser as postgres
# then the USERNAME and GROUPNAME in our configuration will be apprunner
# DAEMON_PROCESS_NAME could be anything like apprunnerGroup or similar.
```
- USERNAME is the Linux username used to run the daemons (and possesses according permissions such as connect to Databases)
- GROUPNAME is the Linux group name, usually the same as USERNAME if not otherwise specified for the USERNAME (in OS configurations).
- HOME_DIRECTORY will be the current working directory of spawn daemons, important for I/O activities: reading secrets, etc.

### Understanding database security with PostgreSQL authentication

In config file `/etc/postgresql/[VERSION]/main/pg_hba.conf`
```shell
local   all             postgres                                peer

# TYPE  DATABASE        USER            ADDRESS                 METHOD

# "local" is for Unix domain socket connections only
local   all             all                                     peer
# IPv4 local connections:
host    all             all             127.0.0.1/32            md5
# IPv6 local connections:
host    all             all             ::1/128                 md5
# Allow replication connections from localhost, by a user with the
# replication privilege.
#local   replication     postgres                                peer
#host    replication     postgres        127.0.0.1/32            md5
#host    replication     postgres        ::1/128                 md5
```
Notice the `METHOD` section where we can see options such as peer and md5. Where peer only allows connection from Unix socket (loggedin user, for only Linux users with corresponding (the same) PostgreSQL username, i.e. if a Linix user account joe, that user can only connect to a database accessible to a PostgreSQL username joe. Note that Linux username # PostgresSQL username. In peer auth, PostgreSQL only allows connect from username with the same Linux username.

> By default, PostgreSQL handles authentication by associating Linux user accounts with PostgreSQL accounts. This is called "peer" authentication.

#### Peer authentication

The peer authentication method works by obtaining the client's operating system user name from the kernel and using it as the allowed database user name (with optional user name mapping). This method is only supported on local connections.

#### Password authentication
The password-based authentication methods are md5 and password. These methods operate similarly except for the way that the password is sent across the connection, namely MD5-hashed and clear-text respectively.


## Attributed Resources
- Uncomplicated FireWall [ufw](https://www.linux.com/learn/introduction-uncomplicated-firewall-ufw)
- Configure [Apache virtual hosts](https://serverfault.com/a/520201)
- Configure [Apache, mod_wsgi and Python apps (Flask, Django)](https://www.digitalocean.com/community/tutorials/how-to-run-django-with-mod_wsgi-and-apache-with-a-virtualenv-python-environment-on-a-debian-vps)
- PostgreSQL security by [DigitalOcean](https://www.digitalocean.com/community/tutorials/how-to-secure-postgresql-on-an-ubuntu-vps)
