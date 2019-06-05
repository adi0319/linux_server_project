# Linux Server Configuration Project
In this project, I took a baseline installation of a Linux distribution on a virtual machine and prepared it to host web applications. This included installing updates, securing it from attack vectors, installing/configuring web and database servers.

## IP Address & Port
54.184.241.170, port 2200

## Instructions

## Create Instance and SSH
1. Create an instance
2. Download the key
3. Rename key, change its permission, and move to `.ssh` directory
   - `mv LightsailDefaultKey-us-west-2.pem lightsail-udacity.pem`
   - `chmod 400 lightsail-udacity.pem`
   - `mv lightsail-udacity.pem ~/.ssh/`
4. SSH into the VM
   - `ssh ubuntu@54.184.241.170 -i ~/.ssh/lightsail-udacity.pem`

## Update Existing Packages
1. `sudo apt-get update`
2. `sudo apt-get upgrade`

## Secure Server
1. Open port 2200 for SSH
   - `sudo vi /etc/ssh/sshd_config`
   - line 5, 22 -> 2200 and save file
2. Reload SSH using `sudo service ssh restart`
3. Back on Amazon Lighsail page, update ports by removing port 22 and adding port 2200 under the Firewall section
4. Confirm SSH port works
   - `ssh ubuntu@54.184.241.170 -p 2200 -i ~/.ssh/lightsail-udacity.pem`
5. Configure UFW, allow port 2200 for SSH, port 80 for HTTP, and port 123 for NTP
   - `sudo ufw status`, should be inactive
   - `sudo ufw allow 2200/tcp`
   - `sudo ufw allow 80/tcp`
   - `sudo ufw allow 123/udp`
   - `sudo ufw default deny incoming`
   - `sudo ufw default allow outgoing`
   - `sudo ufw enable`
   - `sudo ufw status`, should now be active

## Create user `grader`
1. Create new user by running: `sudo adduser grader`
2. Install `finger` to verify that the user was created
   - `sudo apt install finger`
   - `finger grader`
2. Give `grader` sudo access
   - Create a new file for grader in `/etc/sudoers.d/` by using `sudo vi /etc/sudoers.d/grader`
   - Enter `grader ALL=(ALL) ALL:ALL` in the file and save
   - Verify by using `sudo cat /etc/sudoers.d/grader`
3. Create SSH key for `grader`
   - On local machine, run `ssh-keygen`
   - filename: grader_key
4. Back on VM
   - Switch to user: `su - grader`
   - Make SSH dir: `mkdir .ssh`
   - Create file: `vi .ssh/authorized_keys`
   - Enter the public key created back in step 3, file: `grader_key.pub`
   - Save and close the file
5. Change permissions of new directory and file
   - `chmod 700 .ssh`
   - `chmod 644 .ssh/authorized_keys`
6. Restart SSH: `sudo service ssh restart`
7. Verify changes by logging in as `grader`
   - `ssh grader@54.184.241.170 -p 2200 -i ~/.ssh/grader_key`

## Configure Timezone
1. `sudo dpkg-reconfigure tzdata`
2. Select option 'None of the Above'
3. Select option 'UTC'
4. Verify timezone is set to UTC by using the `date` command

## Install Apache and Python mod_wsgi
1. `sudo apt-get install apache2`
2. `sudo apt-get install libapache2-mod-wsgi`
3. Verify by going to `http://54.184.241.170` where the default Apache page should appear

## Install and Configure PostgreSQL
1. `sudo apt-get install postgresql`
2. Default is to not allow remote connections, verify here: `sudo cat /etc/postgresql/9.5/main/pg_hba.conf`
3. User `postgres` was created when downloading package, switch to this user: `sudo su - postgres`
4. Enter PostgreSQL shell using `psql`
5. Create catalog database: `CREATE DATABASE catalog;`
6. Create user `catalog`
   - `CREATE USER catalog WITH PASSWORD 'catalog';`
7. Give `catalog` user access to the catalog database
   - `GRANT ALL PRIVILEGES ON DATABASE catalog to catalog;`
8. Verify changes by using `\l`
9. Quit PostgreSQL: `\q`
10. Exit from user `postgres` using `exit`

## Install Git
1. `sudo apt-get install git`

## Set up Item Catalog Project
1. Change directory: `cd /var/www`
2. Create directory for project: `mkdir project` and `cd` into it
3. Clone the Item Catalog project into this directory
   - `sudo git clone https://github.com/adi0319/items_project.git`
4. Go into the directory: `cd items_project`
5. Change the name of `views.py`: `sudo mv views.py __init__.py`
6. Update the line `engine = create_engine('sqlite:///tech.db')` in `models.py` and `__init__.py` to `engine = create_engine('postgresql://catalog:catalog@localhost/catalog')`
7. Install pip in order to install all dependencies
   - `sudo apt-get install python-pip`
8. Install dependencies
   - `sudo -H pip install sqlalchemy`
   - `sudo apt-get install python-psycopg2`
   - `sudo -H pip install psycopg2`
   - `sudo -H pip install requests`
   - `sudo -H pip install flask`
   - `sudo -H pip install httplib2`
   - `sudo -H pip install oauth2client`
9. Follow the instructions for getting OAuth credentials for Google and Facebook from [here](https://github.com/adi0319/items_project) and add `client_secrets.json` and `fb_client_secrets.json` to `/project/items_project`
10. Update the path of the `client_secrets.json` and `fb_client_secrets.json` to include the full path in `__init.py`
    - `client_secrets.json` to `/var/www/project/items_project/client_secrets.json`
    - `fb_client_secrets.json` to `/var/www/project/items_project/fb_client_secrets.json`
10. Use `sudo python models.py` to create database schema
11. Create wsgi file: `sudo vi /var/www/project/items-project.wsgi`
    - Enter the following in this new file
    ```
    #!/usr/bin/python
    import sys
    import logging
    logging.basicConfig(stream=sys.stderr)
    sys.path.insert(0,"/var/www/project/")

    from items_project import app as application
    application.secret_key = 'DEV_SECRET_KEY'
    ```
11. Create file: `sudo vi /etc/apache2/sites-available/project.conf`
    - Enter the following into this new file:
    ```
    <VirtualHost *:80>
    	ServerName 54.184.241.170
      ServerAlias 54.184.241.170.xip.io
    	ServerAdmin ap739s@att.com
    	WSGIScriptAlias / /var/www/project/items-project.wsgi
    	<Directory /var/www/project/items_project/>
    		Order allow,deny
    		Allow from all
    	</Directory>
    	Alias /static /var/www/project/items_project/static
    	<Directory /var/www/project/items_project/static/>
    		Order allow,deny
    		Allow from all
    	</Directory>
    	ErrorLog ${APACHE_LOG_DIR}/error.log
    	LogLevel warn
    	CustomLog ${APACHE_LOG_DIR}/access.log combined
    </VirtualHost>
    ```
12. Check syntax of conf file
    - `cd /etc/apache2/sites-available`
    - `apachectl configtest`: should return `Syntax OK`
13. Enable the virtual host: `sudo a2ensite project`
14. Restart Apache: `sudo service apache2 restart`
15. Head to `http://54.184.241.170` to items category project app

Note: Items were added manually as login is not currently working.

# Author
Adilene Pulgarin
