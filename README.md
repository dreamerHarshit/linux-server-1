# Linux Server Configuration
Taking a baseline installation of a Ubuntu Linux distribution on a virtual machine on Amazon EC2
 and preparing it to host a web application.<br/>
 Includes installing updates,
 securing it from a number of attack vectors and installing/configuring web and database servers.

 ## Server Information
 - IP Address: 52.42.38.177
 - SSH port: 2200
 - URL of hosted web application: [http://ec2-52-42-38-177.us-west-2.compute.amazonaws.com/](http://ec2-52-42-38-177.us-west-2.compute.amazonaws.com/)
 - SSH connection command: ```ssh -i [key_file_path] grader@52.42.38.177 -p 2200```<br/>
 where [key_file_path] is the path to the file containing the supplied private key of the grader user.

 ## Configuration Steps

 ### Create New User Named grader And Grant user sudo permissions
 ```
 # create linux user
 sudo adduser grader

 # give user sudo permissions
 echo "grader ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/grader
```

### Enable key based login For User grader
On local machine, use command ssh-keygen to create public and private keys for the user grader
```
ssh-keygen -f ~/.ssh/grader
```
Deploy the public key to the grader user .ssh folder on the server
```
su grader
mkdir /home/grader/.ssh
nano /home/grader/.ssh/authorized_keys
# copy public key from local machine and paste into authorized_keys and save
```


 ### Update Installed Packages
 ```
 sudo apt-get update
sudo apt-get upgrade
```

 ### Change SSH Port From 22 To 2200
 ```
 sudo nano /etc/ssh/sshd_config
 # change line Port 22 to Port 2200
 ```

 ### Configure Uncomplicated Firewall
 ```
 # close all incoming ports
sudo ufw default deny incoming
# open all outgoing ports
sudo ufw default allow outgoing
# open ssh port
sudo ufw allow 2200/tcp
# open http port
sudo ufw allow 80/tcp
# open ntp port
sudo ufw allow 123/udp
# turn on firewall
sudo ufw enable
```

 ### Configure Local Timezone to UTC
 Machine already set to UTC
 ```
 sudo dpkg-reconfigure tzdata
 # choose 'None of the above' and then select 'UTC'
 ```

 ### Install Apache and mod_wsgi module
 ```
 sudo apt-get install apache2 libapache2-mod-wsgi
 ```

 ### Install and configure PostgreSQL
 ```
 sudo apt-get install PostgreSQL
 # check that remote connections are not allowed in PostgreSQL config file
 sudo nano /etc/postgresql/9.3/main/pg_hba.config
 ```

 ### Create user named catalog that has limited permissions to catalog application database
 
 ```
# create linux user catalog
sudo adduser catalog

# create PostgreSQL role catalog and db catalog
sudo -u postgres -i
postgres:~$ creatuser catalog
postgres:~$ createdb catalog
postgres:~$ psql
postgres=# ALTER DATABASE catalog OWNER TO catalog;
postgres=# ALTER USER catalog WITH PASSWORD 'catalog'
postgres=# \q
postgres:~$ exit
```

 ### Install git and clone web application  project
 ```
 sudo apt-get install git
 cd /var/www
git clone https://github.com/iainbx/item-catalog.git

# ensure git folder is not accessible via web server
echo "RedirectMatch 404 /\.git" > /var/www/.htaccess
 ```

 ### Install Python libs required by web application
 ```
 $ sudo apt-get install python-pip python-dev python-psycopg2
$ sudo pip install -r /var/www/item-catalog/requirements.txt
```
 ### Configure web app to use PostgreSQL db instead of SQLLite db
 ```
 sudo nano /var/www/item-catalog/config.py
 # change DATABASE_URI line in file from sqllite to  postgresql://catalog:db_password@localhost/catalog
 ```

 ### Create schema and Populate catalog db with sample database
 ```
 python /var/www/create_sample_data.py
 ```

 ### Configure Apache to serve wsgi web app
 create wsgi file
 ```
 sudo nano /var/www/item-catalog/app.wsgi
```
add following lines:
```
#!/usr/bin/python
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0,"/var/www/item-catalog/")
from catalog import app as application
application.secret_key = 'a secret key'
```

edit /etc/apache2/sites-enabled/000-default.conf
```
sudo nano /etc/apache2/sites-enabled/000-default.conf
```
add line inside <VirtialHost *:80> element:
WSGIScriptAlias / /var/www/item-catalog/app.wsgi

restart apache
```
sudo apache2ctl restart
```

### Update Authorized Origins For Google and Facebook Logins
Get the Google Sign-In button working by updating "Authorized JavaScript
Origins" for the application in the Google Developers Console so it
includes the following origins (x.x.x.x/x-x-x-x are the IP):
http://x.x.x.x
http://ec2-x-x-x-x.us-west-2.compute.amazonaws.com

download google_client_secrets.json with new origins and place in app folder

update facebook app domains


### Allow apache to write to image upload folder
 by changing owner to apache user
 ```
sudo chown www-data /var/www/item-catalog/catalog/static/uploads
sudo chmod 744 /var/www/item-catalog/catalog/static/uploads
```

### Fix sudo warning message
To fix this, the hostname was added to the loopback address in the /etc/hosts file so that th first line now reads: 127.0.0.1 localhost ip-10-20-47-177

### Fix apache2ctl warning message
apache2ctl warning message: 
"apache2: Could not reliably determine the server's fully qualified domain name, using 127.0.0.1. Set the 'ServerName' directive globally to suppress this message". 
So I added the line below to/etc/apache2/apache2.conf:
```
sudo echo "ServerName localhost" >> /etc/apache2/apache2.conf ???


