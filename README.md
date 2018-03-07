# Linux-Server-Configuration
This is the fifth and final project for the Udacity [Full Stack Developer Nanodegree](https://www.udacity.com/course/full-stack-web-developer-nanodegree--nd004).

## About

In this project, we configure a brand-new, bare bones, Linux server into a secure and efficient web server capable of hosting the [Item Catalog Webapp](https://github.com/fabioderendinger/item-catalog).
More precisely, we will:
- create an Amazon Lightsail instance with bare bonnes Ubuntu image installed
- secure the server from a number of attack vectors
- install and configure a database server
- deploy the Item Catalog project

Visit http://ec2-18-197-97-251.eu-central-1.compute.amazonaws.com to view the deployed web application.

## Step-by-Step Guide
### Create an Amazon Lightsail instance
1. Sign-up for Amazon Lightsail and create a new instance with a bare bones Ubuntu image installed
2. Download the Key-Pair file, rename it to "lightsail_default.pem" and place it in `C:/User/yourusername/.ssh/`

### First time login through ssh
(Note: only for first time login. Port is changed from 22 to 2200 afterwards)
1. Log into the remote VM through ssh: `$ ssh ubuntu@18.197.97.251 -p 22 -i ~/.ssh/lightsail_default.pem`.

### Update all currently installed packages
1. `$ sudo apt-get update`
2. `$ sudo apt-get upgrade`

### Change the SSH Port from 22 to 2200
1. `$ sudo nano /etc/ssh/sshd_config`. Find the *Port* line and edit it to *2200*.
2. `$ sudo service ssh restart`

### Configure the Uncomplicated Firewall (UFW)
1. In the AWS Lightsail Console go to "Networking" tab and add custom firewalls for the ports 2200 and 123.
2. `$ sudo ufw default deny incoming`
3. `$ sudo ufw default allow outgoing`
4. `$ sudo ufw allow 2200/tcp`
5. `$ sudo ufw allow http`
6. `$ sudo ufw allow ntp`
7. `$ sudo ufw enable`

Now we are able to log into the remote VM through ssh with the following command: `$ ssh ubuntu@18.197.97.251 -p 2200 -i ~/.ssh/lightsail_default.pem`.

### Create a new user named `Grader` and grant this user sudo permissions
1. `$ sudo apt-get install finger`
2. `$ sudo adduser grader`. The password is set to: `grader`.
Next, we want to give sudo permission to the user `grader`:
3. `$ sudo touch /etc/sudoers.d/grader`
4. `$ sudo nano /etc/sudoers.d/grader`. Add the following line the the file: `grader ALL=(ALL) NOPASSWD:ALL`.

### Create Key-Pair for user `Grader` (to be done locally!)
1. Open up a new bash window
2. `$ ssh-keygen`. If not specified, the generated puplic and private key will be stored in `/c/User/yourusername/.ssh/`.

### Configure the key-based authentication for the user `Grader`
Back in the bash window connected to the remote server:
1. `$ cd /home/grader`
2. `$ sudo mkdir .ssh`
3. `$ sudo touch .ssh/authorized_keys`
4. `$ sudo nano .ssh/authorized_keys`. Paste the public key generated above into the authorized_key file.
Next, we change the owner of the `.ssh` folder and `authorized_keys` file to Grader
5. `$ sudo chown grader:grader .ssh/authorized_keys`
6. `$ sudo chown grader:grader .ssh`

(Now we would be able to log into the remote VM as user `Grader` through ssh with the following command: `$ ssh grader@18.197.97.251 -p 2200 -i ~/.ssh/id_rsa`)

### Set the local timezone to UTC
1. `$ sudo dpkg-reconfigure tzdata`. Select `none of the above`; `UTC`.

### Disable ssh root login access
1. `$ sudo nano /etc/ssh/sshd_config`
2. `$ sudo service ssh restart`

### Install and configure Apache to serve a Python mod_wsgi application
1. `$ sudo apt-get install apache2`
2. `$ sudo apt-get install libapache2-mod-wsgi`
3. `$ sudo service apache2 restart`

### Install and configure PostgreSQL
1. `$ sudo apt-get install postgresql postgresql-contrib`. By installing PostgreSQL, a new Ubuntu user with the name `postgres` is automatically created.
2. `$ sudo nano /etc/postgresql/9.5/main/pg_hba.conf`. To prevent potential attacks from the outer world we double check that no remote connections to the database are allowed.
3. Authenticate as PostgrSQL: `$ sudo -i -u postgres`
4. Create a PostgrSQL database user (not Ubuntu user) that has limited permissions to our catalog application database:`$ createuser --interactive`. Name: `catalog`; Answer to subsequent questions: `n`; `n`; `n`;
5. Create a PostgrSQL datbase named `catalog`: `$ createdb catalog`
6. `$ psql`
7. `$ ALTER USER catalog WITH PASSWORD 'fakepw';`. This sets a password for the user `catalog`. Close the psql console with `\q`.
6. `$ exit`

### Install git, clone and setup the Catalog App project
1. `$ sudo apt-get install git`
2. `$ cd /var/www/`
3. `$ sudo mkdir FlaskApp`
4. `$ sudo git clone https://github.com/fabioderendinger/item-catalog.git`
5. `$ sudo mv item-catalog itemcatalog`
6. `$ cd itemcatalog`
7. Rename the main python file of the app to `__init__.py`: `$ sudo mv views.py __init__.py`. By doing so, the directory `itemcatalog` is considered to be a python module and the command `from itemcatalog import app as application` in the `itemcatalog.wsgi` file is correctly interpreted.
Next, we need update the database connection information in our python files to make sure our app can connect to the correct database (the one we created above). We use `engine = create_engine('postgresql://dbuser:userpw@localhost/dbname')` with dbuser equal to `catalog` (the database user from which the application access the database), userpw equal to `fakepw` (the pw we set for the database user `catalog`) and dbname equal to `catalog` (the name of the database we want to connect to):
8. `$ sudo nano database_setup.py`. Change the line `create_engine('sqlite:///itemcatalog.db')` to `engine = create_engine('postgresql://catalog:fakepw@localhost/catalog')`.
9. `$ sudo nano views.py`. Change the line `create_engine('sqlite:///itemcatalog.db')` to `engine = create_engine('postgresql://catalog:fakepw@localhost/catalog')`.
10. `$ sudo nana defaultcatalog.py`. Change the line `create_engine('sqlite:///itemcatalog.db')` to `engine = create_engine('postgresql://catalog:fakepw@localhost/catalog')`.
11. `$ sudo apt-get install python-pip`
12. Install packages/dependencies used by the Catalog App: `$ sudo pip install -r requirements.txt`
13. `$ sudo -i -u postgres`
14. `$ sudo apt-get -qqy install postgresql python-psycopg2`
15. `$ cd /var/www/FlaskApp/itemcatalog`
16. Create database schema: `$ python database_setup.py`
17. Populate database with default items: `$ python defaultcatalog.py`
18. `$ exit`

### Create the .wsgi file for this installation
1. `$ cd /var/www/FlaskApp`
2. Create the .wsgi file:`$ sudo nano itemcatalog.wsgi`. Paste in the following lines of code:
```
#!/usr/bin/python
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0,"/var/www/FlaskApp/")

from itemcatalog import app as application
application.secret_key = 'Add your secret key'
application.config['UPLOAD_FOLDER'] = '/var/www/FlaskApp/itemcatalog/static/img/'
```
3. `$ sudo service apache2 restart`

### Configure and enable a new virtual host
1. Create a virtual host conifg file: `$ sudo nano /etc/apache2/sites-available/itemcatalog.conf`. Paste in the following lines of code:
```
<VirtualHost *:80>
        ServerName 18.197.97.251
        ServerAlias http://ec2-18-197-97-251.eu-central-1.compute.amazonaws.com/
        WSGIScriptAlias / /var/www/FlaskApp/itemcatalog.wsgi
        <Directory /var/www/FlaskApp/itemcatalog/>
                Require all granted
        </Directory>
        Alias /static /var/www/FlaskApp/itemcatalog/static
        <Directory /var/www/FlaskApp/itemcatalog/static/>
                Require all granted
        </Directory>
        ErrorLog ${APACHE_LOG_DIR}/error.log
        LogLevel warn
        CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```
2. Disable the default virtual host with: `$ sudo a2dissite 000-default.conf`
3. Then enable the catalog app virtual host: `$ sudo a2ensite itemcatalog`
4. To make these Apache2 configuration changes live, reload Apache: `$ sudo service apache2 reload`

### Allow the application to save files (e.g. images) in the static/img folder
Apache is run as the user www-data. Hence we need to provide this user with the necessary rights to read and write files in this folder by making him the owner of the folder:
`$ sudo chown www-data:www-data /var/FlaskApp/itemcatalog/static/img`

### Update the OAuth configuration
**Google OAuth**
1. Update the fields `javascript_origins` and `redirect_uris` in the Google `client_secrets.json` file with the respective AWS assigend URLs (use domain names, not IP addresses): `$ sudo nano /var/www/FlaskApp/itemcatalog/client_secrets.json`
2. These addresses also need to be entered into the Google Developers Console -> API Manager -> Credentials
3. Change the relative paths to absolute paths in the `__init__.py` file
    - `$ sudo nano /var/www/FlaskApp/itemcatalog/__init__.py`
    - Replace `client_secrets.json` with `/var/www/FlaskApp/itemcatalog/client_secrets.json` everywhere


**Facebook OAuth**
1. Change the relative paths to absolute paths in the `__init__.py` file
    - `$ sudo nano /var/www/FlaskApp/itemcatalog/__init__.py`
    - Replace `fb_client_secrets.json` with `/var/www/FlaskApp/itemcatalog/fb_client_secrets.json` everywhere

## Error Resolution
In case the browers returns an error, you should consult the Error Log file to identify the root cause:
1. `$ cd /var/log/apache2`
2. `$ cat 'error.log'`

If you get the error `(psycopg2.InternalError) current transaction is aborted, commands ignored until end of transaction block`, try to re-create the database `category` from scratch:
1. `$ Drop and re-create the database catalog`
    - `$ sudo -i -u postgres`
    - In psql console (open console with `$ psql`): `$ SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE datname='catalog';`. This command should remove all connections to the database. Close the psql console with `\q`.
    - `$ dropdb catalog`
    - `$ createdb catalog`
    - Re-populate the database with the default items (run `defaultcatalog.py` file)
2. `$ sudo service apache2 restart`