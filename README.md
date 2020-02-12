# TODO
- [x] Install Rundeck on Ubuntu
- [x] Test Rundeck with one Node and one simple action
- [x] Activate SSL
- [x] Use MySQL

# Data
* URL (MySQL): https://rundeck.demoshop.rocks:4443
* URL (H2): https://rundeck.demoshop.rocks:5443
* login: admin:admin
* ssh
  * server: ssh localhost -p 2222
  * client: ssh localhost -p 2223
  * server with MySQL: ssh localhost -p 2224
* TerminalUser: roman:qwer
* MySQL
  * root:1234
  * rundeck:1234

# Install Rundeck on Ubuntu
* Install Ubuntu-server in VM
* Install Rundeck as described in this documentation: https://docs.rundeck.com/docs/administration/install/linux-deb.html
```
echo "deb https://rundeck.bintray.com/rundeck-deb /" | sudo tee -a /etc/apt/sources.list.d/rundeck.list
curl 'https://bintray.com/user/downloadSubjectPublicKey?username=bintray' | sudo apt-key add -
sudo apt update
sudo apt install rundeck
```

# Update Rundeck
```
sudo apt update
sudo apt install rundeck
```

# Setup Rundeck to start automatically on system start
* Start/Stop Rundeck
```
sudo systemctl start rundeckd
```

* Autostart Rundeck
```
sudo systemctl enable rundeckd
```

# Sicherheitspunkt 1

# Activate SSL
* For creating keystore run the following command:
```
#Install certbot
sudo apt install certbot

# copy the given txt challenge to _acme-challenge.rundeck.demoshop.rocks (united-domains)
# check the new acme challenge
nslookup -type=txt _acme-challenge.rundeck.demoshop.rocks

# DNS check for letsencrypt
sudo certbot certonly --manual --preferred-challenges dns --email kontakt@fatchip.de --no-eff-email --manual-public-ip-logging-ok --agree-tos -d rundeck.demoshop.rocks

# Create cert.pkcs12 with password adminin
sudo openssl pkcs12 -export -in /etc/letsencrypt/live/rundeck.demoshop.rocks/cert.pem -inkey /etc/letsencrypt/live/rundeck.demoshop.rocks/privkey.pem -CAfile /etc/letsencrypt/live/rundeck.demoshop.rocks/chain.pem -out ~/cert.pkcs12 -name rundeck.demoshop.rocks -caname root

# Import keystore with pasword adminin
sudo keytool -importkeystore -destkeystore /etc/rundeck/ssl/keystore -srckeystore ~/cert.pkcs12 -srcstoretype PKCS12

# Copy the keystore as the truststore for CLI tools
sudo cp /etc/rundeck/ssl/keystore /etc/rundeck/ssl/truststore

```
* sudo nano /etc/rundeck/ssl/ssl.properties
* Check Passwords
* sudo nano /etc/rundeck/framework.properties
```
framework.server.name = rundeck.demoshop.rocks
framework.server.hostname = rundeck.demoshop.rocks
framework.server.port = 4443
framework.server.url = https://rundeck.demoshop.rocks:4443
```
* sudo nano /etc/rundeck/rundeck-config.properties
```
grails.serverURL=https://rundeck.demoshop.rocks:4443
```
* sudo nano /etc/default/rundeckd
```
export RUNDECK_WITH_SSL=true
export RDECK_HTTPS_PORT=4443
export RD_URL=https://rundeck.demoshop.rocks:4443
```
* Run command
```
source /etc/default/rundeckd
```
* sudo nano /etc/hosts
```
127.0.0.1   rundeck.demoshop.rocks
```
* Restart rundeck service
```
sudo systemctl restart rundeckd
```
* Check log
```
tail -f /var/log/rundeck/service.log
```

* If rundeck.demoshop.rocks is not a real DNS hostname, you have to enter it into your /etc/hosts

# Sicherheitspunkt 2

# Use MySQL
## Install MySQL
```
sudo apt install mysql-server
```
* Create root-password
```
# open root shell
sudo -i

#open mysql
mysql -u root

# create root-password
ALTER USER 'root'@'localhost'
IDENTIFIED WITH mysql_native_password BY '1234';
# create database
CREATE database rundeck;
```
* Setup MySQL for connections from outside
  * sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
```
bind-address            = 0.0.0.0
```
* Create new user
```
# grant all permissions to new user
GRANT ALL on rundeck.* to 'rundeck'@'%' identified by '1234';
FLUSH privileges;
QUIT;
```
* Check databases
```
SHOW DATABASES;
```
* Check GRANTS to current user
```
SHOW GRANTS;
```
* Check all GRANTS to all users
```
select * from information_schema.user_privileges;
```

## Setup Rundeck to use MySQL
* sudo nano /etc/rundeck/rundeck-config.properties
```
dataSource.url = jdbc:mysql://rundeck.demoshop.rocks/rundeck?autoReconnect=true&useSSL=false
dataSource.username=rundeck
dataSource.password=1234
dataSource.driverClassName=com.mysql.jdbc.Driver
```

# Sicherheitspunkt 3

# Create new project
* Should be trivial

