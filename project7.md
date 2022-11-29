## Setup and technologies used in Project 7

As a member of a DevOps team, you will implement a tooling website solution which makes access to DevOps tools within the corporate infrastructure easily accessible.

In this project you will implement a solution that consists of following components:

1. Infrastructure: AWS
2. Webserver Linux: Red Hat Enterprise Linux 8
3. Database Server: Ubuntu 20.04 + MySQL
4. Storage Server: Red Hat Enterprise Linux 8 + NFS Server
5. Programming Language: PHP
6. Code Repository: GitHub

![architecture](./images/architecture.PNG)

It is important to know what storage solution is suitable for what use cases, for this – you need to answer following questions: what data will be stored, in what format, how this data will be accessed, by whom, from where, how frequently, etc. Base on this you will be able to choose the right storage system for your solution.

## PREPARE NFS SERVER

Spin up a new EC2 instance with RHEL Linux 8 Operating System.

![NFS](./images/NFSserver.PNG)

Based on your LVM experience from Project 6, Configure LVM on the Server.

![LVM](./images/LVM.PNG)

Attach all three volumes one by one to your Web Server EC2 instance

Open up the Linux terminal to begin configuration

Use `lsblk` command to inspect what block devices are attached to the server.

`lsblk`

![LVM](./images/lsblk.PNG)

Use `df -h` command to see all mounts and free space on your server

`df -h`

![df -h](./images/df%20-h.PNG)

Use `gdisk` utility to create a single partition on each of the 3 disks

`sudo gdisk /dev/xvdf`

![gdisk](./images/createLVM001.PNG)

![gdisk](./images/createLVM002.PNG)

![gdisk](./images/createLVM003.PNG)

Use `lsblk` utility to view the newly configured partition on each of the 3 disks

`lsblk`

![lsblk2](./images/lsblk2.PNG)

Install `lvm2` package using `sudo yum install lvm2`

`sudo yum install lvm2`

![lvm2](./images/install%20lvm2.PNG)

Run `sudo lvmdiskscan` command to check for available partitions

`sudo lvmdiskscan`

![lvmdisk](./images/lvmdiskscan.PNG)

**Note**: Previously, in Ubuntu we used apt command to install packages, in RedHat/CentOS a different package manager is used, so we shall use yum command instead.

Use `pvcreate` utility to mark each of 3 disks as physical volumes (PVs) to be used by LVM

```
sudo pvcreate /dev/xvdf1
sudo pvcreate /dev/xvdg1
sudo pvcreate /dev/xvdh1
```

![pvcreate](./images/pvcreate.PNG)

Verify that your Physical volume has been created successfully by running `sudo pvs`

`sudo pvs`

![pvs](./images/pvs.PNG)

Use vgcreate utility to add all 3 PVs to a volume group (VG). Name the VG **nfs-vg**

`sudo vgcreate nfs-vg /dev/xvdh1 /dev/xvdg1 /dev/xvdf1`

![vgcreate](./images/vgcreate%20nfs-vg.PNG)

Verify that your VG has been created successfully by running `sudo vgs`

`sudo vgs`

![vgs](./images/vgs.PNG)

Use lvcreate utility to create 3 logical volumes. opt-lv, apps-lv, and logs-lv. NOTE: apps-lv will be used to store data for the Website while, logs-lv will be used to store data for logs, and opt-lv will be used by Jenkins server in Project 8

```
sudo lvcreate -L 10G -n lv-apps nfs-vg 
sudo lvcreate -L 10G -n lv-opt nfs-vg
sudo lvcreate -L 5G -n lv-logs nfs-vg
```

![lvcreate group](./images/lvcreate.PNG)

Verify that your Logical Volume has been created successfully by running `sudo lvs`

`sudo lvs`

![lvs](./images/lvs.PNG)

Verify the entire setup

`sudo vgdisplay -v #view complete setup - VG, PV, and LV`

![lvs](./images/vgdisplay.PNG)

`sudo lsblk`

![lvs](./images/lsblk3.PNG)


Instead of formating the disks as `ext4` you will have to format them as `xfs`

Use `mkfs.ext4` to format the logical volumes with `xfs` filesystem

`sudo mkfs -t xfs /dev/nfs-vg/lv-opt`

![mkfs](./images/mkfs.xfs1.PNG)

`sudo mkfs -t xfs /dev/nfs-vg/lv-apps`

![mkfs](./images/mkfs.xfs2.PNG)

`sudo mkfs -t xfs /dev/nfs-vg/lv-logs`

