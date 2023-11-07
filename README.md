# zabbix-6.4-debbian-bookworm-timescaledb
Install Zabbix 6.4 on Debian 12 (Bookworm) with Postgres 15 and TimescaleDB 2.11.2 with Nginx as Web Frontend

Below instructions were put together after I had lots of trouble trying to get Zabbix 6.4 working on Debian 12 (bookworm) with TimescaleDB.

Below steps are expected to be executed on a Debian system with sudo working. When installing the Debian OS, do not set the root password. This will force Debian installer to install sudo as default.

### References
Ref: https://www.zabbix.com/download?zabbix=6.4&os_distribution=debian&os_version=12&components=server_frontend_agent&db=pgsql&ws=nginx

---
## Server Setup Prep Steps

The server hostname can be set to "**zabbix01"** using

	sudo hostnamectl set-hostname zabbix01

##### Set the Locale

Set the system Locale. For example set to: **en_AU.UTF-8**

	sudo locale-gen --purge en_AU.UTF-8
	sudo dpkg-reconfigure --frontend noninteractive locales

##### Set the Timezone

Replace the timezone to your correct TimeZone.

	sudo ln -fs /usr/share/zoneinfo/Australia/Sydney /etc/localtime
	sudo dpkg-reconfigure -f noninteractive tzdata

Update the packages

	sudo apt-get update -y
	sudo apt-get upgrade -y

Set host banner to show IP Address

	sudo bash -c 'echo "IPv4 Address: \4" >> /etc/issue'
	sudo bash -c 'echo " " >> /etc/issue'

---
### Install Curl package

	sudo apt-get install curl -y

---
### Remove Exim4 and replace with DMA (email daemon)

	sudo apt-get install dma -y

---
### Install FirewallD

	sudo apt install firewalld -y

#Check ethernet adapter name by using: ip addr

	sudo firewall-cmd --get-active-zones

	sudo firewall-cmd --add-service=ssh --permanent
	sudo systemctl start firewalld
	sudo systemctl enable firewalld
	sudo firewall-cmd --reload

---
## Setup Nginx Web Server

	sudo apt install nginx -y

Ref: https://ubiq.co/tech-blog/change-nginx-port-number-ubuntu/

Change Nginx default site port number to 81 as Zabbix will be using port 80 later.

	sudo nano /etc/nginx/sites-enabled/default

Edit to match below:

>  listen 81 default_server;
>   listen [::]:81 default_server;

Run below to check for any errors after editing the config file

	sudo nginx -t

Restart Nginx web server service

	sudo systemctl restart nginx
	sudo systemctl status nginx

Press q to quit out of the Nginx status

Add some permanent firewall rules for Nginx incoming ports

	sudo firewall-cmd --add-service=http --permanent
	sudo firewall-cmd --add-service=https --permanent
	sudo firewall-cmd --add-port=81/tcp  --permanent
	
Reload the firewall rules

	sudo firewall-cmd --reload

##### Nginx Note: 

Alternatively the default Nginx website instance can be removed by running below command. In that case no port changes are required.
	
	sudo rm /etc/nginx/sites-enabled/default

---
## Install TimescaleDB version 2.11.2 with Postgres 15

**Notes:** Postgres config file location @ /etc/postgresql/15/main/postgresql.conf

Note: Do not install the OSS version of TimescaleDB as it does not support Compression in the community license! Also Zabbix 6.4 can only work with TimescaleDB highest version 2.11.2 it will not work with higher version and will fail.


##### Setup the TimecaleDB repo from packagecloud.io

	curl -o packagecloud.io-repo-timescaledb.sh https://packagecloud.io/install/repositories/timescale/timescaledb/script.deb.sh	

	sudo chmod +x packagecloud.io-repo-timescaledb.sh
	sudo ./packagecloud.io-repo-timescaledb.sh

	sudo apt-get update -y

Install TimescaleDB
	
	sudo apt-get install timescaledb-2-2.11.2-postgresql-15=2.11.2~debian12 -y

##### Run the initial timescalDB tune

	sudo timescaledb-tune --quiet --yes

Use psql client to connect to PostgreSQL:

	sudo -u postgres psql

Set the password for the postgres user:
	
	\password postgres

Exit PostgreSQL Shell by typing

	\q

Restart postgres server to load TimescaleDB module

	sudo systemctl restart postgresql

##### Setup TimescaleDB Extension

Connect to postgres shell

	sudo -u postgres psql

At the psql prompt, create an empty database. Our database is called tsdb:

	CREATE database tsdb;

Switch into the tsdb database

	\c tsdb

