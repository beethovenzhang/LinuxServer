# Linux Server Configuration

## Description

This documentation describes how to setup and secure a linux server on [Amazon Lightsail](https://aws.amazon.com/lightsail/) and deploy an Flask application on the server.


The application deployed is the [Favorite Movie App](https://github.com/beethovenzhang/FavoriteMovies).

## Useful info

IP address: [34.221.214.78](http://34.221.214.78).

Accessible SSH port: 2200.

Application URL: [http://ec2-34-221-214-78.us-west-2.compute.amazonaws.com](http://ec2-34-221-214-78.us-west-2.compute.amazonaws.com).

**Note**: If you access the application through the application url, the app will have all the functionalities. But when you use the IP address to access the applicationthe, the Google Oauth login won't work since Google API only allows redict urls that end with a public top-level domain like .com. 

## Softwares installed
1. Time synchronization: ntp daemon ntpd
2. Apache2: apache2
3. mod-wsgi: libapache2-mod-wsgi-py3 python-dev
4. pip3: python3-pip
5. Git
6. Virtual environment: virtualenv
7. Required dependencies: bleach httplib2 request oauth2client sqlalchemy python-psycopg2
8. Postgresql and related tools: libpq-dev python-dev postgresql postgresql-contrib
9. System monitor tool: glances

## Configuration details

### 1 - Create a new user named *grader* and grant this user sudo permissions.

1. Log into the remote VM as *ubuntu* user through ssh: `$ ssh ubuntu@34.221.214.78`.

**Note** There are several ways to [connect to your Ubuntu instance on Lightsail] (https://lightsail.aws.amazon.com/ls/docs/en/articles/lightsail-how-to-connect-to-your-instance-virtual-private-server). You can either connect using your browser which is the easiest way but can become inconvenient later or you can download the SSH key to your local machine and connect from your terminal(recommended).
2. Add a new user called *grader*: `$ sudo adduser grader`.
3. Create a new file under the suoders directory: `$ sudo nano /etc/sudoers.d/grader`. Fill that newly created file with the following line of text: "grader ALL=(ALL:ALL) ALL", then save it.

### 2 - Update all currently installed packages

1. `$ sudo apt-get update`.
2. `$ sudo apt-get upgrade`.
3. Install *finger*, a utility software to check users' status: `$ apt-get install finger`.

### 3 - Configure the local timezone to UTC

1. Open time configuration dialog and set it to UTC with: `$ sudo dpkg-reconfigure tzdata`.
2. Install *ntp daemon ntpd* for a better synchronization of the server's time over the network connection: `$ sudo apt-get install ntp`.

Source: [UbuntuTime](https://help.ubuntu.com/community/UbuntuTime).

### 4 - Configure the key-based authentication for *grader* user

1. Generate an encryption key **on your local machine** with: `$ ssh-keygen -f ~/.ssh/udacity_key.rsa`.
2. Log into the remote VM as *ubuntu* user through ssh and create the following file: `$ touch /home/grader/.ssh/authorized_keys`.
3. Copy the content of the *udacity_key.pub* file from your local machine to the */home/grader/.ssh/authorized_keys* file you just created on the remote VM. Then change some permissions:
	1. `$ sudo chmod 700 /home/grader/.ssh`.
	2. `$ sudo chmod 644 /home/grader/.ssh/authorized_keys`.
	3. Finally change the owner from *ubuntu* to *grader*: `$ sudo chown -R grader:grader /home/grader/.ssh`.
4. Now you are able to log into the remote VM through ssh with the following command: `$ ssh -i ~/.ssh/udacity_key.rsa grader@34.221.214.78`.

### 5 - Enforce key-based authentication
1. `$ sudo nano /etc/ssh/sshd_config`. Find the *PasswordAuthentication* line and edit it to *no*.
2. `$ sudo service ssh restart`.

### 6 - Change the SSH port from 22 to 2200
1. `$ sudo nano /etc/ssh/sshd_config`. Find the *Port* line and edit it to *2200*. Then go to the Lightsail page and add a custom tcp port at 2200 in the `firewall` section. You can delete the original SSH port at 22 on the same section after you confirm that the 2200 port is working.
2. `$ sudo service ssh restart`.
3. Now you are able to log into the remote VM through ssh with the following command: `$ ssh -i ~/.ssh/udacity_key.rsa -p 2200 grader@34.221.214.78`.

Source: [Ubuntu forums](http://ubuntuforums.org/showthread.php?t=1739013).

### 7 - Disable ssh login for *root* user
1. `$ sudo nano /etc/ssh/sshd_config`. Find the *PermitRootLogin* line and edit it to *no*.
2. `$ sudo service ssh restart`.

Source: [Askubuntu](http://askubuntu.com/questions/27559/how-do-i-disable-remote-ssh-login-as-root-from-a-server).

### 8 - Configure the Uncomplicated Firewall (UFW)

Project requirements need the server to only allow incoming connections for SSH (port 2200), HTTP (port 80), and NTP (port 123).

1. `$ sudo ufw allow 2200/tcp`.
2. `$ sudo ufw allow 80/tcp`.
3. `$ sudo ufw allow 123/udp`.
4. `$ sudo ufw enable`.

### 9 - Install Apache, mod_wsgi

1. `$ sudo apt-get install apache2`.
2. Mod_wsgi is an Apache HTTP server mod that enables Apache to serve Flask applications. Install *mod_wsgi* with the following command: `$ sudo apt-get install libapache2-mod-wsgi-py3 python-dev`. Python 3 version is used since the application is in Python 3.
3. Enable *mod_wsgi*: `$ sudo a2enmod wsgi`.
3. `$ sudo service apache2 start`.

### 10 - Install Git

1. `$ sudo apt-get install git`.


### 11 - Clone the Catalog app from Github

1. `$ cd /var/www`. Then: `$ sudo mkdir catalog`.
2. Change owner for the *catalog* folder: `$ sudo chown -R grader:grader catalog`.
3. Move inside that newly created folder: `$ cd /catalog` and clone the catalog repository from Github: `$ git clone https://yourprojecturl catalog`.
4. Make a *catalog.wsgi* file to serve the application over the *mod_wsgi*. That file should look like this:

```python
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0, "/var/www/catalog/")
sys.path.insert(0, "/var/www/catalog/catalog/")

from catalog import app as application
```

Please note that the second `sys.path.insert` is optional. It's helpful if you need to import custom modules in your applications.
5. Some tweaks were needed to deploay the catalog app, so I made a *deployment* branch which slightly differs from the *master*. Move inside the repository, `$ cd /var/www/catalog/catalog` and change branch with: `$ git checkout deployment`.

**Notes**: the *.git* folder will be inaccessible from the web without any particular setting. The only directory that can be listed in the browser will be the static folder: [static assets](http://ec2-34-221-214-78.us-west-2.compute.amazonaws.com/static/).

### 12 - Install virtual environment, Flask and the project's dependencies

1. Install *pip*, the tool for installing Python packages: `$ sudo apt-get install python3-pip`.
2. If *virtualenv* is not installed, use *pip* to install it using the following command: `$ sudo pip install virtualenv`.
3. Move to the *catalog* folder: `$ cd /var/www/catalog`. Then create a new virtual environment with the following command: `$ sudo virtualenv -p python3 venv3`.
4. Activate the virtual environment: `$ source venv3/bin/activate`.
5. Change permissions to the virtual environment folder: `$ sudo chmod -R 777 venv3`.
6. Install Flask: `$ pip install Flask`.
7. Install all the other project's dependencies: `$ pip install bleach httplib2 request oauth2client sqlalchemy python-psycopg2` etc.

Sources: [DigitalOcean](https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps), [Dabapps](http://www.dabapps.com/blog/introduction-to-pip-and-virtualenv-python/).

### 13 - Configure and enable a new virtual host

1. Create a virtual host conifg file: `$ sudo nano /etc/apache2/sites-available/catalog.conf`.
2. Paste in the following lines of code:
```
<VirtualHost *:80>
    ServerName YOUR_REMOTE_IP
    ServerAlias YOUR_REMOTE_NAME_SERVER (optional)
    ServerAdmin ubuntu@YOUR_REMOTE_IP
    WSGIDaemonProcess catalog python-path=/var/www/catalog:/var/www/catalog/catalog/venv3/lib/python3.6/site-packages
    WSGIProcessGroup catalog
    WSGIScriptAlias / /var/www/catalog/catalog.wsgi
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
* The **WSGIDaemonProcess** line specifies what Python to use and can save you from a big headache. In this case we are explicitly saying to use the virtual environment and its packages to run the application.

3. Enable the new virtual host: `$ sudo a2ensite catalog`.

Source: [DigitalOcean](https://www.digitalocean.com/community/tutorials/how-to-run-django-with-mod_wsgi-and-apache-with-a-virtualenv-python-environment-on-a-debian-vps).

### 14 - Install and configure PostgreSQL

1. Install some necessary Python packages for working with PostgreSQL: `$ sudo apt-get install libpq-dev python-dev`.
2. Install PostgreSQL: `$ sudo apt-get install postgresql postgresql-contrib`.
3. Postgres is automatically creating a new user during its installation, whose name is 'postgres'. That is a tusted user who can access the database software. So let's change the user with: `$ sudo su - postgres`, then connect to the database system with `$ psql`.
4. Create a new user called 'catalog' with his password: `# CREATE USER catalog WITH PASSWORD '0000';`.
5. Give *catalog* user the CREATEDB capability: `# ALTER USER catalog CREATEDB;`.
6. Create the 'catalog' database owned by *catalog* user: `# CREATE DATABASE catalog WITH OWNER catalog;`.
7. Connect to the database: `# \c catalog`.
8. Revoke all rights: `# REVOKE ALL ON SCHEMA public FROM public;`.
9. Lock down the permissions to only let *catalog* role create tables: `# GRANT ALL ON SCHEMA public TO catalog;`.
10. Log out from PostgreSQL: `# \q`. Then return to the *grader* user: `$ exit`.
11. Inside the Flask application, the database connection is now performed with:
```python
engine = create_engine('postgresql://catalog:0000@localhost/catalog')
```
12. Setup the database with: `$ python /var/www/catalog/catalog/database_setup.py`.

Source: [DigitalOcean](https://www.digitalocean.com/community/tutorials/how-to-secure-postgresql-on-an-ubuntu-vps).

### 15 - Install system monitor tools

1. `$ sudo apt-get update`.
2. `$ sudo apt-get install glances`.
3. To start this system monitor program just type this from the command line: `$ glances`.
4. Type `$ glances -h` to know more about this program's options.

Source: [eHowStuff](http://www.ehowstuff.com/how-to-install-and-use-glances-system-monitor-in-ubuntu/).

### 16 - Update OAuth authorized JavaScript origins

1. To let users correctly log-in change the authorized URI to [http://ec2-34-221-214-78.us-west-2.compute.amazonaws.com/](http://ec2-34-221-214-78.us-west-2.compute.amazonaws.com) on both Google developer dashboards.

### 17 - Restart Apache to launch the app
1. `$ sudo service apache2 restart`.
Now your app should be accessible from the browser through both the public IP and the url.

#### Special thanks to
[*jungleBadger*](https://github.com/jungleBadger) for his helpful [README](https://github.com/jungleBadger/-nanodegree-linux-server-troubleshoot/blob/master/python3+venv+wsgi/README.md) on how to configure the server for python 3 and
 [*stueken*](https://github.com/stueken) who wrote a really helpful README in his [repository](https://github.com/stueken/FSND-P5_Linux-Server-Configuration).
