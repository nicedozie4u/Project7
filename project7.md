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

