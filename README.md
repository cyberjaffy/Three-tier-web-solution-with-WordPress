text
# 🚀 WordPress Three-Tier Architecture Deployment on AWS EC2 (RedHat)

---

## 📝 Overview

This project demonstrates deploying a **WordPress CMS** on a **RedHat EC2 instance** connected to a remote **MySQL database** on a separate RedHat EC2 instance, implementing a classic **Three-Tier Architecture**:

- 🌐 **Presentation Layer:** Client browser accessing WordPress
- 🏗️ **Business Layer:** WordPress running on RedHat web server EC2
- 💾 **Data Access Layer:** MySQL running on RedHat database server EC2

Additionally, the project applies **Linux disk partitioning** and **Logical Volume Management (LVM)** for optimized and resilient storage on both servers.

---

## ⚙️ Prerequisites

Before you begin, ensure the following:

- AWS account with two RedHat EC2 instances running
- SSH access configured via `ec2-user`
- Security groups set to allow:
  - SSH (port 22)
  - HTTP (port 80) on web server
  - MySQL (port 3306) on DB server limited to web server IP

---

## 🛠 Step 1 — Prepare the Web Server

### 📦 Attach EBS Volumes & Partitioning

Identify block devices attached:

lsblk
df -h

text

Create partitions on each EBS volume using `gdisk`:

sudo gdisk /dev/xvdf

Commands inside gdisk:
- n (new partition)
- Accept defaults
- Code: 8E00 (Linux LVM)
- w (write changes)
Repeat for /dev/xvdg and /dev/xvdh
text

### 💿 Setup LVM

Install LVM tools:

sudo yum install lvm2 -y

text

Create physical volumes:

sudo pvcreate /dev/xvdf1 /dev/xvdg1 /dev/xvdh1

text

Create a volume group:

sudo vgcreate webdata-vg /dev/xvdf1 /dev/xvdg1 /dev/xvdh1

text

Create logical volumes for apps and logs:

sudo lvcreate -n apps-lv -L 14G webdata-vg
sudo lvcreate -n logs-lv -L 14G webdata-vg

text

Format logical volumes with ext4:

sudo mkfs.ext4 /dev/webdata-vg/apps-lv
sudo mkfs.ext4 /dev/webdata-vg/logs-lv

text

Create mount points and mount volumes:

sudo mkdir -p /var/www/html
sudo mkdir -p /home/recovery/logs
sudo mount /dev/webdata-vg/apps-lv /var/www/html

text

Backup existing logs and mount logs volume:

sudo rsync -av /var/log/ /home/recovery/logs/
sudo mount /dev/webdata-vg/logs-lv /var/log
sudo rsync -av /home/recovery/logs/ /var/log

text

Make mounts persistent via `/etc/fstab`:

sudo blkid
sudo vi /etc/fstab

Add lines (replace UUIDs with your device UUIDs):
UUID=<UUID_apps-lv> /var/www/html ext4 defaults 0 2
UUID=<UUID_logs-lv> /var/log ext4 defaults 0 2
text

Test mounts and reload daemon:

sudo mount -a
sudo systemctl daemon-reload
df -h

text

---

![Partitioning](./screenshots/partitioning.png)  
*Example of gdisk partitioning*

![LVM Setup](./screenshots/lvm_setup.png)  
*Logical Volume creation*

---

## 🛠 Step 2 — Prepare the Database Server

Repeat the above steps to configure LVM storage on the DB server. Create a logical volume `db-lv` mounted at `/db`.

---

## 🛠 Step 3 — Install WordPress on Web Server

Update system and install required packages:

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

text

Download WordPress and configure permissions:

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

---

![WordPress Install](./screenshots/wordpress_installation.png)  
*WordPress files ready in web directory*

---

## 🛠 Step 4 — Install MySQL on DB Server

sudo yum update -y
sudo yum install mysql-server -y
sudo systemctl enable mysqld --now

text

---

## 🛠 Step 5 — Configure Database for WordPress

sudo mysql
CREATE DATABASE wordpress;
CREATE USER 'myuser'@'<web-server-private-ip>' IDENTIFIED BY 'mypass';
GRANT ALL ON wordpress.* TO 'myuser'@'<web-server-private-ip>';
FLUSH PRIVILEGES;
EXIT;

text

---

## 🛠 Step 6 — Connect WordPress to Remote Database

1. Open port `3306` restricted to web server IP on DB server’s security group.
2. On web server, install MySQL client and test connection:

sudo yum install mysql -y
mysql -u myuser -p -h <db-server-private-ip>
SHOW DATABASES;

text

3. Ensure HTTP (`80`) port open on web server.
4. Access WordPress setup:

http://<web-server-public-ip>/wordpress/

text

Enter DB credentials and complete installation.

---

## ⚠️ Important Notes

- Stop EC2 instances after to avoid costs!
- Replace all `<placeholders>` with your actual IPs/UUIDs.
- Security best practices: restrict access where possible.

---