![mkfs](./images/mkfs.xfs3.PNG)

Ensure there are 3 Logical Volumes. `lv-opt` `lv-apps`, and `lv-logs`

Create mount points on `/mnt` directory for the logical volumes as follow:

Mount `lv-apps` on `/mnt/apps` – To be used by webservers

Mount `lv-logs` on `/mnt/logs` – To be used by webserver logs

Mount `lv-opt` on `/mnt/opt` – To be used by Jenkins server in Project 8

Create **/mnt/apps**, **/mnt/logs**, **/mnt/opt** directory to store files

`sudo mkdir -p /mnt/apps`

`sudo mkdir -p /mnt/logs`

`sudo mkdir -p /mnt/opt`

![mk mnt points](./images/mkdir.PNG)

`sudo mount /dev/nfs-vg/lv-apps /mnt/apps`

`sudo mount /dev/nfs-vg/lv-logs /mnt/logs`

`sudo mount /dev/nfs-vg/lv-opt /mnt/opt`

![mnt LV](./images/mnt%20lv.PNG)

Install NFS server, configure it to start on reboot and make sure it is up and running

```
sudo yum -y update
sudo yum install nfs-utils -y
sudo systemctl start nfs-server.service
sudo systemctl enable nfs-server.service
sudo systemctl status nfs-server.service
```
![update](./images/yum%20update.PNG)

![install NFS](./images/install%20nfs%20util.PNG)

![enable NFS](./images/start%20%26%20enable%20service.PNG)

![NFS running](./images/status%20service.PNG)

Export the mounts for webservers’ subnet cidr to connect as clients. For simplicity, you will install your all three Web Servers inside the same subnet, but in production set up you would probably want to separate each tier inside its own subnet for higher level of security.
To check your subnet cidr – open your EC2 details in AWS web console and locate ‘Networking’ tab and open a Subnet link:

Make sure we set up permission that will allow our Web servers to read, write and execute files on NFS:

```
sudo chown -R nobody: /mnt/apps
sudo chown -R nobody: /mnt/logs
sudo chown -R nobody: /mnt/opt

sudo chmod -R 777 /mnt/apps
sudo chmod -R 777 /mnt/logs
sudo chmod -R 777 /mnt/opt

sudo systemctl restart nfs-server.service
```

![update permissions](./images/update%20permissions.PNG)

Configure access to NFS for clients within the same subnet (example of Subnet CIDR – 172.31.32.0/20 ):

```
sudo vi /etc/exports

/mnt/apps <Subnet-CIDR>(rw,sync,no_all_squash,no_root_squash)
/mnt/logs <Subnet-CIDR>(rw,sync,no_all_squash,no_root_squash)
/mnt/opt <Subnet-CIDR>(rw,sync,no_all_squash,no_root_squash)

Esc + :wq!

sudo exportfs -arv
```

![edit etc/exports](./images/edit%20etcexports.PNG)

![edit etc/exports2](./images/edit%20etcexports2.PNG)

Check which port is used by NFS and open it using Security Groups (add new Inbound Rule)

`rpcinfo -p | grep nfs`

![check NFS ports](./images/check%20NFS%20port.PNG)

**Important note**: In order for NFS server to be accessible from your client, you must also open following ports: TCP 111, UDP 111, UDP 2049

![open NFS ports](./images/Open%20NFS%20ports.PNG)


## CONFIGURE THE DATABASE SERVER

By now you should know how to install and configure a MySQL DBMS to work with remote Web Server

Spin up a new EC2 instance with Ubuntu 20.04 Operating System.

![DB](./images/db%20ec2.PNG)

Install MySQL server

```
sudo yum update
sudo yum install mysql-server
```

![install mysql](./images/install%20MySql%20server.PNG)

Create a database and name it *tooling*

![create database](./images/login%20%26%20create%20database.PNG)

Create a database user and name it *webaccess*

Grant permission to *webaccess* user on *tooling* database to do anything only from the webservers *subnet cidr*

![grant priviledges](./images/grant%20privileges.PNG)


## Prepare the Web Servers

We need to make sure that our Web Servers can serve the same content from shared storage solutions, in our case – NFS Server and MySQL database.
You already know that one DB can be accessed for *reads* and *writes* by multiple clients. For storing shared files that our Web Servers will use – we will utilize NFS and mount previously created Logical Volume *lv-apps* to the folder where Apache stores files to be served to the users (*/var/www*).

This approach will make our Web Servers *stateless*, which means we will be able to add new ones or remove them whenever we need, and the integrity of the data (in the database and on NFS) will be preserved.