# Add new Node
* Create new file "resources.xml" with following content (where roman is the user to run commands on the node)
```
<?xml version="1.0" encoding="UTF-8"?>
<project>
<node name="client"
  osFamily="unix"
  username="roman"
  hostname="10.0.2.15"
  />
</project>
```
* In Project Settings click on "Edit Nodes..."->"Add a new Node Source"->"File"
* Enter the path to the previous created file "resources.xml"

# SSH keys
* Create a temporary key pair on localhost (ssh-keygen)
```
ssh-keygen -t rsa -b 2048 -m PEM -f rundeck_key
```
* Click in page header on cog->"Key Storage"->"Add or Upload a Key"
  * Key Type: Private Key
  * Upload File: Select the previous created private key
* Name it (private_key) and click "save"
* Click in page header on cog->"Key Storage"->"Add or Upload a Key"
  * Key Type: Public Key
  * Upload File: Select the previous created public key
* Name it and click "save"
* Don't forget to delete the keys
* In Project Settings click on "Edit Configuration..."->"Default Node Executor"
* Delete the "SSH Key File path"
* Add the path to the previous created private key (private_key) to "SSH Key Storage Path"
* Add the public key to authorized_keys of user in previous specified node (roman)

# Add new Job
* should be trivial

# Migrate from server A to server B
## Migrate certificates
* Copy /etc/rundeck/ssl
## Migrate port and hostname settings
* Copy /etc/rundeck/framework.properties
* Copy /etc/rundeck/rundeck-config.properties
* Copy /etc/default/rundeckd
## Setup rundeck CLI
* Install rundeck-cli
```
sudo apt install rundeck-cli
```

* Add user to group rundeck
```
sudo gpasswd -a roman rundeck
# log out and back in to take effect
```

* Create Rundeck-Client configuration file .rd/rd.conf
```
export RD_URL=https://rundeck.demoshop.rocks:4443
export RD_USER=admin
export RD_PASSWORD=admin
export RD_OPTS="-Djavax.net.ssl.trustStore=/etc/rundeck/ssl/truststore"
```
## Migrate projects
* Export project
```
rd projects archives export -i jobs -i configs -i readmes -i acls -i scm -f ~/<project-name>.jar -p <project-name>
```

* Import project
```
rd projects create -p <project-name>
rd projects archives import -f <project-name>.jar -p <project-name>
```
## Backup files (groups, users, nodes, private keys, projects)
```
sudo tar -czvf rundeck_bkp.tar.gz /etc/rundeck/*.aclpolicy /etc/rundeck/realm.properties /var/rundeck /var/lib/rundeck/.ssh/id_rsa ~/<project-name>.jar /var/www/html/fcMonitorTests/testCases/fcTestCaseDiskSpace.php
tar -xzvf rundeck_bkp.tar.gz -C rundeck_bkp
sudo mkdir -p /var/rundeck/projects/<project-name>/etc/
sudo chown -R rundeck.rundeck /var/rundeck
```
* Copy rundeck_bkp.tar.gz to server B
* Copy as user rundeck all files from rundeck_bkp.tar.gz to its directories stored in the backup file
    * The whole path to /var/rundeck/projects/<project-name>/etc/resources.xml must be owned by user rundeck.rundeck (sudo chown -R rundeck.rundeck /var/rundeck/)
## Import nodes
* Import resources.xml via GUI "PROJECT SETTINGS"-->"Edit Nodes..."-->"Sources"-->"Add a new Node Source"-->"File"
  * Enter file path to "resources.xml"
    * No further settings needed
## Secure apache config
* Disable directory index
  * sudo nano /etc/apache2/sites-available/000-default.conf
```
<Directory /var/www>
    Options -Indexes
    AllowOverride All
    Order allow,deny
    Allow from all
</Directory>
```

# AWS
## Setup base image
* Rundeck
  * Hostname: https://rundeck.demoshop.rocks:4443
  * user: admin:admin
* MySQL:
  * root:1234
  * rundeck:1234
## Setup jvtest (VM die später rundeck.janvanderstorm.de werden soll)
* Rundeck
  * Hostname: https://jvtest.demoshop.rocks:4443
  * user: admin:adminin
* MySQL:
  * root:root
  * rundeck:rundeck

# TODO after clone in AWS
## Change username and password
* rundeck
  * sudo nano /etc/rundeck/realm.properties (clear text or md5)
* rundeck-cli
  * sudo nano ~/.rd/rd.conf
  * source ~/.rd/rd.conf
* MySQL
  * ... MYSQL command to change root and rundeck
```
#open mysql
mysql -u root -p1234

# change root-password
ALTER USER 'root'@'localhost'
IDENTIFIED WITH mysql_native_password BY '<newPW>';

# change rundeck-password
ALTER USER 'rundeck'@'%'
IDENTIFIED WITH mysql_native_password BY '<newPW>';
```
## Change hostname/port
* sudo nano /etc/hosts
* sudo nano /etc/rundeck/framework.properties
* sudo nano /etc/rundeck/rundeck-config.properties
* sudo nano /etc/default/rundeckd
* source /etc/default/rundeckd
## Create new certificate for new hostname
* follow 'Activate SSL'
## Migrate data
* follow 'Migrate from server A to server B'-->'Migrate projects'