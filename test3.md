# 🚀 WordPress Three-Tier Architecture Deployment on AWS EC2 (RedHat)

---

## 📝 Overview

This project demonstrates deploying **WordPress CMS** on a **RedHat EC2 instance** connected to a remote **MySQL database** on a separate RedHat EC2 instance, implementing a classic **Three-Tier Architecture**:

1. 🌐 **Presentation Layer:** Client browser accessing WordPress  
2. 🏗️ **Business Layer:** WordPress running on RedHat web server EC2  
3. 💾 **Data Access Layer:** MySQL running on RedHat database server EC2  

Additionally, the project applies **Linux disk partitioning** and **Logical Volume Management (LVM)** for optimized and resilient storage on both servers.

---

## ⚙️ Prerequisites

Before you begin, ensure the following:

- Two RedHat EC2 instances on AWS  
- SSH access setup via `ec2-user`  
- Security group rules allowing access on:  
  - SSH (port 22)  
  - HTTP (port 80) for web server  
  - MySQL (port 3306) for DB server, restricted to web server IP  

---

## 🛠 Step 1 — Prepare the Web Server

1. **Launch and attach storage volumes**

   Create three 10 GiB EBS volumes and attach to the Web Server EC2.

   ![EBS Volumes](https://github.com/user-attachments/assets/0b08fa7e-48f2-4255-881e-63c041d78b30)

2. **SSH into the Web Server**

ssh -i <key-pair.pem> ec2-user@<web-server-public-ip>

text

3. **Verify attached volumes**

lsblk
df -h

text

![Block Devices](https://github.com/user-attachments/assets/311e50a9-5a84-446e-80c8-4d9c2bfb3cc4)

4. **Install and use gdisk to partition volumes**

sudo subscription-manager repos --enable codeready-builder-for-rhel-10-$(arch)-rpms
sudo dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-10.noarch.rpm
sudo dnf install gdisk
sudo gdisk /dev/nvme1n1

text

Within gdisk:

n
[Enter]
[Enter]
8300
w

text

Repeat for `/dev/nvme2n1` and `/dev/nvme3n1`.

![gdisk partitioning](https://github.com/user-attachments/assets/98cc4bc1-219f-4811-a44b-4dd1c41144f7)

5. **Install LVM and create volume group**

sudo yum install lvm2 -y
sudo pvcreate /dev/nvme1n1p1 /dev/nvme2n1p1 /dev/nvme3n1p1
sudo vgcreate webdata-vg /dev/nvme1n1p1 /dev/nvme2n1p1 /dev/nvme3n1p1

text

![LVM setup](https://github.com/user-attachments/assets/96d814bd-e0bf-4455-b8da-2e54358f6b13)

6. **Create logical volumes**

sudo lvcreate -n apps-lv -L 14G webdata-vg
sudo lvcreate -n logs-lv -L 14G webdata-vg

text

![Logical Volumes](https://github.com/user-attachments/assets/aeb400df-be16-4fef-a025-0e103b4f24b6)

7. **Format and mount**

sudo mkfs.ext4 /dev/webdata-vg/apps-lv
sudo mkfs.ext4 /dev/webdata-vg/logs-lv

sudo mkdir -p /var/www/html
sudo mkdir -p /home/recovery/logs
sudo mount /dev/webdata-vg/apps-lv /var/www/html

text

8. **Backup logs and mount logs volume**

sudo rsync -av /var/log/ /home/recovery/logs/
sudo mount /dev/webdata-vg/logs-lv /var/log
sudo rsync -av /home/recovery/logs/ /var/log

text

![Logs backup mount](https://github.com/user-attachments/assets/ec23170e-1b02-4370-a343-f094dbef6734)

9. **Update /etc/fstab**

sudo blkid
sudo vi /etc/fstab

text

Add:

UUID=<UUID_apps-lv> /var/www/html ext4 defaults 0 2
UUID=<UUID_logs-lv> /var/log ext4 defaults 0 2

text

![fstab file](https://github.com/user-attachments/assets/008b57da-d001-4f6e-b385-11299486de30)

10. **Verify mounts**

 ```
 sudo mount -a
 sudo systemctl daemon-reload
 df -h
 ```

 ![df-h output](https://github.com/user-attachments/assets/58eed98f-5d34-4614-9413-cae20f4fb5d0)

---

## 🛠 Step 2 — Prepare the Database Server

Repeat Step 1 for DB server, creating one logical volume `db-lv` mounted at `/db`.

---

## 🛠 Step 3 — Install WordPress on Web Server

1. **Update and install packages**

sudo yum -y update
sudo yum -y install wget httpd php php-mysqlnd php-fpm php-json
sudo systemctl enable httpd --now
sudo yum install epel-release yum-utils -y
sudo yum install http://rpms.remirepo.net/enterprise/remi-release-8.rpm -y
sudo yum module reset php -y
sudo yum module enable php:remi-7.4 -y
sudo yum install php php-opcache php-gd php-curl php-mysqlnd -y
sudo systemctl enable php-fpm --now
sudo setsebool -P httpd_execmem 1
sudo systemctl restart httpd
sudo systemctl status httpd

text

![Apache status](https://github.com/user-attachments/assets/30b1f598-54d8-400f-8cea-03698a749bc3)

2. **Download and prepare WordPress**

mkdir wordpress && cd wordpress
sudo wget http://wordpress.org/latest.tar.gz
sudo tar -xzvf latest.tar.gz
sudo rm -f latest.tar.gz
sudo cp wordpress/wp-config-sample.php wordpress/wp-config.php
sudo cp -r wordpress /var/www/html/
sudo chown -R apache:apache /var/www/html/wordpress
sudo chcon -t httpd_sys_rw_content_t /var/www/html/wordpress -R
sudo setsebool -P httpd_can_network_connect=1

text

![WordPress files](https://github.com/user-attachments/assets/3ed88df5-bdab-4563-951c-49d39f6c445c)

---

## 🛠 Step 4 — Install MySQL Server on DB

sudo yum update -y
sudo yum install mysql-server -y
sudo systemctl enable mysqld --now

text

---

## 🛠 Step 5 — Configure MySQL for WordPress

sudo mysql
CREATE DATABASE wordpress;
CREATE USER 'myuser'@'<web-server-private-ip>' IDENTIFIED BY 'mypass';
GRANT ALL ON wordpress.* TO 'myuser'@'<web-server-private-ip>';
FLUSH PRIVILEGES;
EXIT;

text

---

## 🛠 Step 6 — Connect WordPress to Remote Database

1. Open inbound port 3306 in DB server security group, restricting source to web server IP.

   ![MySQL Security Rule](https://github.com/user-attachments/assets/31c4e6be-0329-4cb6-abdb-e8cdafd02464)

2. On Web Server, install MySQL client and test DB connection:

sudo yum install mysql -y
mysql -u myuser -p -h <db-server-private-ip>
SHOW DATABASES;

text

3. Open HTTP (80) port on Web Server security group.

![HTTP Security Rule](https://github.com/user-attachments/assets/61b18f26-c37a-4fdc-8f3c-5a1c4c0f9a5e)

4. Access WordPress setup in browser:

http://<web-server-public-ip>/wordpress/

text

![WordPress Setup](https://github.com/user-attachments/assets/3348932f-1d6c-4aeb-8c65-5569dbe2dec5)

5. Complete installation and setup a customized WordPress site.

![WordPress Site](https://github.com/user-attachments/assets/0b808d47-024a-439c-8a27-53b108787607)

---

## ⚠️ Important Notes

- Stop EC2 instances after use to avoid extra charges.  
- Replace placeholders with actual IPs and UUIDs.  
- Restrict permission access carefully.  
- Troubleshooting notes: RedHat 10 repos may require enabling for some packages. Correct `wp-config.php` setup needed for DB connectivity.

---
