# Guide for on-prem hosting of Znuny within a Debian LXC using Proxmox VE
This guide assumes you already have PVE installed.
<br>
# Getting Started by Installing Debian LXC
In order to get started we'll start of by using a script provided by <a href="https://tteck.github.io/Proxmox/">Proxmox VE Helper-Scripts</a> to install a Debian LXC.
Head on over to your PVE WebUI and open it's shell then paste and enter this script.
```
bash -c "$(wget -qLO - https://github.com/tteck/Proxmox/raw/main/ct/debian.sh)"
```
-Once the script starts you will be greeted with a prompt asking "This will create a New Debian LXC. Proceed?" select yes and hit enter.<br>

-In the next prompt you'll be asked if you want to use the default settings, you may want to set some specifics such as an id, hostname, and static IP, so select advanced and hit enter.<br>

-The next prompt we need to make a selection on asks us to choose a distribution, select debian by using the arrow keys and hitting space bar, hit enter to accept.<br>

-Select Bookworm for the version.<br>

-Container type we're going to want to do unprivileged.<br>

-Now create a password and then confirm it.<br>

-Now we set a container ID, if you plan on running a lot of containers and virtual machines inside your Proxmox I recommend coming up with some sort of ID scheme.<br>

-For the hostname I would go with znuny to help identify it in your list.<br>

-A 2GB disk size should be enough, we can always increase this later if we need.<br>

-We'll probably want at least 2 cores, if needed we can insrease this later as well.<br>

-For RAM we likely want at least 2GB so enter 2048, this is another setting we can increase later if we need.<br>

-The rest of the settings depends on you, and can mostly be left blank or default. Don't forget to set a static IP however and include your subnet mask at the end(usually /24).<br>

-After this install finished we can proceed to installing Znuny.

# Installing Znuny
We'll be following Znuny's installation guide for Ubuntu / Debian found <a href="https://doc.znuny.org/znuny_lts/releases/installupdate/install.html#">here</a>.
<br>
After we've got our Debian LXC installed we'll need to access it's shell. You can SSH in or use PVE's WebUI.<br>
We'll start up by making sure we're up to date with
```
apt update
```
If we need to upgrade we'll run
```
apt full-upgrade
```
Next we'll install a few dependancies such as apache, mariadb, and cpanminus
```
apt install -y apache2 mariadb-client mariadb-server cpanminus
```
Now we'll install the Znuny source.<br>
- We'll start by downloading the source. Let's cd into /opt and download.
```
cd /opt
wget https://download.znuny.org/releases/znuny-latest-6.5.tar.gz
```
- Now we'll extract the download.
```
tar xfz znuny-latest-6.5.tar.gz
```
- Next we need to create a symlink. Make sure to change znuny-6.5.11 to match the version you downloaded.
```
ln -s /opt/znuny-6.5.11 /opt/otrs
```
- Then we need to add the user otrs.
```
useradd -d /opt/otrs -c 'Znuny user' -g www-data -s /bin/bash -M -N otrs
```
- We'll now set some permissions.
```
/opt/otrs/bin/otrs.SetPermissions.pl
```
- Now we'll rename the default cron jobs as the otrs user.
```
su - otrs
cd /opt/otrs/var/cron
for foo in *.dist; do cp $foo `basename $foo .dist`; done
```
- We'll need to switch back to the root user.
```
su
```
- Now we need to install the required pearl modules.
```
apt -y install apache2 mariadb-client mariadb-server cpanminus libapache2-mod-perl2 libdbd-mysql-perl libtimedate-perl libnet-dns-perl libnet-ldap-perl libio-socket-ssl-perl libpdf-api2-perl libsoap-lite-perl libtext-csv-xs-perl libjson-xs-perl libapache-dbi-perl libxml-libxml-perl libxml-libxslt-perl libyaml-perl libarchive-zip-perl libcrypt-eksblowfish-perl libencode-hanextra-perl libmail-imapclient-perl libtemplate-perl libdatetime-perl libmoo-perl bash-completion libyaml-libyaml-perl libjavascript-minifier-xs-perl libcss-minifier-xs-perl libauthen-sasl-perl libauthen-ntlm-perl libhash-merge-perl libical-parser-perl libspreadsheet-xlsx-perl libcrypt-jwt-perl libcrypt-openssl-x509-perl jq
```
- Then.
```
cpanm install Jq
```
- Now we need to configure MariaDB.
```
nano /etc/mysql/mariadb.conf.d/50-znuny_config.cnf
```
Make the file look like this then exit with ctrl + x, save with y then enter.
```
[mysql]
max_allowed_packet=256M
[mysqldump]
max_allowed_packet=256M

[mysqld]
innodb_file_per_table
innodb_log_file_size = 256M
max_allowed_packet=256M
character-set-server  = utf8
collation-server      = utf8_general_ci
```
- Now enable MariaDB
```
systemctl enable --now mariadb
```
Now we need to configure the Webserver.
- First enable apache2.
```
systemctl enable --now apache2
```
- Check apache version with the following command.
```
apache2 -v
```
if you get a command not found error we need to add a path with the following command.
```
export PATH=$PATH:/usr/sbin
```
- Then we need to create another system link to some sample configs.
```
ln -s /opt/otrs/scripts/apache2-httpd.include.conf /etc/apache2/conf-available/zzz_znuny.conf
```
- Now we need to enable some modules within apache2.
```
a2enmod perl headers deflate filter cgi
a2dismod mpm_event
a2enmod mpm_prefork
a2enconf zzz_znuny
```
- Restart apache
```
systemctl restart apache2
```
- Create a database and user in MariaDB by running these commands, replacing znuny_user with your desired username, and your_password with your desired password.
```
CREATE DATABASE otrs;
CREATE USER 'znuny_user'@'localhost' IDENTIFIED BY 'your_password';
GRANT ALL PRIVILEGES ON otrs.* TO 'znuny_user'@'localhost';
FLUSH PRIVILEGES;
exit
```
- Restart MariaDB
```
systemctl restart mariadb
```
We should now be able to access the web installer script for Znuny by visiting http://x.x.x.x/otrs/installer.pl where x.x.x.x is the static IP you created when setting up the Debian LXC.<br>
