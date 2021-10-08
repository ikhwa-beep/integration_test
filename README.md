# System Integration Test Documentation - IKHWANUL MUSLIM EFFENDIE #
This documentation is created to fulfill **Xtremax** System Integration Test.

This documentation will explain for steps taken to achieve the required specification in delivering integrated webserver and a monitoring system.

## Systems Information

**VM1 (Chamilo LMS WebServer)**  | **VM2 (Zabbix Monitoring)**
------------- | -------------
Domain Name: Effendie-setest-web.xtremax.id | Domain Name: Effendie-setest-monitoring.xtremax.id
IP address: 147.139.167.48 | IP address: 149.129.244.49
OS: CentOs 7  | OS: CentOs 7
Username: syseng | Username: syseng
Password: Xtrsyseng2020! | Password: Xtrsyseng2020!

**Database Information**

**VM1 (Chamilo)**  | **VM2 (Zabbix Monitoring)**
------------- | -------------
Database: MariaDB | Database: MariaDB
DB Name: chamilo  | DB Name: zabbix8
Username: root | Username: zabbix8
Password: Xtrsyseng2020! | Password: xtrsyseng2020!

## Installation
### Standard System Checking
This step is important to make sure all services and kernel from both server up to date.  You will need to check and update the both system:

```sh
$ sudo yum -y update 
```

---

### Setup Chamilo on VM 1 

To start setup Chamilo, we need to install and run Nginx, PHP 7, and MariaDB services or usually known as LEMP Stack.

#### Install Nginx as a Webserver

Nginx is a high-performance web server that will be responsible to display the webpages of Chamilo.
To install latest Nginx version, EPEL repository must be installed in CentOS 7 Operating System:

```sh
$ sudo yum install epel-release
```

After EPEL Repository is installed in our VM 1 system, we can can start install Nginx from Yum repository:

```sh 
$ sudo yum install nginx
```

After installation done, start Nginx service:

```sh
$ sudo systemctl start nginx
```
To enable Nginx to start on boot:
```sh
$ sudo systemctl enable nginx
```

#### Install and Configure PHP 7

PHP is used by the Chamilo to proccess the data and display the dynamic contents.
To install PHP 7+ we need to download and install remi repository first:

```sh
$ sudo wget -q https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
$ sudo wget -q http://rpms.remirepo.net/enterprise/remi-release-7.rpm
$ sudo rpm -i remi-release-7.rpm epel-release-latest-7.noarch.rpm
```
For this Integration test, we will install PHP 7.3 from Remi Repository, therefore we need to enable the repository using yum-utils:

```sh
$ sudo yum-config-manager --enable remi-php73
```

> If bash: yum-config-manager: command not found error shown, please install yum-utils first using yum install yum-utils.

Now we can directly installing PHP using yum:

```sh
$ syum install php php-mcrypt php-cli php-gd php-curl php-mysql php-ldap php-zip php-fileinfo php-fpm
```

To check and confirm that PHP is successfully installed:

