# LAMP stack on Azure
Deploy an Apache web server, MySQL, and PHP (the LAMP stack) on an Ubuntu VM in Azure.

## Create an Ubuntu VM in Azure
```bash
rgName=rg-lamp
location=canadacentral
# Create resource group
az group create --name $rgName --location $location
# Creates a VM and SSH keys if they do not already exist in a default key location (~/.ssh)
# To use a specific set of keys, use the --ssh-key-value option
vmName=vm-ubuntu-lamp
az vm create \
    --resource-group $rgName \
    --name $vmName \
    --image UbuntuLTS \
    --admin-username azureuser \
    --generate-ssh-keys
# Get the public IP address
publicIP=$(az vm show -d -g $rgName -n $vmName --query publicIps -o tsv)
```

## Enable web traffic to the Ubuntu VM
```bash
# Open port 80 for web traffic
az vm open-port --port 80 -g $rgName --name $vmName
# Verify the NSG rule
nsgName=$(az network nsg list -g $rgName --query "[].name" -o tsv)
az network nsg show -n $nsgName -g $rgName --query "securityRules" -o table
```

## Configure the Ubuntu server
```bash
# Login to the VM
ssh azureuser@$publicIP
# update Ubuntu package sources and install Apache, MySQL, and PHP
# Note the caret (^) which is part of the lamp-server^ package name
sudo apt update && sudo apt install lamp-server^
# Verify apache installation
apache2 -v
curl localhost -I
# Verify php installation
php -v
# create a quick PHP info page to view in a browser
sudo sh -c 'echo "<?php phpinfo(); ?>" > /var/www/html/info.php'
curl localhost/info.php -I
# Verify mysql installation
mysql -V
# secure the installation of MySQL, including setting a root password, run the mysql_secure_installation script
sudo mysql_secure_installation
# login to MySQL to validate connectivity locally
sudo mysql -u root -p
```

## Instal & configure WordPress
```bash
# Install the WordPress package
sudo apt install wordpress
```
### Configure WordPress to use MySQL
```bash
# Create a text file wordpress.sql to configure the MySQL database for WordPress
sudo sensible-editor wordpress.sql
# Add the following commands, substituting a database password of your choice. Save the file
```
```sql
CREATE DATABASE wordpress;
GRANT SELECT,INSERT,UPDATE,DELETE,CREATE,DROP,ALTER
ON wordpress.*
TO wordpress@localhost
IDENTIFIED BY 'password';
```
```bash
# create the database & grant wordpress as a user on the localhost
cat wordpress.sql | sudo mysql --defaults-extra-file=/etc/mysql/debian.cnf
```
Enable remote connectivity to the mysql database to a new user
```bash
# Login to the mysql server
sudo mysql -u root -p
# Grant all permissions on all DB to user "developer" for remote hosts (%)
GRANT ALL ON *.* TO 'developer'@'%' IDENTIFIED BY 'password';
# Reload the grant tables enabling the changes to take effect without restarting mysql service
FLUSH PRIVILEGES;

# By default, MySQL listens for connections only from localhost
bind-address = 127.0.0.1
# It is telling MySQL to only accept local connections (127.0.0.1 / localhost)
# If you know the IP that you are trying to remotely connect from, 
# you should enter it here to restrict remote connections to that IP.
# If you have to allow all IP addresses to connect, comment out the line

# Make the update in the configuration file in the below location
sudo vi /etc/mysql/mysql.conf.d/mysqld.cnf

```
Comment the below line
```bash
# bind-address = 127.0.0.1
```

```bash
# Restart the SQL service
sudo systemctl restart mysql
# Check the SQL service 
systemctl status mysql
```

Validate the connectivity to MySQL
```bash
# Validate connectivity locally from the server
netstat -tan | grep 3306
telnet localhost 3306
telnet 127.0.0.1 3306
# Open port 3306 on the NSG in Azure
az vm open-port --port 3306 -g $rgName --name $vmName --priority 1100
# Validate connectivity from a remote VM 
telnet <ip_address> 3306
# Validate connectivity using MySQL workbench from a remote VM
Refer to the snapshot below
```
![alt txt](/images/mysql-workbench-connectivity.png)

> The private key "id_rsa" to authenticate can be found in the location ~/.ssh

### Configure WordPress to use PHP
```bash
# create the file 
sudo sensible-editor /etc/wordpress/config-localhost.php
# Copy the below php lines to the file & save it
```
```php
<?php
define('DB_NAME', 'wordpress');
define('DB_USER', 'wordpress');
define('DB_PASSWORD', 'password');
define('DB_HOST', 'localhost');
define('WP_CONTENT_DIR', '/usr/share/wordpress/wp-content');
?>
```
```bash
# Move the WordPress installation to the web server document root
sudo ln -s /usr/share/wordpress /var/www/html/wordpress

sudo mv /etc/wordpress/config-localhost.php /etc/wordpress/config-default.php
```
Complete the WordPress setup and publish on the platform. Open a browser and go to http://yourPublicIPAddress/wordpress. Substitute the public IP address of your VM.

## Clean up resources
```bash
# Delete the resource group
az group delete --name $rgName --no-wait --yes
```