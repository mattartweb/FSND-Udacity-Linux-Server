# Project: Linux Server Configuration

> Matthew Weber

## About

This project is taking a basic linux server, in this case provided by Amazon Lightsail, and updating it, adjusting the security, and getting a web application running on it.

- IP Address: 18.220.250.224
- SSH Port: 2200
- DNS Address: http://ec2-18-220-250-224.us-east-2.compute.amazonaws.com/

## To Run

### Requirements
- Web Browser
- SSH Client or Terminal (I used PuTTY)

## Setup

### 1 - Creating Amazon Lightsail instance

- Log in to https://lightsail.aws.amazon.com/ls/webapp/home/instances and create a new instance. Choose OS only and the Ubuntu image. You can also choose the free first month.

	> NOTE! If you cannot create an instance you might need to setup a Lightsail policy and a Lightsail user via the Amazon IAM panel, instructions are located here - https://lightsail.aws.amazon.com/ls/docs/en/articles/create-policy-that-grants-access-to-amazon-lightsail. 

- Once you get the instance setup you can connect to it via the Connect using SSH button.

### 2 - SSH Key for PuTTY

- Just below the Connect using SSH button is a link to download your default private key from the Account page. Click it, then download the key from the next page.
- Open PuTTYgen and select load, the select the key that was downloaded (a .pem file) and the choose Save Private Key. This will save a key as a .ppk file and allow you to use PuTTY to remotely access the server.
- Setup a PuTTY profile with the host name as ubuntu@18.220.250.224, port 22, and the SSH AUTH as the .ppk key, then click open. You should SSH into the server.

### 3 - Run some updates!

- Run these commands to update the repo and then upgrade the current software to the latest.

```
$ sudo apt update
$ sudo apt upgrade
```

### 4 - Turn off Remote Root and change the SSH port

- Edit the /etc/ssh/sshd_config file to change the port and turn off remote root access.

```
$ sudo nano /etc/ssh/sshd_config
```

- Change the Port from 22 to 2200, then change PermitRootLogin prohibit-password to PermitRootLogin no.
- CTRL + X will exit the editor, then Y to save.
- Restart the SSH service by running.

```
$ sudo service sshd restart
```

- You will have to update the port on the PuTTY profile you are using, and also update the firewall settings on the Amazon Lightsail instance beneath the Networking tab. For the Amazon Lightsail firewall, remove the SSH port 22 entry and add the following.

```
Custom   TCP   2200
Custom   UDP   123
```

- You can now reconnect via PuTTY.

### 5 - Changing the Ubuntu Server Firewall

- We changed the firewall settings on the Amazon Lightsail instance, but they also need changed on the Ubuntu Server using UFW. First block everything incoming, then allow outgoing, then allow the ports we have specified on the Lightsail firewall - HTTP on Port 80, SSH on port 2200, and NTP on port 123.
- Run these commands to make those updates.

```
$ sudo ufw default deny incoming
$ sudo ufw default allow outgoing
$ sudo ufw allow www
$ sudo ufw allow 2200/tcp
$ sudo ufw allow ntp
$ sudo ufw enable
```

### 6 - Updating the Timezone

 - Simply run the following command to update the timezone to UTC.

```
$ sudo timedatectl set-timezone UTC
```

### 7 - Creating a new user and providing auth and access

- We will be creating a new user called grader and providing them with access to sudo. To create the user, run this command.

```
$ sudo adduser grader
```

- Now to give them access to sudo, edit the /etc/sudoers.d/grader file and add the following line.

```
$ sudo nano /etc/sudoers.d/grader

grader ALL=(ALL) NOPASSWD: ALL
```

- Log in as the grader by using the command.

```
$ sudo su - grader
```

- To provide this user access using the same keys we already created before you can copy the keys from the ubuntu user.

```
$ cd ~/
$ mkdir .ssh
$ sudo cat /home/ubuntu/.ssh/authorized_keys
```

- Copy the key displayed from there and then paste it into the grader's.

```
$ sudo nano .ssh/authorized_keys
```

- Paste in what you copied and save it via CTRL + X, then take ownership.

```
$ chmod 700 .ssh
$ sudo chmod 644 .ssh/authorized_keys
```

- You can now update the PuTTY login to grader@18.220.250.224 on port 2200 with the same .ppk key.

### 8 - Install Apache, Git, Pip, and Python libraries

- Run the following.

```
$ sudo apt install apache2 libapache2-mod-wsgi
$ sudo apt install git
$ sudo apt install python-pip
$ sudo pip install flask httplib2 oauth2client sqlalchemy psycopg2 requests
```


### 9 - Install PostgreSQL Database

- Run the following to install Postgresql.