```sh
$ php --version
```
![image](https://user-images.githubusercontent.com/85183201/136519049-b652509f-03e8-499d-b79f-404b345e2c31.png)

Open `/etc/php-fpm.d/www.conf` configuration file using nano or preferable text editor like vi or vim:

```sh
$ sudo nano /etc/php-fpm.d/www.conf
```

find the `user` and `group` directive/variables. If the variables are set to apache, please change these to `nginx`.
![image](https://user-images.githubusercontent.com/85183201/136519837-53c25ab2-95cb-4dd5-ae9e-2f700c266e1d.png)

Next, find the `listen` directive/variables and changes the variable to `/var/run/php-fpm/php-fpm.sock;`

![image](https://user-images.githubusercontent.com/85183201/136520026-5f5ca659-1972-44d4-9efd-0ff9e704e820.png)

Lastly, please change the owner and group settings for the socket file in Listen directive by changing the `listen.owner` `listen.group` and `listen.mode` values in configuration file.

![image](https://user-images.githubusercontent.com/85183201/136520321-dbba6ea8-639b-4f16-bbbb-667c6d0661ae.png)

Save and close the configuration file.

To start and enable php-fpm service:

```sh 
$ sudo systemctl start php-fpm
``` 

Finally, our PHP 7+ environment is now ready.

#### Install and configure MariaDB 

To properly install and run chamilo, we need to install database server. 
Install MariaDB database server:

```sh
$ sudo yum -y install mariadb
```

Start the mariaDB server and enable it to start on boot:

```sh
$ sudo systemctl start mariadb
$ sudo systemctl enable mariadb
```

Once the installation of mariaDB and MariaDB is started, execute the `mysql_secure_installation` and follow basic security recommendation and set the credentials for root. Please enter Yes to all except for remote root access.

We need to create and configure new database and user for Chamilo installation:

```sh
$ mysql -u root -p

MariaDB [(none)]> CREATE DATABASE chamilo;
MariaDB [(none)]> CREATE USER 'chamilo'@'localhost' IDENTIFIED BY 'PASSWORD';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON chamilo.* TO 'chamilo'@'localhost';
MariaDB [(none)]> FLUSH PRIVILEGES
MariaDB [(none)]> exit;
```

>change the 'PASSWORD' with desired secure password

#### Download and Install Chamilo

Check the latest stable version from  [Chamilo official website](https://chamilo.org/en/download/)
For this System Integration test documentation, we will use version [1.11.16](https://github.com/chamilo/chamilo-lms/releases/download/v1.11.16/chamilo-1.11.16.zip)
Download Chamilo to our VM 1 server:

```sh
$ sudo wget https://github.com/chamilo/chamilo-lms/releases/download/v1.11.16/chamilo-1.11.16.zip
```
Unpack the downloaded archieve. For example, the file will be extracted to `/var/www/html/`:

```sh
$ sudo unzip chamilo-1.11.8-php7.zip -d /var/www/html/
```
> If unzip not yet installed, please install first using `sudo yum install unzip`

Rename the directory to simpler name:

```sh
$ cd /var/www/html
$ sudo mv chamilo-1.11.8-php7 chamilo1
```

Change the ownership of the Chamilo files:

```sh
$ sudo chown -R nginx:nginx chamilo1
```

#### Configure Nginx to Proccess Chamilo Files

Edit `/etc/nginx/nginx.conf` configuration file:

```sh
$ sudo nano /etc/nginx/nginx.conf
```

change the server block:

```sh
server {
    server_name  effendie-setest-web.xtremax.id www.effendie-setest-web.xtremax.id;

    root   /var/www/html/chamilo1;
    index index.php index.html index.htm;

    client_max_body_size 100M;

    location / {
          try_files $uri /index.php$is_args$args;
    }
    location ~ \.php$ {
        try_files $uri =404;
        fastcgi_pass unix:/var/run/php-fpm/php-fpm.sock;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }
}
```

#### Install SSL Certificate and redirect http to https

To able installing SSL Certificate and redirect the pages to secure pages, we will use Certbot with Let's Encrypt Certificate.

** Install Snapd package to install certbot:

```sh
$ sudo yum install snapd
```

Then enable systemd unit that manage snapd main communication socket:

```sh
$ sudo systemctl enable --now snapd.socket
```

To enable classic snapd support, create this symbolink:

```sh
$ sudo ln -s /var/lib/snapd/snap /snap
```

Ensure  up dated and latest version of snapd:

```sh
$ sudo snap install core; sudo snap refresh core
```

** Install Certbot

To install certbot, just run this command:

```sh
$ sudo snap install --classic certbot
$ sudo ln -s /snap/bin/certbot /usr/bin/certbot
```

Get SSL Certificate and let Certbot edit Nginx Configuration file automatically, turning on HTTPS access:

```sh
$ sudo certbot --nginx
```

![image](https://user-images.githubusercontent.com/85183201/136525827-65913e6a-f93b-4311-a8d3-d83ae5aeefe5.png)

**Credentials for Chamilo Admin:**

User: admin

Password: Xtrsyseng2020!

---

### Setup Zabbix Monitoring System on VM 2