Create the TimescaleDB extention int he tsdb database

	CREATE EXTENSION IF NOT EXISTS timescaledb WITH VERSION '2.11.2' CASCADE;

If successful you should get a message that includes below line. This confirms version 2.11.2 has been installed succesfully.

> Running version 2.11.2

Next run below command to show the extension 

	\dx

If TimescaleDB is working correctly you should get somehting like below:

                                            List of installed extensions

    Name | Version | Schema | Description --------+---------+------------+-------------------------------------------------------------------- 
    plpgsql | 1.0 | pg_catalog | PL/pgSQL procedural language timescaledb | 2.11.2 | public | Enables scalable inserts and complex queries for time-series data (Community Edition) (2 rows)

Quit out of Postgres command line

	\q

After you have created the extension and the database, you can connect to your database directly using this command:

	psql -U postgres -h localhost -d tsdb

If connection is successful. Quit out of the Postgres shell using:

	\q

----

## Setup Zabbix

##### Install Zabbix from official packages

	sudo wget https://repo.zabbix.com/zabbix/6.4/debian/pool/main/z/zabbix-release/zabbix-release_6.4-1+debian12_all.deb
	sudo dpkg -i zabbix-release_6.4-1+debian12_all.deb
	sudo apt update -y
	sudo apt install zabbix-server-pgsql zabbix-frontend-php php8.2-pgsql zabbix-nginx-conf zabbix-sql-scripts zabbix-agent -y

##### Create postgres zabbix user and database. This will be used by Zabbix to acces the zabbix database

	sudo -u postgres createuser --pwprompt zabbix
	sudo -u postgres createdb -O zabbix zabbix

Import the zabbix base data into the database
	
	zcat /usr/share/zabbix-sql-scripts/postgresql/server.sql.gz | sudo -u zabbix psql zabbix

----------
##### Setup TimescaleDB extension for the Zabbix database

	sudo echo "CREATE EXTENSION IF NOT EXISTS timescaledb WITH VERSION '2.11.2' CASCADE;" | sudo -u postgres psql zabbix

Login to Postgres shell to check Timescaledb in installed correctly

	sudo -u postgres psql zabbix
	
In the Postgres shell run below to show the TimescaleDB Extension is working	

	\dx

You should see "**Running version 2.11.2**" as part of the output

Quit out of the Postgres shell:

	\q

##### Import the timescaledb sql into the zabbix database

	sudo cat /usr/share/zabbix-sql-scripts/postgresql/timescaledb.sql | sudo -u zabbix psql zabbix

Edit Zabbix server config file and set zabbix user postgres DB password

	sudo nano /etc/zabbix/zabbix_server.conf

Change the "**DBPassword=**" to have the correct password for the zabbix user.

> DBPassword=yourpasswordgoeshere


Nano Editor Hints:
Use Ctrl + W to search and Ctrl + X to save and exit, once done with editing.
Hint: Use the keyboard arrow keys to navigate around the text editor.

Edit Zabbix Web GUI Nginx config nginx to set the Web GUI port to 80 (it default to port 8080)

	sudo nano /etc/zabbix/nginx.conf

Edit the below 2 lines to match the actual server name. Replace the server name with the actual Zabbix FQDN server name

> listen          80; server_name     hostname-goes-here;

Use Ctrl + X to save the file and exit the editor.

Restart nginx web server

	sudo systemctl restart nginx

Open firewall ports for Zabbix Server & Agent

	sudo firewall-cmd --add-port=10050/tcp --permanent
	sudo firewall-cmd --add-port=10051/tcp --permanent
	sudo firewall-cmd --reload

Start the Zabbix Server & Web GUI & Agent services

	sudo systemctl restart zabbix-server zabbix-agent nginx php8.2-fpm

Enable the Zabbix Server & Web GUI & Agent services to start on boot
	
	sudo systemctl enable zabbix-server zabbix-agent nginx php8.2-fpm

Start the Web GUI setup steps
Access and continue Zabbix setup via web GUI Start Zabbix server & WebGUI. To get the IP address

	ip addr | grep "inet "

Setup steps for Zabbix GUI
https://www.zabbix.com/documentation/6.4/en/manual/installation/frontend

Default Zabbix web GUI login is: **Admin** / **zabbix**

Make sure to change the Admin password after 1st login to secure the web GUI.

Password can be change  under  Users->Users

After WebGUI setup follow below for basic initial setup steps
https://www.zabbix.com/documentation/6.4/en/manual/quickstart/login

IMPORTANT: Once Zabbix Web GUI setup has been completed. Make sure to change the Admin password and not leave the default password!!!!