```
$ sudo apt install libpq-dev python-dev postgresql postgresql-contrib
```

- When postgresql is isntalled it creates its own user, called postgres. To setup the database you have to login as that user and run the commands. To login in as postgres, run the following.

```
$ sudo su - postgres
```
- Then run psql to enter the database setup.

```
$ psql
```

- Once you are in there you will need to run some very exact commands to setup the database. First create a new user named catalogapp.

```
# CREATE USER catalogapp WITH PASSWORD 'password';
```

- Then create a new database named brandcatalog and make catalogapp the owner.

```
# CREATE DATABASE brandcatalog WITH OWNER catalogapp;
```

- Then connect to the new database.

```
# \c brandcatalog
```

- Then take away the public rights and grant all permissions to the catalogapp user. If this isn't done you will get SCHEMA errors on the database.

```
# REVOKE ALL ON SCHEMA public FROM public;
# GRANT ALL ON SCHEMA public TO catalogapp;
```

- You can exit from psql with \q and return to the original user by enterting the word exit.

```
# \q
$ exit
```
-Lastly, add the catalogapp user to the Database admin login by editing the /etc/postgresql/9.5/main/pg_hba.conf file.

```
$ sudo nano /etc/postgresql/9.5/main/pg_hba.conf

local   all             catalogapp                                 password
```


### 10 - Getting and Updating Catalog App

- Apache looks for websites in a specific folder, /var/www/, so create a new folder for our web app there, then clone the github repo to bring over the files.

```
$ sudo mkdir /var/www/catalogapp
$ sudo git clone https://github.com/mattartweb/FSND-Udacity-Item-Catalog.git /var/www/catalogapp
```

- Apache uses WSGI to handle web requests, so we need to create a file for it to use that will point it to the application.

```
$ cd /var/www/catalogapp
$ sudo nano catalogapp.wsgi

import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0, "/var/www/catalogapp/")

from catalog import app as application
application.secret_key = 'SECRET_KEY_THING'
```

- The original application was not designed to be used exactly like this, so some modifications are necessary. First, we need to tell the app it's relative path and also apply that path to some of the other files we are using. Add in an app_path to the application.py file, and also append it to the three places where client_secrets.json is called.

```
$ cd /var/www/catalogapp
$ sudo nano application.py

app_path = '/var/www/catalogapp'

app_path + 'client_secrets.json
```

- We also need to change the python files from calling sqlite3 to using our new postgresql database. In each python file update the engine to the following.

```
engine = create_engine(
    'postgresql://catalogapp:password@localhost/brandcatalog')
```

- Now we can run the database_setup.py to build the database.

```
$ python database_setup.py
```

- We can also remove the facebook button from the login.html because we are no longer on localhost and not using https. Delete or comment out everything between the following comments in /templates/login.html.

```
<!--FACEBOOK SIGN IN -->

<!--END FACEBOOK SIGN IN -->
```

- Lastly we need to update the clients_secret.json file by visitinig https://console.developers.google.com and updating the Authorizard Javascript Origins and the Authorized Redirect URIs with the website information, downloading the JSON, and replacing the existing text in the file with the updated text.

### 11 - Updating the Virtualhost File

- This file will let Apache know where the different parts of code are located in addition to redirecting to port 80 and letting people know who to contact in case of errors. We will update the default file.

```
$ sudo nano /etc/apache2/sites-available/000-default.conf

<VirtualHost *:80>
                ServerName 18.220.250.224
                ServerAdmin mattartweb@gmail.com

                WSGIScriptAlias / /var/www/catalogapp/catalogapp.wsgi

                <Directory /var/www/catalogapp/>
                        Order allow,deny
                        Allow from all
                </Directory>

                Alias /static /var/www/catalogapp/static

                <Directory /var/www/catalogapp/static/>
                        Order allow,deny
                        Allow from all
                </Directory>

                ErrorLog ${APACHE_LOG_DIR}/error.log
                LogLevel warn
                CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

- After you save the 000-default.conf file, restart Apache so that it refreshes everything.

```
$ sudo service apache2 restart
```
### 12 - Access the Website!

- You can access the website via the following links:
> http://ec2-18-220-250-224.us-east-2.compute.amazonaws.com/
> 18.220.250.224

### Resources

- https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps
- http://flask.pocoo.org/docs/1.0/deploying/mod_wsgi/
- https://www.postgresql.org/docs/current/static/runtime-config-client.html
- https://console.developers.google.com
- https://lightsail.aws.amazon.com/ls/docs/en/articles/create-policy-that-grants-access-to-amazon-lightsail
- https://httpd.apache.org/docs/1.3/logs.html
- https://docs.python.org/2.7/faq/programming.html
