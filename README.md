# ud_fullStack_linux_server_config
A project from Udacity to deploy previous catalog app on a linux server.

The source code for this app is [here](https://github.com/xHa0z/ud_fullStack_itemCatalog/tree/deploy)

This branch is little bit different master, I modified some files for deployment.

## Live Demo
* **SSH port**: 2200
* **IP Address**: 34.205.87.169
* **URL**: [http://ec2-34-205-87-169.compute-1.amazonaws.com/](http://ec2-34-205-87-169.compute-1.amazonaws.com/)

## Configuration
### Create a new user **grader** with _sudo_ permission
1. log in VPS with ssh private key
2. add new user grader `$ sudo adduser grader`
3. create a new grader file under `/etc/sudoers.d` 
4. edit the file with _vim_, and add `grader ALL=(ALL) NOPASSWD:ALL`

### Update packages and change timezone
```bash
$ sudo apt-get install update
$ sudo apt-get install upgrade
$ sudo dpkg-reconfigure tzdata
```
and follow the instruction to select `other` --> `UTC`

### Create key pairs for  grager
1. Generate an encryption key **on your local machine** with: `$ ssh-keygen -f ~/.ssh/ud_grader`.
2. Log into the VPS as *ubuntu* through ssh and create the following file: `$ touch /home/grader/.ssh/authorized_keys`.
3. Copy the content of the *ud_grader.pub* file from your local machine to the */home/grader/.ssh/authorized_keys* on VOS. Then change some permissions:
```bash
$ sudo chmod 700 /home/grader/.ssh
$ sudo chmod 644 /home/grader/.ssh/authorized_keys
$ sudo chown -R grader:grader /home/grader/.ssh
```
4. Now you can login to VPS through ssh: `$ ssh -i ~/.ssh/ud_grader grader@34.205.87.169`.

** Since aws handel password authentication for us, these is no need here to change `sshd_config` file

### Change SSH port and disable _root_ login
change ssh to 2200
```bash
$ sudo vi /etc/ssh/sshd_config
```
chang port from 22 to 2200

change **PermitRootLogin** to no

```$ sudo service ssh restart```

### Configure firewall
only allow incoming connections for SSH (port 2200), HTTP (port 80), and NTP (port 123)
```bash
$ sudo ufw allow 2200/tcp
$ sudo ufw allow 80/tcp
$ sudo ufw allow 123/udp
$ sudo ufw enable
```
### Install apache2, mog_wsgi
```bash
$ sudo apt-get install apache2
$ sudo apt-get installlibapache2-mod-wsgi python-dev
$ sudo a2enmod wsgi
$ sudo service apache2 start
```

### Install git and clone previous project and make it web inaccessbile
```bash
$ sudo apt-get install git
$ git config --global user.name <username>
$ cd /var/www
$ sudo mkdir catalog
$ cd /catalog
$ git clone https://github.com/xHa0z/ud_fullStack_itemCatalog.git
$ git checkout -b deploy
$ sudo cp app.py __init__.py
```
For web inaccessible
`$ cd /var/www/catalog`
find
`$ sudo vim .htaccess`
insert
`RedirectMatch 404 /\.git`


### wsgi and apache config
1. create a .wsgi file unser `/var/www/catalog` as `catalog.wsig`
2. add content
```python
mport sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0, "/var/www/catalog/")

from catalog import app as application
```
### Create virtual environment and install depencencies
```bash
$ sudo apt-get install python-pip
$ sudo pip install virtualenv
$ cd /var/www/catalog
$ sudo virtualenv venv
$ source venv/bin/activate
$ sudo chmod -R 777 venv
$ sudo pip install Flask bleach httplib2 request oauth2client sqlalchemy psycopg2 python-sqlalchemy
```
### Configure apache 
`$ sudo vim /etc/apache2/sites-available/catalog.conf`
add the content
```
<VirtualHost *:80>
    ServerName 34.205.87.169
    ServerAlias http://ec2-34-205-87-169.compute-1.amazonaws.com/
    ServerAdmin admin@34.205.87.169
    WSGIDaemonProcess catalog python-path=/var/www/catalog:/var/www/catalog/venv/lib/python2.7/site-packages
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
```bash
sudo a2ensite catalog
```

### Install and configure PostgreSQL
1. Install dependencies: `$ sudo apt-get install libpq-dev python-dev`.
2. Install PostgreSQL: `$ sudo apt-get install postgresql postgresql-contrib`.
3. Postgres is automatically creating a new user during its installation, whose name is 'postgres'. That is a tusted user who can access the database software. So let's change the user with: `$ sudo su - postgres`, then connect to the database system with `$ psql`.
4. Create a new user called 'catalog' with his password: `# CREATE USER catalog WITH PASSWORD 'dummy';`.
5. Give *catalog* user the CREATEDB capability: `# ALTER USER catalog CREATEDB;`.
6. Create the 'catalog' database owned by *catalog* user: `# CREATE DATABASE catalog WITH OWNER catalog;`.
7. Connect to the database: `# \c catalog`.
8. Revoke all rights: `# REVOKE ALL ON SCHEMA public FROM public;`.
9. Lock down the permissions to only let *catalog* role create tables: `# GRANT ALL ON SCHEMA public TO catalog;`.
10. Log out from PostgreSQL: `# \q`. Then return to the *grader* user: `$ exit`.
11. Inside the Flask application, the database connection is now performed with: 
```python
engine = create_engine('postgresql://catalog:dummy@localhost/catalog')
```
12. Setup the database with: `$ python /var/www/catalog/catalog/setup_database.py`.


### Updata Google Oauth
1. go to google cloud console add ip address and URL for this server
2. re-download the credential json file and copy content to the server
3. **change json path from relative path to absoule path**

### Restart the apache2 and run app
`$ sudo service apache2 restart`

launch your browser and enter the URL: http://ec2-34-205-87-169.compute-1.amazonaws.com/



#### Citation
[DigitalOcean for wsgi](https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps)

[DigitalOcean for postgresql](https://www.digitalocean.com/community/tutorials/how-to-secure-postgresql-on-an-ubuntu-vps)

This [GitHub Repo](https://github.com/iliketomatoes/linux_server_configuration) for `catalog.conf` file and python dependencies



