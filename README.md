# Linux Server Configuration

#### Table of Contents
- [Introduction](#Introduction)
- [Get the server](#get-the-server)
- [Secure the server](#secure-the-server)
- [Give grader access](#give-grader-access)
- [Prepare to deploy the project](#prepare-to-deploy-the-project)
- [Deploy the item catalog project](#deploy-the-item-catalog-project)

### Introduction
In this project, I will deploy the [Catalog Item webapp](https://github.com/zhengrui315/Udacity-FSND-Catalog-Item) to a linux server instance. 
I will choose [Amazon Lightsail](https://aws.amazon.com/lightsail/). Useful tutorials include:
https://github.com/kcalata/Linux-Server-Configuration/blob/master/README.md and https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps. 

### Get the server
1, Create a Amazon Lightsail instance. The `platform` is `Linux/Unix`. 
The `blueprint` is `Ubuntu 16.04 LTS`. For the plan, I picked the free first month trial. 
After the instance starts up, the public IP address will be displayed.
For my instance, the public IP address is: [34.229.221.174](34.229.221.174). 

2, Log onto the server remotely as user `ubuntu`. In the main page of Lightsail, click `Account`. 
In the `SSH Keys` tab, click `Download` to download the `pem` file. Rename it `udacity.rsa` and move it to `~/.ssh/`.
To connect to the server, run 

```ssh -i ~/.ssh/udacity.rsa ubuntu@34.229.221.174```

### Secure the server
3, Update all currently installed packages. In the termial, run:
```buildoutcfg
sudo apt-get update
sudo apt-get upgrade
```
4, Change the SSH port from **22** to **2200**. Edit `/etc/ssh/sshd_config` and 
change the port number on line 5 from 22 to 2200. Also set the value of `PermitRootLogin` to `no`.

5, Configure the Uncomplicated Firewall (UFW) to only allow incoming connections for SSH (port 2200), HTTP(port 80), and NTP (port 123) by running:
```buildoutcfg
sudo ufw status
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 2200/tcp
sudo ufw allow www
sudo ufw allow 123/udp
sudo ufw deny 22
sudo ufw enable
```
Go to Lightsail and configure accordingly. 

### Give `grader` access
6, Create a new user account named `grader` by `sudo adduser grader` and enter the password. 

7, Give `grader` the permission to `sudo`. The users with `sudo` permission can be found in `/etc/sudoer` file or in `/etc/sudoers.d` directory. We can 

8, Create an SSH key pair for `grader`. In the terminal of the local machine, run
```buildoutcfg
cd ~/.ssh
ssh-keygen -f ~/.ssh/grader_key.rsa
 ```
Copy the content of `grader_key.rsa.pub` and add it to `/home/grader/.ssh/authorized_keys`. Then on grader's terminal, run
```
sudo chmod 700 /home/grader/.ssh
sudo chmod 644 /home/grader/.ssh/authorized_keys
sudo chown -R grader:grader /home/grader/.ssh
sudo service ssh restart
```
Now I can ssh as `grader` by running the following command: `ssh -i ~/.ssh/grader_key.rsa -p 2200 grader@34.229.221.174`.


### Prepare to deploy the project
9, Configure the local timezone to UTC. Run `sudo dpkg-reconfigure tzdata`, choose Etc or `None of the above`, and finally `UTC`.

10, Install and configure Apache to serve a python mod_wsgi application. 
While logged in as grader, run `sudo apt-get install apache2`.
If the Apache2 Ubuntu Default Page loads after entering the public IP address into the browser, Apache was successfully installed.
Run the following commands:
```buildoutcfg
sudo apt-get install libapache2-mod-wsgi-py3
sudo a2enmod wsgi
sudo service apache2 start
```


11, Install `git` by running `sudo apt-get install git` while logged in as `grader`.

### Deploy the Item Catalog project
12, Clone the project from github. 
```buildoutcfg
sudo mkdir /var/www/catalog
cd /var/www/catalog
sudo git clone https://github.com/zhengrui315/Udacity-FSND-Catalog-Item.git
cd /var/www
sudo chown -R grader:grader catalog/
cd /var/www/catalog/catalog
mv application.py __init__.py
```
- In the `main` function of `__init__.py`, keep only `app.run()` line. 
- In both `__init__.py`, `initialize.py`, and `models.py`, replace the `create_engine` lines by `engine = create_engine('postgresql://catalog:catalog@localhost/catalog')`. 
- Create WSGI file by `sudo vim /var/www/catalog/catalog.wsgi` and paste these lines:
```buildoutcfg
activate_this = '/var/www/catalog/catalog/venv/bin/activate_this.py'
with open(activate_this) as file_:
    exec(file_.read(), dict(__file__=activate_this))

#!/usr/bin/python
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0, "/var/www/catalog/catalog/")
sys.path.insert(1, "/var/www/catalog/")

from catalog import app as application
application.secret_key = 'super_secret_key'
```

13, Install virtual environment.
```buildoutcfg
sudo apt-get install python-pip
sudo pip install virtualenv
cd /var/www/catalog/catalog
virtualenv -p python venv
sudo chown -R grader:grader venv/
source venv/bin/activate
pip install Flask
pip install httplib2
pip install requests
pip install --upgrade oauth2client
pip install sqlalchemy
sudo apt-get install libpq-dev
pip install pyscopg2
deactivate
```
If error occurs for `pyscopg2`, first `pip uninstall psycopg2 psycopg2-binary` Then `pip install psycopg2`. 


14, Configure a new virtual host. Run `sudo nano /etc/apache2/mods-enabled/wsgi.conf`. 
Below where it says `#WSGIPythonPath directory|directory-1:directory-2:...` add the following line: `WSGIPythonPath /var/www/catalog/catalog/venv/lib/python2.7/site-packages`.
Then `sudo vim /etc/apache2/sites-available/catalog.conf` and paste
```buildoutcfg
<VirtualHost *:80>
    ServerName 34.229.221.174
  ServerAlias ec34-229-221-174.compute-1.amazonaws.com
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
Then
```buildoutcfg
sudo a2ensite catalog
sudo service apache2 reload
```

15, Install and configure PostgreSQL.
```buildoutcfg
sudo apt-get install libpq-dev python-dev
sudo apt-get install postgresql postgresql-contrib
sudo su - postgres
psql
CREATE USER catalog WITH PASSWORD 'catalog';
ALTER USER catalog CREATEDB;
CREATE DATABASE catalog WITH OWNER catalog;
\c catalog
REVOKE ALL ON SCHEMA public FROM public;
GRANT ALL ON SCHEMA public TO catalog;
\q
exit
source venv/bin/activate
python /var/www/catalog/catalog/initialize.py
deactivate
```











