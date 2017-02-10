# Setup Ubuntu Webserver for Flask application

# Content
 1. [The server](#the-server)
 2. [Login to Amazon server](#login-to-amazon-server)
 3. [Adding new sudo user](#adding-new-sudo-user)
 4. [Installing Postgresql and create database](#installing-postgresql-and-create-database)
 5. [Installing Apache, Flask enviroment and the application](#installing-apache-flask-enviroment-and-the-application)
 6. [Firewall setup](#firewall-setup)
 7. [Configure the local timezone](#configure-the-local-timezone)
 8. [Sources](#sources)
 
# The server
 Server IP address: [35.162.49.152](ssh://root@35.162.49.152:2200) SSH port: 2200
 
 Web address: [ec2-35-162-49-152.us-west-2.compute.amazonaws.com/](http://ec2-35-162-49-152.us-west-2.compute.amazonaws.com/)
 
# Login to Amazon server
 * Download your private_key as your_private_key.rsa
 * Move it to your ssh directory and change its permission:
 
  ```
  mv ~/Downloads/your_private_key.rsa ~/.ssh/
  chmod 600 ~/.ssh/your_private_key.rsa
  ```
 * Login to the server (change YOUR_SERVER_IP_ADDRESS to your server's ip address):
 
  ```
  ssh -i ~/.ssh/your_private_key.rsa root@YOUR_SERVER_IP_ADDRESS
  ```
 
 If you want use PuTTY terminal to login the server, you should convert the private key with PuTTYgen!
 
## Adding new sudo user

 ```
 adduser grader
 usermod -aG sudo grader
 ```
 
## Installing Postgresql and create database
 * Installing Postgresql
 
  ```
   sudo apt-get install postgresql postgresql-server-dev-all
  ```
 * Create database and add new database user
 
  ```
  su - postgres
  createuser catalog_user
  createdb catalog
  psql "alter user catalog_user with encrypted password 'CATALOG_USER_PASSWORD';"
  psql "grant all privileges on database catalog to catalog_user;"
  ```
 * Setup Postgresql for accessing only from localhost
  Edit postgresql.conf:
 
  ```
  sudo nano /etc/postgresql/9.3/main/postgresql.conf
  ```
  Changing listen_addresses parameter (uncomment if needed):
  ```
  listen_addresses = 'localhost'
  ```
  
  Changing pg_hha.conf file:
  
  ```
  sudo nano /etc/postgresql/9.3/main/pg_hba.conf
  ```
  Add the folowwing line:
  
  ```
  # TYPE  DATABASE        USER            ADDRESS                 METHOD
  host    catalog         catalog         127.0.0.1/32            md5
  ```
  
  Restart Postgresql:
  
  ```
  sudo service postgresql restart
  ```
## Installing Apache, Flask enviroment and the application
### Installing Apache and WSGI mod:
  
  ```
  sudo apt-get install apache2
  sudo apt-get install libapache2-mod-wsgi
  ```
  
### Cloning the application:
  ```
  mkdir /var/www/flask_application
  git clone https://github.com/janosvincze/catalog /var/www/flask_application
  ```
  
### Setup virtualenv
  
  ```
  sudo apt-get install python-virtualenv
  ```
  
  Create the application own environment
  
  ```
  cd /var/www/flask_application
  virtualenv venv
  . venv/bin/activate
  pip install Flask Flask-Login
  pip install psycopg2 sqlalchemy
  ```
  
  Create a wsgi file to deploy the application:
  ```
  nano /var/www/flask_application/deploy.wsgi
  ```
  ```
  import os
  import sys
  import site

  # Add virtualenv site packages
  site.addsitedir(os.path.join(os.path.dirname(__file__), 'env/local/lib64/python2.7/site-packages'))
  
  # Path of execution
  sys.path.append('/var/www/flask_application')

  # Fired up virtualenv before include application
  activate_env = os.path.expanduser(os.path.join(os.path.dirname(__file__), 'env/bin/activate_this.py'))
  execfile(activate_env, dict(__file__=activate_env))

  # import catalog as application
  from catalog import app as application
  application.secret_key = 'YOUR_SUPER_SECRET_KEY'
  application.static_folder = 'static'
  ```
  
### Setup the application
  
  Changing database connection in the python files:
  
  ```
  engine = create_engine('postgresql://catalog:CATALOG_USER_PASSWORD@localhost/catalog')
  ```

### Configuring Apache
  
  ```
  nano /etc/apache2/sites-available/000-default.conf
  ```
  And add the required lines:
  
  ```
  <VirtualHost *:80>
        ServerName YOUR_SERVER_ADDRESS

        ServerAdmin your@emailaddress
        DocumentRoot /var/www/
        
        WSGIDaemonProcess catalog user=www-data group=www-data threads=5 home=/var/www/flask_application/
        WSGIScriptAlias / /var/www/flask_application/deploy.wsgi
        WSGIScriptAlias /static/ /var/wwww/flask_application/static/
        Alias "/templates/" "/var/www/flask_application/templates/"
        Alias "/static/" "/var/www/flask_application/static/"

        <directory /var/www/flask_application>
                WSGIProcessGroup catalog
                WSGIApplicationGroup %{GLOBAL}
                WSGIScriptReloading On
                Order deny,allow
                Allow from all
        </directory>
        <directory "/var/www/flask_application/static/">
                Order deny,allow
                Allow from all
        </directory>
        <directory "/var/www/flask_application/templates/">
                Order deny,allow
                Allow from all
        </directory>
  </VirtualHost>
  ```
  
  Enable the site:
  ```
  a2ensite YOUR_SERVER_ADDRESS
  ```
  
  Restarting Apache
  ```
  service apache2 restart
  ```
  
## Firewall setup
 Changing ssh port
 
 ```
 sudo nano /etc/ssh/sshd_config
 ```
 
 Change the config as
 ```
 # What ports, IPs and protocols we listen for
  Port 2200
 ```
 
 Restart ssh service
 ```
 service ssh restart
 ```
 
 Configure the Uncomplicated Firewall (UFW) to only allow incoming connections for SSH (port 2200), HTTP (port 80), and NTP (port 123)
 
 ```
 sudo ufw default deny incoming
 sudo ufw default allow outgoing
 sudo ufw allow 2200/tcp
 sudo ufw allow www
 sudo ufw allow ntp
 sudo ufw status
 sudo ufw enable
 ```

## Configure the local timezone
 ```
 sudo timedatectl set-timezone UTC
 ```

## Sources
  * Udacity course
  * [Flask documentation](http://flask.pocoo.org/docs/0.12/installation/#virtualenv) 
  * [The Hitchhiker’s Guide to Python](http://docs.python-guide.org/en/latest/dev/virtualenvs/)
  
  
