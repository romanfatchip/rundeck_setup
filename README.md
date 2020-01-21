guak# TODO
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
sudo certbot certonly --manual --preferred-challenges dns --email roman.wedemeier@fatchip.de --no-eff-email --manual-public-ip-logging-ok --agree-tos -d rundeck.demoshop.rocks

# Create cert.pkcs12 with password adminin
sudo openssl pkcs12 -export -in /etc/letsencrypt/live/rundeck.demoshop.rocks/cert.pem -inkey /etc/letsencrypt/live/rundeck.demoshop.rocks/privkey.pem -CAfile /etc/letsencrypt/live/rundeck.demoshop.rocks/chain.pem -out ~/cert.pkcs12 -name rundeck.demoshop.rocks -caname root

# Import keystore with pasword adminin
sudo keytool -importkeystore -destkeystore /etc/rundeck/ssl/keystore -srckeystore ~/cert.pkcs12 -srcstoretype PKCS12

# Copy the keystore as the truststore for CLI tools
sudo cp /etc/rundeck/ssl/keystore /etc/rundeck/ssl/truststore

```
* Edit /etc/rundeck/ssl/ssl.properties
* Check Passwords
* Edit /etc/rundeck/framework.properties
```
framework.server.name = rundeck.demoshop.rocks
framework.server.hostname = rundeck.demoshop.rocks
framework.server.port = 4443
framework.server.url = https://rundeck.demoshop.rocks:4443
```
* Edit /etc/rundeck/rundeck-config.properties
```
grails.serverURL=https://rundeck.demoshop.rocks:4443
```
* Edit /etc/default/rundeckd
```
export RUNDECK_WITH_SSL=true
export RDECK_HTTPS_PORT=4443
export RD_URL=https://rundeck.demoshop.rocks:4443
```
* Run command
```
source /etc/default/rundeckd
```
* Edit /etc/hosts
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
FLUSH privileges;
QUIT;
```
* Create database
```
create database rundeck;
```
* Grant all permissions to new user
```
grant ALL on rundeck.* to 'rundeck'@'localhost' identified by '1234';
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
* Edit /etc/rundeck/rundeck-config.properties
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
* In Project Settings click on "Edit Configuration..."->"Default Node Executor"
* Check the "SSH Key File path" (/var/lib/rundeck/.ssh/id_rsa)
* Create a ssh key pair (ssh-keygen) in the "SSH Key File path" as user "rundeck"
```
sudo -u rundeck ssh-keygen -f /var/lib/rundeck/.ssh/id_rsa
```
* Create new file on node (if not exist) /home/roman/.ssh/authorized_keys (where roman is the user to run commands on the node)
* Enter the corresponding public key to the previous used private key (/var/lib/rundeck/.ssh/id_rsa) into the authorized_keys

# Add new Job
* should be trivial

# Migrate from server A to server B
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

* Give the /etc/rundeck dir to rundeck group
```
sudo chown -R root:rundeck /etc/rundeck
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
rd projects archives export -f local_test.jar -p local_test
```

* Import project
```
rd projects create -p local_test
rd projects archives import -f local_test.jar -p local_test
```
## Migrate users
* Copy
  * /etc/rundeck/*.aclpolicy
  * /etc/rundeck/realm.properties
## Migrate certificates
## Migrate nodes
## Migrate jobs
## Migrate commands
