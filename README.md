# Linux Server Configuration Project

We are deploying a Flask application onto Digital Ocean. The aim of the project was to host a web application and configure it to serve our web app and protext it against basic attack vectors. Following this documentation will give you steps in creating your own instance to the application. Currently using ubuntu 16.04.3 x64

Grader IP: ssh grader@104.131.146 -p 2200

IP address: 104.131.146
Port: 2200

## Creating Digital Ocean Instance for ubuntu
1. Signup into [Digital Ocean](https://cloud.digitalocean.com/registrations/new) by creating email and password to Digital Ocean.

2. Once you are logged into the site, on the top right corner, there is a button called "Create" that dropdowns to several options. Click "Create Droplet".
![Create Droplet](/images/createdroplet.png)

3. Choose the Ubuntu 16.04.3 x64 image.
![Ubuntu](/images/step1.png)

4. Select size (go for $5/mo since not using that much server space).
![Size](/images/step2.png)

5. Select the datacenter region that is nearest to you.
![Datacentre](/images/step3.png) 

6. Leave the rest of the options unchecked. We will be creating an SSH key later.
![SSHKey](/images/step4.png)

7. You can name your instance or leave it to its default hostname. 

Once we have created our ubuntu image, we can now start configuring the server to deploy our flask application.

## Accessing the instance
Please refer to the Digital Ocean's Initial Setup to access server instance: [Server](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-14-04).

1. To connect to the server, log in as `root` user using the following command: 
```
local$ ssh root@SERVER_IP_ADDRESS
```

You are successfully able to access the instance.

## Updating the server

Since running ubuntu, only need to update server with following commands:
```
sudo apt-get update
sudo apt-get upgrade
```

## Creating new user with sudo permissions

Creating new user called grader:

`sudo adduser grader`

It will ask for a password. Type whatever is works best for you. Once user is greate, run this command to give grader sudo privelages:

`sudo visudo`

Once in visudo file, go to section user privilege specification

`root ALL=(ALL:ALL) ALL`

Once you come across root user in the file, you will need to do the same for the grader. Please type the following:

`grader ALL=(ALL:ALL) ALL`

To check if the login works for the grader, type this command: `sudo login grader`. We have created our new user.

## Configuring SSH to non-default port

In the Digital Ocean home page of the droplet, on the lefthand side, click 'Networking'.
![Networking](/images/networking.png)

Scroll down till you see Firewalls and start configuring the ports by doing the following
![Firewall](/images/firewall.png)

Follow the text below where `#` means commented out.
![Options](/images/options.png)
```
# What ports, IPs and protocols we listen for
# Port 22
Port 2200

# Authentication:
LoginGraceTime 120
#PermitRootLogin prohibit-password
PermitRootLogin no
StrictModes yes

# Change to no to disable tunnelled clear text passwords
PasswordAuthentication no
```
Once the changes are made, restart ssh: `sudo service ssh restart`. Any user can login into the machine using a specified port: `sudo user@PublicIP -p 2200`

## Generating a Key Pair

To generate new key pair, enter the following command on terminal of local machine:
`local$ ssh-keygen`

Assuming your local user is called "localuser", the following output will be 
`Generating public/private rsa key pair. Enter file in which to save the key (/User/localuser/.ssh/id_rsa)`

Hit return to accept file name and path (or enter new name)

Once generated SSH key pair, will want to copy public key to new server. Run the `ssh-copy-id` script for specifying user and IP address of server you want to install key on like this:
`local$ ssh-copy-id dem@SERVER_IP_ADDRESS`

Once provided password at prompt, public key will be added to remote user's `.ssh/authorized_keys` file. The corresponding private key can now be used to log into the server.

## Configuring Firewall
Need to configure firewall rule using UFW. Check to see if ufw is active: `sudo ufw status`. If not active, add the following commands:
```
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 2200/tcp
sudo ufw allow 80/tcp 
sudo ufw allow 123/udp
```

Once rules enabled: `sudo ufw enable` and check the status to see what rules are active.

## Configure Timezone for server
`sudo dpkg-reconfigure tzdata`

Chose none of the above and choose UTC. The server by default is on UTC.

## Install Apache, Git, and flask
We will be installing apache on our server. Type the following command:
`sudo apt-get install apache2`

If already installed apache, welcome page will appear when use your PublicIP. Install mod wsgi, python setup tools, and pytho-dev
`sudo apt-get install libapache2-mod-wsgi python-dev`

We need to enable mod wsgi if it isn't enabled: `sudo a2enmod wsgi`

Setup wsgi file and sites-available conf file for application. Create WSGI file in `path/to/the/application directory`

```
import sys 
import logging

logging.basicConfig(stream=sys.stderr)
sys.path.insert(0, '/var/www/items_catalog')

from finalProject import app as application
application.secret_key='super_secret_key'
``` 

Please note finalProject.py is my python file. Put in the file wherever you housed your application logic.

To set up virtual host file: 
`cd /etc/apache2/sites-available`
`sudo vim items_catalog.conf`

```
<VirtualHost *:80>
     ServerName  PublicIP
     ServerAdmin root@YOUR_PUBLIC_IP
     #Location of the items-catalog WSGI file
     WSGIScriptAlias / /var/www/items_catalog/catalog.wsgi
     #Allow Apache to serve the WSGI app from our catalog directory
     <Directory /var/www/itemsCatalog/vagrant/catalog>
          Order allow,deny
          Allow from all
     </Directory>
     #Allow Apache to deploy static content
     <Directory /var/www/items_catalog/static>
        Order allow,deny
        Allow from all
     </Directory>
      ErrorLog ${APACHE_LOG_DIR}/error.log
      LogLevel warn
      CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

## Git

`sudo apt-get install git`

Clone repository into apache directory. In cd /var/www

```
mkdir items_catalog
mkdir /var/www/items_catalog

sudo git clone https://github.com/ltphan/FSND-Virtual-Env.git
```

## Flask
Do these commands:

```
sudo apt-get install python-pip python-flask python-sqlalchemy python-psycopg2
sudo pip install oauth2client requests httplib2
```

## Install PostGreSql
Install PostGreSQL: `sudo apt-get install postgresql`. To make sure not remote are not allowed, checking the following configuration file: `sudo vim /etc/postgresql/9.5/main/pg_hba.conf`

## Create database
`sudo su - postgres` then type `psql` as postgre user.

Do the following commands:
```
postgres=# CREATE USER catalog WITH PASSWORD 'catalog';
postgres=# CREATE DATABASE catalog WITH OWNER catalog;
```

Connect to catalog database: `\c catalog`
```
catalog=# REVOKE ALL ON SCHEMA public FROM public;
catalog=# GRANT ALL ON SCHEMA public TO catalog;
```

Exit postgres and postgres user
```
postgres=# \q
postgres@PublicIP~$ exit
```
Then update database setup and application python files to illustrate new engine connection:
`engine = create_engine('postgresql://catalog:catalog@localhost/catalog')`

Run `sudo python database_setup.py`

## Oauth Client Login
To setup Oauth client, update the path in finalProject.py program. Update the client id and oauth_flow

```
CLIENT_ID = json.loads(
    open('/var/www/item_catalog/client_secrets.json', 'r').read())['web']['client_id']

oauth_flow = flow_from_clientsecrets('/var/www/item_catalog/client_secrets.json', scope='')
```

## Bibiliography
1. [Initial Server Setup](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-14-04)
2. [Hari Vedam](https://github.com/harushimo/linux-server-configuration/blob/master/README.md)
3. [Connect Droplet with SSH](https://www.digitalocean.com/community/tutorials/how-to-connect-to-your-droplet-with-ssh)
4. [Additional Resource](https://www.digitalocean.com/community/tutorials/additional-recommended-steps-for-new-ubuntu-14-04-servers)



