# 6.-wordpress-web-solution
msql, apache, php, wordpress

In this project we will prepare a storage infrastructure on two Linux servers and implement a basic web solution using WordPress. We will use mysql database for its backend Relational Database Management System.

This project will display the full concept of the three-tier architecture upon which web solutions are biult:

1.  **Presentation Layer**: This is the clients browser, smartphone or laptop.

2: **Business Layer**: This stands for the backend component that implements business logic. Theses are the servers or application.

3. **Data Access or Management Layer (DAL)**: This is the layer for computer data storage and data access. Database Server or File System Server such as FTP server, or NFS Server.

![Screenshot from 2023-03-10 18-35-26](https://user-images.githubusercontent.com/77943759/224384376-6db9c34c-aaba-4071-8d02-e27ece791363.png)

REDHAT Os will be used for this project

## **CONFIGURE 2 CENTOS SERVERS AND LAUNCH THEM**

![instance](https://user-images.githubusercontent.com/77943759/224384870-60f57ea8-6611-42b2-ae8d-c322c93ceaa6.png)

## **For the Database server(DB-cent)

Navigate to the volume section and create 3 volumes, 10G each in the same availability zone of the created insances

![createvol](https://user-images.githubusercontent.com/77943759/224387203-536ea20d-6eae-4547-a370-74a6b6f7b026.png)

Click on the created volumes, one after the other and attach them to the Database server

Open terminal and verify for the attached volumes

`lsblk`

![lsblkdb](https://user-images.githubusercontent.com/77943759/224387883-29c95367-5fc8-4a98-aa2e-9ff1f9baea13.png)

To see all mounts and free space on your server: Run:

`df -h`

Create a single partition on each of the 3 disks using gdisk utility

`sudo gdisk /dev/xvdf`
`sudo gdisk /dev/xvdg`
`sudo gdisk /dev/xvdh`

![gdiskxvdf](https://user-images.githubusercontent.com/77943759/224388673-e9e3b67b-07cf-442b-9c67-7e17f0764d59.png)

Type n to create new partition, 1 to indicate the number, p to check the partition created and w to write the partition and type y for yes to complete the partition

Run `lsblk` to see the new partitions

![lsblkpart](https://user-images.githubusercontent.com/77943759/224391561-33f8bfd6-addc-438f-8013-52fe03006008.png)

To check the available volume, we will user lvm2

Install lvm3. Run: 

`sudo yum install lvm2`
Check the available volume . Run:

`sudo lvmdiskscan`

![physicalvol](https://user-images.githubusercontent.com/77943759/224396094-a8911403-289d-47e7-9fc5-3a9f19ce6b40.png)


Use pvcreate utility to mark each of 3 disks as physical volumes (PVs) to be used by LVM
Run:
```
sudo pvcreate /dev/xvdf1
sudo pvcreate /dev/xvdg1
sudo pvcreate /dev/xvdh1
```
Confirm the volumes are created, Run:

![sudopvs](https://user-images.githubusercontent.com/77943759/224393119-67ee02a7-5345-4b31-b745-3012faec6508.png)

This shows the physical volumes have been marked

Create a volume group using vgcreate, name of the group here is webdata-vg

`sudo vgcreate webdata-vg /dev/xvdf1 /dev/xvdg1 /dev/xvdh1`

Run `sudo vgs` to confirm the created volume group

![vgcreatendsudo](https://user-images.githubusercontent.com/77943759/224394024-44199132-35d5-47ce-bdf2-26c0c8335c23.png)

Next, we use lvcreate to create 2 logical volumes to store web data and logs. Apps-lv and logs-lv respectively

```
sudo lvcreate -n apps-lv -L 14G webdata-vg
sudo lvcreate -n logs-lv -L 14G webdata-vg
```
Run:

`sudo lvs` to see the logical volumes created 

![lvndsudo](https://user-images.githubusercontent.com/77943759/224395538-54491395-764c-42f9-93dd-3a7a733908de.png)

Run `sudo lsblk` to check the entire created volumes

![createsummary](https://user-images.githubusercontent.com/77943759/224396585-84a043a1-5b15-4661-bd51-dffc9bdeb173.png)

Use mkfs.ext4 to format the logical volumes with ext4 filesystem

```
sudo mkfs -t ext4 /dev/webdata-vg/apps-lv
sudo mkfs -t ext4 /dev/webdata-vg/logs-lv
```


 Create /var/www/html directory to store website files

 `sudo mkdir -p /var/www/html`

Create /home/recovery/logs to store backup of log data

`sudo mkdir -p /home/recovery/logs`

Mount /var/www/html on apps-lv logical volume

`sudo mount /dev/webdata-vg/apps-lv /var/www/html/`

We use rsync utility to backup all the files in the log directory /var/log into /home/recovery/logs. We do this before mouting on the /var/log because evrything on the directory will be formatted upon mounting leading to loss of valuable data.

`sudo rsync -av /var/log/. /home/recovery/logs/`

Now, we mount on /var/log directory

`sudo mount /dev/webdata-vg/logs-lv /var/log`

Return the backed files from /home/recovery/logs

`sudo rsync -av /home/recovery/logs/. /var/log`

Next we update the /etc/fstab file

Run

`sudo blkid`

![blkid](https://user-images.githubusercontent.com/77943759/224403269-00121db0-ef75-490a-bd6a-ff99eecaba91.png)

Next, update /etc/fstab

`sudo vi /etc/fstab`

![fstabedit](https://user-images.githubusercontent.com/77943759/224403869-f0deb8af-0072-4140-9c1d-b104cdce2fd7.png)

Mount and reload daemon

```
sudo mount -a
sudo systemctl daemon-reload
```
Use `df -h` to see the total

## **PREPARE THE DATABASE SERVER**

Repeat the same steps as for the Web Server, but instead of apps-lv create db-lv and mount it to /db directory instead of /var/www/html/.

## **Install WordPress on your Web Server EC2**

`sudo yum -y update`

Install wget, Apache and it’s dependencies

`sudo yum -y install wget httpd php php-mysqlnd php-fpm php-json`

Start Apache

```
sudo systemctl enable httpd
sudo systemctl start httpd
```
To install PHP and it’s depemdencies

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
Restart Apache

`sudo systemctl restart httpd`

Download wordpress and copy wordpress to var/www/html

```
mkdir wordpress
  cd   wordpress
  sudo wget http://wordpress.org/latest.tar.gz
  sudo tar xzvf latest.tar.gz
  sudo rm -rf latest.tar.gz
  cp wordpress/wp-config-sample.php wordpress/wp-config.php
  cp -R wordpress /var/www/html/
```

Install MySQL on your DB Server EC2
```
sudo yum update
sudo yum install mysql-server
```

Check the status if the service is running. Run

`sudo systemctl status mysqld`

To start and enable mysqld, Run:

```
sudo systemctl restart mysqld
sudo systemctl enable mysqld
```

## **Configure DB to work with WordPress**

```
sudo mysql
CREATE DATABASE wordpress;
CREATE USER `myuser`@`<Web-Server-Private-IP-Address>` IDENTIFIED BY 'mypass';
GRANT ALL ON wordpress.* TO 'myuser'@'<Web-Server-Private-IP-Address>';
FLUSH PRIVILEGES;
SHOW DATABASES;
exit
```

![mysqluser](https://user-images.githubusercontent.com/77943759/224437771-27fd9525-246c-494d-9f7f-a06390c56631.png)

![Screenshot from 2023-03-10 23-10-29](https://user-images.githubusercontent.com/77943759/224437875-0854ccac-77c4-4462-9fce-2a5fefed199a.png)

## **Configure WordPress to connect to remote database**

open MySQL port 3306 on DB Server EC2, also, set the ip address to the public address of the webserver and add /32 to it

![ec2mysql](https://user-images.githubusercontent.com/77943759/224439030-db5cafe7-0d50-4bf8-8f50-aee2f04fc89a.png)

Install MySQL client and test that we can connect the database server from the webserver.

```
sudo yum install mysql
sudo mysql -u admin -p -h <DB-Server-Private-IP-address>
```
On mysql `show database` to confirm if you can see the database created on the webserver

![dbconfirm](https://user-images.githubusercontent.com/77943759/224439748-80a0fe9c-1d99-4ebf-a69d-d56d26850465.png)

Change permissions and configuration so Apache could use WordPress:

Configure SELinux Policies

```
sudo chown -R apache:apache /var/www/html/wordpress
sudo chcon -t httpd_sys_rw_content_t /var/www/html/wordpress -R
sudo setsebool -P httpd_can_network_connect=1
```
Enable TCP port 80 in Inbound Rules configuration for your Web Server EC2 from anywhere 0.0.0.0/0 or system ip


![Screenshot from 2023-03-10 23-35-26](https://user-images.githubusercontent.com/77943759/224441129-02f02fc9-0a13-48a5-9300-1e7153626e3b.png)

![Screenshot from 2023-03-10 23-36-26](https://user-images.githubusercontent.com/77943759/224441191-43db75a0-2cf7-40eb-9256-f5dc605480b8.png)

END