During the next steps we will do following:

1. Configure NFS client (this step must be done on all three servers)
2. Deploy a Tooling application to our Web Servers into a shared NFS folder
3. Configure the Web Servers to work with a single MySQL database

Launch a new EC2 instance with RHEL 8 Operating System

![Web Servers](./images/repeat%20process.PNG)

Install NFS client

`sudo yum install nfs-utils nfs4-acl-tools -y`

![insall NFS client](./images/install%20NFS%20client.PNG)

Mount */var/www/* and target the NFS server’s export for apps

```
sudo mkdir /var/www
sudo mount -t nfs -o rw,nosuid <NFS-Server-Private-IP-Address>:/mnt/apps /var/www
```

![create var/www](./images/create%20var_www.PNG)

Verify that NFS was mounted successfully by running *df -h*. Make sure that the changes will persist on Web Server after reboot:

`sudo vi /etc/fstab`

add following line

`<NFS-Server-Private-IP-Address>:/mnt/apps /var/www nfs defaults 0 0`

![check changes](./images/makesure%20chnges%20parmanent%20on%20NFS%20server.PNG)

Install *Remi’s repository*, Apache and PHP

```s
sudo yum install httpd -y

sudo dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm

sudo dnf install dnf-utils http://rpms.remirepo.net/enterprise/remi-release-8.rpm

sudo dnf module reset php

sudo dnf module enable php:remi-7.4

sudo dnf install php php-opcache php-gd php-curl php-mysqlnd

sudo systemctl start php-fpm

sudo systemctl enable php-fpm

setsebool -P httpd_execmem 1
```

![check changes](./images/install%20dnf.PNG)

![check changes](./images/install%20dnf%20util.PNG)

![check changes](./images/install%20dnf%20module.PNG)

![check changes](./images/install%20dnf%20module%20enable.PNG)

![check changes](./images/dnf%20install%20php.PNG)

![enable php](./images/start%20%26%20enable%20php.PNG)

## Repeat steps 1-5 for another 2 Web Servers

Verify that Apache files and directories are available on the Web Server in */var/www* and also on the NFS server in */mnt/apps*. If you see the same files – it means NFS is mounted correctly. You can try to create a new file *touch test.txt* from one server and check if the same file is accessible from other Web Servers.

![verify](./images/verify%20NFS%20mount.PNG)

![create test](./images/create%20test%20file.PNG)


![verify test](./images/verify%20test%20file.PNG)

![verify test](./images/verify%20test%20file02.PNG)

Locate the log folder for Apache on the Web Server and mount it to NFS server’s export for logs. Repeat step №4 to make sure the mount point will persist after reboot.

![verify](./images/verify.png)

`sudo vi /etc/fstab`

add following line(for all web servers)

`<NFS-Server-Private-IP-Address>:/mnt/apps /var/www nfs defaults 0 0`

Fork the tooling source code from *Darey.io Github Account* to your Github account. 

![install git](./images/install%20git.PNG)

Deploy the tooling website’s code to the Webserver. Ensure that the html folder from the repository is deployed to */var/www/html*

![git init](./images/webserver2.PNG)

**Note 1**: Do not forget to open TCP port 80 on the Web Server.

**Note 2**: If you encounter 403 Error – check permissions to your */var/www/html* folder and also disable SELinux `sudo setenforce 0`
To make this change permanent – open following config file `sudo vi /etc/sysconfig/selinux` and set `SELINUX=disabled` then restart httpd.

![check permissions](./images/webserver2.0.PNG)

Update the website’s configuration to connect to the database (in `/var/www/html/functions.php file`). Apply `tooling-db.sql` script to your database using this command `mysql -h <databse-private-ip> -u <db-username> -p <db-pasword> < tooling-db.sql`

![update wesite config](./images/update%20website%20config.PNG)

Create in MySQL a new admin user with username: *myuser* and password: *password*

```sql
INSERT INTO ‘users’ (‘id’, ‘username’, ‘password’, ’email’, ‘user_type’, ‘status’) VALUES
-> (1, ‘myuser’, ‘5f4dcc3b5aa765d61d8327deb882cf99’, ‘user@mail.com’, ‘admin’, ‘1’);
```

![create user](./images/create%20user02.png)

Open the website in your browser `http://<Web-Server-Public-IP-Address-or-Public-DNS-Name>/index.php` and make sure you can login into the websute with `myuser` user.

![login](./images/login%20page.PNG)


![login](./images/logged%20in.PNG)

# Congratulations.
