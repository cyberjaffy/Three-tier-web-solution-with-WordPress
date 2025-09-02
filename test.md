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

<img width="1455" height="460" alt="EBS Volume Attachment" src="https://github.com/user-attachments/assets/0b08fa7e-48f2-4255-881e-63c041d78b30" />

Identify block devices attached:

lsblk
df -h

text

<img width="726" height="517" alt="Block Devices and Disk Usage" src="https://github.com/user-attachments/assets/311e50a9-5a84-446e-80c8-4d9c2bfb3cc4" />

Create partitions on each EBS volume using `gdisk`:

Install gdisk on RedHat 10:

sudo subscription-manager repos --enable codeready-builder-for-rhel-10-$(arch)-rpms
sudo dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-10.noarch.rpm
sudo dnf install gdisk

text

Partition disks:

sudo gdisk /dev/nvme1n1

text

Commands inside gdisk:

- `n` (new partition)
- Accept defaults
- Code: `8E00` (Linux LVM)
- `w` (write changes)

Repeat for `/dev/nvme2n1` and `/dev/nvme3n1`

![Gdisk Partitioning](https://github.com/user-attachments/assets/98cc4bc1-219f-4811-a44b-4dd1c41144f7)

---

### 💿 Setup LVM

Install LVM tools:

sudo yum install lvm2 -y

text

![LVM Installation](https://github.com/user-attachments/assets/44989d43-d8fe-40e6-a508-9c17f82d6066)

Create physical volumes:

sudo pvcreate /dev/nvme1n1p1 /dev/nvme2n1p1 /dev/nvme3n1p1

text

![Physical Volumes Creation](https://github.com/user-attachments/assets/a697d108-a073-4c4b-93d8-3b4e6bafcd3d)

Create volume group:

sudo vgcreate webdata-vg /dev/nvme1n1p1 /dev/nvme2n1p1 /dev/nvme3n1p1

text

![Volume Group Creation](https://github.com/user-attachments/assets/96d814bd-e0bf-4455-b8da-2e54358f6b13)

Create logical volumes for apps and logs:

sudo lvcreate -n apps-lv -L 14G webdata-vg
sudo lvcreate -n logs-lv -L 14G webdata-vg

text

![Logical Volumes Creation](https://github.com/user-attachments/assets/aeb400df-be16-4fef-a025-0e103b4f24b6)

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

![Logs Backup and Mount](https://github.com/user-attachments/assets/ec23170e-1b02-4370-a343-f094dbef6734)

Make mounts persistent via `/etc/fstab`:

sudo blkid
sudo vi /etc/fstab

text

Add lines (with your UUIDs):

UUID=<UUID_apps-lv> /var/www/html ext4 defaults 0 2
UUID=<UUID_logs-lv> /var/log ext4 defaults 0 2

text

![fstab editing](https://github.com/user-attachments/assets/008b57da-d001-4f6e-b385-11299486de30)

Test mounts and reload daemon:

sudo mount -a
sudo systemctl daemon-reload
df -h

text

![Mount Test](https://github.com/user-attachments/assets/58eed98f-5d34-4614-9413-cae20f4fb5d0)

---

## 🛠 Step 2 — Prepare the Database Server

Repeat the above steps on the DB server, but create a logical volume named `db-lv` mounted at `/db`.

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
sudo systemctl status httpd

text

<img width="1306" height="432" alt="Apache Status" src="https://github.com/user-attachments/assets/30b1f598-54d8-400f-8cea-03698a749bc3" />

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

<img width="1381" height="569" alt="WordPress Install" src="https://github.com/user-attachments/assets/3ed88df5-bdab-4563-951c-49d39f6c445c" />

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

<img width="1588" height="231" alt="Security Group MySQL Inbound Rule" src="https://github.com/user-attachments/assets/31c4e6be-0329-4cb6-abdb-e8cdafd02464" />

2. On web server, install MySQL client and test connection:

sudo yum install mysql -y
mysql -u myuser -p -h <db-server-private-ip>
SHOW DATABASES;

text

3. Ensure HTTP (`80`) port open on web server security group.

<img width="1637" height="384" alt="Security Group HTTP inbound rule" src="https://github.com/user-attachments/assets/61b18f26-c37a-4fdc-8f3c-5a1c4c0f9a5e" />

4. Access WordPress setup:

http://<web-server-public-ip>/wordpress/

text

<img width="1838" height="824" alt="WordPress Setup Screen" src="https://github.com/user-attachments/assets/3348932f-1d6c-4aeb-8c65-5569dbe2dec5" />

Enter DB credentials and complete installation. Customize your WordPress site to demonstrate its functionality.

<img width="1873" height="979" alt="Custom WordPress Site" src="https://github.com/user-attachments/assets/0b808d47-024a-439c-8a27-53b108787607" />

---

## ⚠️ Important Notes

- Stop EC2 instances after use to avoid extra costs.
- Replace placeholder IP addresses and UUIDs with your actual values.
- Follow security best practices and restrict access in your security groups.
- Troubleshooting Notes:
  - RedHat Linux 10 required enabling repositories manually to install some packages like `gdisk` and `mysql`.
  - Database connection issues were resolved by properly configuring WordPress’s `wp-config.php` with database user and IP settings.

---
