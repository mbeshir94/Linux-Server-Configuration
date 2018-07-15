# Linux server configuration project

### About

In this project, a Linux virtual machine is configurated to host the Item Catalog application.

The application can be visited from http://bookcatalog.beshir.me/.

## Useful info

IP address: 18.184.23.32.

Accessible SSH port: 2200.

Application URL: [http://bookcatalog.beshir.me/](http://bookcatalog.beshir.me/).

## Configuration Steps

### 1 - Create a new user named *grader*, grant this user sudo permissions and prevent root remotely login.

1. Log into the remote VM as *root* user through ssh: `$ ssh ubuntu@18.184.23.32`.
2. Add a new user called *grader*: `$ sudo adduser grader`.
3. Create a new file under the suoders directory: `$ sudo nano /etc/sudoers.d/grader`. Fill that newly created file with the following line of text: "grader ALL=(ALL:ALL) ALL", then save it.
4. Prevent root remotely login, PermitRootLogin property should be set to no in /etc/ssh/sshd_config

### 2 - Update installed packages

1. `$ sudo apt-get update`.
2. `$ sudo apt-get upgrade`.
3. Install *finger*, a utility software to check users' status: `$ apt-get install finger`.

### 3 - Configure the local timezone to UTC

1. Open time configuration dialog and set it to UTC with: `$ sudo dpkg-reconfigure tzdata`.
2. Install *ntp daemon ntpd* for a better synchronization of the server's time over the network connection: `$ sudo apt-get install ntp`.

### 4 - Configure the key-based authentication for *grader* user

1. Generate an encryption key with: `ssh-keygen`.
2. Log into the remote VM as *root* user through ssh and create the following file: `$ touch /home/grader/.ssh/authorized_keys`.
3. Copy the content of the *udacity_key.pub* file from your local machine to the */home/grader/.ssh/authorized_keys* file you just created on the remote VM. Then change some permissions:
	1. `$ sudo chmod 700 /home/grader/.ssh`.
	2. `$ sudo chmod 644 /home/grader/.ssh/authorized_keys`.
	3. Finally change the owner from *root* to *grader*: `$ sudo chown -R grader:grader /home/grader/.ssh`.
4. Now you are able to log into the remote VM through ssh with the following command: `$ ssh -i ~/SSHPrivateKey.txt grader@18.184.23.32`.

### 5 - Enforce key-based authentication
1. `$ sudo nano /etc/ssh/sshd_config`. Find the *PasswordAuthentication* line and edit it to *no*.
2. `$ sudo service ssh restart`.

### 6 - Change the SSH port from 22 to 2200
1. `$ sudo nano /etc/ssh/sshd_config`. Find the *Port* line and edit it to *2200*.
2. `$ sudo service ssh restart`.
3. Now you are able to log into the remote VM through ssh with the following command: `$ ssh -i ~/SSHPrivateKey.txt -p 2200 grader@18.184.23.32`.

### 7 - Configure the Uncomplicated Firewall (UFW)

Project requirements need the server to only allow incoming connections for SSH (port 2200), HTTP (port 80), and NTP (port 123).

1. `$ sudo ufw default deny incoming`
2. `$ sudo ufw default allow outgoing`
3. `$ sudo ufw allow 2200/tcp`.
4. `$ sudo ufw allow 80/tcp`.
5. `$ sudo ufw allow 123/udp`.
6. `$ sudo ufw enable`.

### 8 - Install Apache, mod_wsgi

1. `$ sudo apt-get install apache2`.
2. Mod_wsgi is an Apache HTTP server mod that enables Apache to serve Flask applications. Install *mod_wsgi* with the following command: `$ sudo apt-get install libapache2-mod-wsgi python-dev`.
3. Enable *mod_wsgi*: `$ sudo a2enmod wsgi`.
4. `$ sudo service apache2 start`.

### 9 - Install Git

1. `$ sudo apt-get install git`.

### 10 - Clone the Catalog app from Github

1. `$ cd /var/www`. Then: `$ sudo mkdir item-catalog`.
2. Change owner for the *catalog* folder: `$ sudo chown -R grader:grader item-catalog`.
3. Move inside that newly created folder: `$ cd /item-catalog` and clone the catalog repository from Github: `$ git clone https://github.com/mbeshir94/ItemCatalog.git`.
4. Make a *catalog.wsgi* file to serve the application over the *mod_wsgi*. That file should look like this:

```python
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0, "/var/www/item-catalog/")

from item-catalog import app as application
```

### 11 - Install  Flask and the project's dependencies

1. Install project's dependencies: `$ sudo apt-get install python-flask python-httplib2 python-requests python-oauth2client python-sqlalchemy python-psycopg2`. 

### 12 - Configure and enable a new virtual host

1. Create a virtual host conifg file: `$ sudo nano /etc/apache2/sites-available/webtool.conf`.
2. Paste in the following lines of code:
```
<virtualhost *:80>

        ServerName my.webtool

    WSGIDaemonProcess webtool user=www-data group=www-data threads=5 home=/var/$
    WSGIScriptAlias / /var/www/item-catalog/webtool.wsgi

    <directory /var/www/item-catalog>
        WSGIProcessGroup webtool
        WSGIApplicationGroup %{GLOBAL}
        WSGIScriptReloading On
        Order deny,allow
        Allow from all
    </directory>
</virtualhost>

```

3. Enable the new virtual host: `$ sudo a2ensite item-catalog`.

### 13 - Install and configure PostgreSQL

1. Install PostgreSQL: `$ sudo apt-get install postgresql`.
2. Postgres is automatically creating a new user during its installation, whose name is 'postgres'. That is a trusted user who can access the database software. So let's change the user with: `$ sudo su - postgres`, then connect to the database system with `$ psql`.
3. Create a new user called 'catalog' with his password: `# CREATE USER catalog WITH PASSWORD 'catalog123';`.
4. Create the 'BookCatalog' database: `# CREATE DATABASE postgres;`.
5. Connect to the database: `# \c BookCatalog`.
6. Log out from PostgreSQL: `# \q`. Then return to the *grader* user: `$ exit`.
7. Inside the Flask application, the database connection is now performed with: 
```python
engine = create_engine('postgresql://postgres:postgres@localhost/BookCatalog')
```
8. Setup the database with: `$ python /var/www/item-catalog/database_setup.py`.
9. To prevent potential attacks from the outer world we double check that no remote connections to the database are allowed. Open the following file: `$ sudo nano /etc/postgresql/9.3/main/pg_hba.conf` and edit it, if necessary, to make it look like this: 
```
local   all             postgres                                peer
local   all             all                                     peer
host    all             all             127.0.0.1/32            md5
host    all             all             ::1/128                 md5
```

### 14 - Restart Apache to launch the app
1. `$ sudo service apache2 restart`.

#### Special thanks to [*stueken*](https://github.com/stueken) who wrote a really helpful README in his [repository](https://github.com/stueken/FSND-P5_Linux-Server-Configuration).
