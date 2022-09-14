This is project is about learning two things, 
1. volume management in  linux  and, 

2. creating a wordpress website

every web solution or application has  a 3 tier architexture 

1. Presentation Layer (PL): This is the user interface such as the client server or browser on your laptop i.e the frontend
2. Business Layer (BL): This is the backend program that implements business logic i.e the backen
3. Application or Webserver Data Access or Management Layer (DAL): This is the layer for computer data storage and data access. Database Server or File System Server such as FTP server, or NFS Serverj i.e the backend and database layer 

first part of the project is volume management

upon creatin an redhat ec2 instance on AWS, i went head to create 3 volumes attached to my web server. 

![attach_volume](https://github.com/AdebolaM/project6-/blob/main/images/attached%20volume.png?raw=true)

After SSH-ing into the webserver from my terminal, I checked the volumes that I have created 
```
lsblk
```
![lsblk](https://github.com/AdebolaM/project6-/blob/main/images/lsblk.png?raw=true)

the first thing is to create partitions for tthe volumes that were created.

```
sudo gdisk /dev/xvdf
```
* /dev/xvdf is the name of the volume as seen with the lsblk command 

  * n -new volume
  * put the partition number 
  * enter to use up all the space in the volume int the first and second sector   
  *  type of partition -8e00
  * p -check what you have done 
  * w -to write and complete the process

the next thing is ti install Lvm2 - logical volume management *i think*

```
sudo yum install lvm2
````
```
sudo lvmdiskscan
```
to check the volumes available 

* create physical volume -
```sudo pvcreate /dev/xvdf1```
--- name of the partitions create in the step above

* create volume group-
``` sudo vgcreate webdata-vg /dev/xvdf1 /dev/xvdg1 /dev/xvdh1``` vg-webserver is the name of the group 
* create logical volume 
``` 
sudo lvcreate -n apps-lv -L 13G webdata-vg

sudo lvcreate -n logs-lv -L 13G webdata-vg
```
```lsblk```
 to check the whole set up

 ![logical volume system](https://github.com/AdebolaM/project6-/blob/main/images/logical%20volume%20systems.png?raw=true) 

 * make file system 
 ```
 sudo mkfs -t ext4 /dev/webdata-vg/apps-lv
sudo mkfs -t ext4 /dev/webdata-vg/logs-lv
```
create a directory on which the volumes are going to be mounted 

```
sudo mkdir -p /var/www/html
sudo mkdir -p /home/recovery/logs
```
the -p flag is to create the parent directory is its not already present in the system 

mounting apps-lv is pretty easy, just
 ```

sudo mount /dev/webdata-vg/apps-lv /var/www/html/
```
because it is empty 

but to mount logs-lv on var/log/ 
I have to first, copy the content into /home/recovery/logs so that I dont lose them to an overide 
 ```
 sudo rsync -av /var/log/. /home/recovery/logs/
 ```
  then 
  ```
  sudo mount /dev/webdata-vg/logs-lv /var/log
  ```
finally, I can then copy back the content of /var/log.
```
sudo rsync -av /home/recovery/logs/. /var/log
```

the next thing is the update the /etc/fstab

first we get the UUID with ```sudo blkid``
copy the UUID for each of the volumes and then input them into the /etc/fstab file 

``` 
sudo mount -a 
sudo systemctl daemon-reload
```
 to start the configuration 

 ```df -h```
 should show us the whole setup 

 ![setup](https://github.com/AdebolaM/project6-/blob/main/images/set%20up%20for%20server.png?raw=true)

 # Wordpress. 
 then second part of this project 

 first, as usual. I updated Yum and install apache and its dependencies 

 ``` 
 sudo yum -y install wget httpd php php-mysqlnd php-fpm php-json
 ```
 ```
 sudo systemctl enable httpd
sudo systemctl start httpd
```
 to start apache 

 I then install php and its dependencies 

 ```
 sudo yum install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm
sudo yum install yum-utils http://rpms.remirepo.net/enterprise/remi-release-8.rpm
sudo yum module list php
sudo yum module reset php
sudo yum module enable php:remi-7.4
sudo yum install php php-opcache php-gd php-curl php-mysqlnd
sudo systemctl start php-fpm
sudo systemctl enable php-fpm
setsebool -P httpd_execmem 1
```

```
sudo systemctl restart httpd
``` 

the next is to download wordpress and copy it recursively to var/www/html

  ```
  mkdir wordpress
  cd   wordpress
  sudo wget http://wordpress.org/latest.tar.gz
  sudo tar xzvf latest.tar.gz
  sudo rm -rf latest.tar.gz
  cp wordpress/wp-config-sample.php wordpress/wp-config.php
  cp -R wordpress /var/www/html/
  ```

finally its I had to configure SELinux policies because I am using redhat OS. 

  ```sudo chown -R apache:apache /var/www/html/wordpress
  sudo chcon -t httpd_sys_rw_content_t /var/www/html/wordpress -R
  sudo setsebool -P httpd_can_network_connect=1
  sudo setsebool -P httpd_can_network_connnect_db 1
  ```

  the third part of this almost the same as the previous project but there a bit of changes in the names of the files because of the os that i  am using.




 # Database
 For the database, the sample thing is done with the volume system just that one 1 volume is created and its mounted on /db folder. 

 In the database EC2 instance, I installed mysql-server, anabled and restarted it 
 ```
 sudo yum update
sudo yum install mysql-server
sudo systemctl restart mysqld
sudo systemctl enable mysqld
```
* note the d at the end of the mysql

then I configured mysql, created database, created a a user, gave priveleges to the user and flush the priveleges

```
ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'Password';

sudo mysql_secure_installation

sudo mysql
CREATE DATABASE wordpress;
CREATE USER `myuser`@`<Web-Server-Private-IP-Address>` IDENTIFIED BY 'mypass';
GRANT ALL ON wordpress.* TO 'myuser'@'<Web-Server-Private-IP-Address>';
FLUSH PRIVILEGES;
SHOW DATABASES;
exit
```

 I add bind-address=0.0.0.0 
 to 
 ```
 sudo vi /etc/cnf.d/client.cnf
 ```
  file 
  and also 
I update the security of the database server to allow inbound mysql traffic from my webserver'private IP 
 Back  in the webserver instance, 
 I updated the wp-config.php 
 ```
 sudo vi wp-config.php
 ```
  so that it can work with the database 

 ![dbinfo](https://github.com/AdebolaM/project6-/blob/main/images/db%20info.png?raw=true)


 then I connected in to the database with 
```
sudo mysql -u myuser -p -h <database private IP>

```

 and confirm the connect with ```Show database;```

 ![wordpress](https://github.com/AdebolaM/project6-/blob/main/images/web%20show%20the%20db.png?raw=true)


 back on my AWS, I configures the security of my websever to all inbound TCP traffic from port 80 and all IP addresses


http://Web-Server-Public-IP-Address/wordpress/ on my browser to check my website 

![wordpress](https://github.com/AdebolaM/project6-/blob/main/images/wordpress%20site%20.png?raw=true)











   



