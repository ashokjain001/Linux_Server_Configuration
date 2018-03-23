# Linux server configuration & web app deployment project


This is the fifth project for Udacity Fullstack Nanaodegree program, the requirement of this project is to configure a linux server(Amazon lightsail instance) and deploy Catalog web application that was developed earlier in the [program](https://github.com/ashokjain001/Item-catalog-web-app)

IP address for server - [18.217.114.9](18.217.114.9)


## Linux Server Configuration:

Spin up a linux server instance in [Amazon Lightsail](https://aws.amazon.com/lightsail/)

### Update installed packages
```
  sudo apt-get update
  sudo apt-get upgrade
```

### Create new user and grant sudo permission
```
#create linux user
	sudo adduser grader
#enter the passsword

#grant sudo permission
	sudo touch /etc/sudoers.d/grader

#edit file to add this
	grader ALL=(ALL) ALL

```

### Key based authentication for new user 
In your local machine, use command below to generate public private key pair
```
	cd ~/.ssh
	ssh-keygen
```
Copy the content of the .pub file and paste into the .ssh/authorized_keys file of user grader directory 
```
#log in as grader user 
	sudo -su grader

#create .ssh/authorized_keys file and paste the contents
	mkdir /home/grader/.ssh
	touch /home/grader/.ssh/authorized_keys
 	nano /home/grader/.ssh/authorized_keys
 #paste the content of the .pub file from the local machine into /.ssh/authorized_keys file 
```

### Disable remote login of the root user
```
	sudo nano /etc/ssh/sshd_config
#set PermitRootLogin to no, and save the file

#restart ssh service
	sudo service ssh restart
```

### Change SSH access from port 22 to 2200
```
	sudo nano /etc/ssh/sshd_config
	change the line 'Port 22' to 'Port 2200', and save the file
```

### Configure Uncomplicated Firewall 
```
# close all incoming ports
	sudo ufw default deny incoming

# open all outgoing ports
	sudo ufw default allow outgoing

# open ssh on port 2200
	sudo ufw allow 2200/tcp

# open http on port 80
	sudo ufw allow 80/tcp

# open ntp on port 123
	sudo ufw allow 123/udp

# turn on firewall
	sudo ufw enable
```

### Now you should be able to log in through ssh without a password
```
	sudo ssh -vvv -i ~/.ssh/id_rsa grader@18.217.114.9 -p2200
```


### Configure Local timezone to UTC
```
	sudo dpkg-reconfigure tzdata
#choose 'None of the above' in the option and then select 'UTC'
```


# Web server configuration and app deployment

### Apace webserver and MOD-WSGI installation
```
	sudo apt-get install apache2 libapache2-mod-wsgi
```
 MOD-WSGI acts as a gateway to our web application. Anytime we receive a request to access our web application, Apache2 webserver will communicate to our webapp through MOD-WSGI

### PostgreSQl installation and configuration

```
	sudo apt-get install postgresql
```
By default postgreSQL is restricted to listening on localhost, we can confirm by looking at /etc/postgresql/9.5/main/pg_hba.config

```
#TYPE  DATABASE        USER            ADDRESS                 METHOD

#"local" is for Unix domain socket connections only
local   all             all                                     md5
#IPv4 local connections:
host    all             all             127.0.0.1/32            md5
#IPv6 local connections:
host    all             all             ::1/128                 md5
#Allow replication connections from localhost, by a user with the replication privilege.
#local   replication     postgres                                peer
#host    replication     postgres        127.0.0.1/32            md5
#host    replication     postgres        ::1/128                 md5
```

 127.0.0.1/32  corresponds to local connections.
 we can leave as is.


### Create postgres user and creating db for our catalog web app.
postgres creates a user 'postgres' by default while installation, we can use this user access to create a new catalog user and create catalog db for our web application.

```
#connect postgres as postgres user
	sudo su - postgres

#create a new user 'catalogs' with password 'catalogs'
	CREATE USER catalogs WITH PASSWORD 'catalogs'

#create a new DB named 'catalogs' by user 'catalogs' 
	CREATE DATABASE catalogs WITH OWNER catalogs;	

```

now we have our 'catalogs' database ready and we need to reference it in our web application.


### Web application install & config

first step is installing git and cloning our catalog web application
```
#Installing git	
	sudo apt-get install git

#cloning web application at this location 
	cd /var/www/catalog/catalog
	sudo git clone https://github.com/ashokjain001/Item-catalog-web-app.git	

```
Now we will set up virtual environment so that we can install other dependencies required by our project.
```
#Installing pip
	sudo apt-get install python-pip

#Installing virtual env
	sudo pip install virtualvenv

#Create new virtual environment
	virtualenv venv 

#Activating virtual environment
	source venv/bin/activate 			

#Installing all the python packages and dependencies required by this project
	pip install --upgrade -r requirements.txt
```
Configure web application to connect to the postgres catalogs database which we created instead of SQLite 
```
#udpate catalog_db_user.py which converts the python class and converts into postgresql table to use posgres catalogs db

update this 
engine = create_engine('sqlite:///catalogappwithuserslogin.db')

with
engine = create_engine('postgresql://catalogs:catalogs@localhost/catalogs')

#make same changes to lotsofcatalog.py which fills our catalogs db with data. 

#make similar changes to __init__.py which contains code to run our application.
```

Configure Apache to serve the web application using MOD-WSGI
```
#create .wsgi file at this location 
	sudo nano /var/www/catalog/catalog.wsgi
```

Add the following line of code to your .wsgi config file
```
#!/usr/bin/python
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0,"/var/www/catalog/")


from catalog import app as application
application.secret_key = 'Add your secret key'
```

Update the Apache configuration file to serve the web application with WSGI.
```
sudo nano /etc/apache2/sites-enabled/000-default.conf
```
Add following lines of code
```
<VirtualHost *:80>
    ServerName 18.217.114.9
    ServerAlias ec2-18-217-114-9.us-east-2.compute.amazonaws.com
    ServerAdmin admin@18.217.114.9
    WSGIDaemonProcess catalog python-path=/var/www/catalog/catalog:/var/www/catalog/catalog/$
    WSGIProcessGroup catalog
    WSGIScriptAlias / /var/www/catalog/catalog.wsgi
    <Directory /var/www/catalog/catalog>
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


After this step your application should be deployed.















































