# 🚀 WordPress Three-Tier Architecture Deployment on AWS EC2 (RedHat)

---

## 📝 Overview

This project demonstrates deploying a **WordPress CMS** on a **RedHat EC2 instance** connected to a remote **MySQL database** on another RedHat EC2 instance, following a classic **Three-Tier Architecture**:

- 🌐 **Presentation Layer:** Client browser accessing WordPress  
- 🏗️ **Business Layer:** WordPress running on RedHat web server EC2  
- 💾 **Data Access Layer:** MySQL running on RedHat database server EC2  

Both servers also use **Linux disk partitioning** and **Logical Volume Management (LVM)** for optimized and resilient storage.

---

## ⚙️ Prerequisites

Before you begin, make sure you have:

- An AWS account with **two RedHat EC2 instances** (Web Server + DB Server)  
- SSH access configured via `ec2-user`  
- Proper **security group rules**:  
  - Web Server: SSH (22), HTTP (80)  
  - DB Server: SSH (22), MySQL (3306) restricted to Web Server private IP  

---

## 🛠 Step 1 — Prepare the Web Server

### 📦 Attach EBS Volumes & Check Devices

```bash
lsblk
df -h
<img width="1455" height="460" alt="lsblk example" src="https://github.com/user-attachments/assets/0b08fa7e-48f2-4255-881e-63c041d78b30" /> <img width="726" height="517" alt="df example" src="https://github.com/user-attachments/assets/311e50a9-5a84-446e-80c8-4d9c2bfb3cc4" />
💽 Install and Run gdisk
Since this is RedHat 10, enabling repos is slightly different:

bash
Copy code
sudo subscription-manager repos --enable codeready-builder-for-rhel-10-$(arch)-rpms
sudo dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-10.noarch.rpm
sudo dnf install gdisk
Run gdisk:

bash
Copy code
sudo gdisk /dev/nvme1n1
Inside gdisk:

n → new partition

Accept defaults

Code: 8E00 (Linux LVM)

w → write changes

Repeat for /dev/nvme2n1 and /dev/nvme3n1.



🗄️ Setup LVM
1. Install LVM tools

bash
Copy code
sudo yum install lvm2 -y


2. Create physical volumes

bash
Copy code
sudo pvcreate /dev/nvme1n1p1 /dev/nvme2n1p1 /dev/nvme3n1p1


3. Create volume group

bash
Copy code
sudo vgcreate webdata-vg /dev/nvme1n1p1 /dev/nvme2n1p1 /dev/nvme3n1p1


4. Create logical volumes

bash
Copy code
sudo lvcreate -n apps-lv -L 14G webdata-vg
sudo lvcreate -n logs-lv -L 14G webdata-vg


5. Format logical volumes

bash
Copy code
sudo mkfs.ext4 /dev/webdata-vg/apps-lv
sudo mkfs.ext4 /dev/webdata-vg/logs-lv
6. Mount volumes

bash
Copy code
sudo mkdir -p /var/www/html
sudo mkdir -p /home/recovery/logs
sudo mount /dev/webdata-vg/apps-lv /var/www/html
7. Backup and remount logs

bash
Copy code
sudo rsync -av /var/log/ /home/recovery/logs/
sudo mount /dev/webdata-vg/logs-lv /var/log
sudo rsync -av /home/recovery/logs/ /var/log


8. Make mounts persistent

bash
Copy code
sudo blkid
sudo vi /etc/fstab
Add entries (replace UUIDs):

ini
Copy code
UUID=<UUID_apps-lv> /var/www/html ext4 defaults 0 2
UUID=<UUID_logs-lv> /var/log     ext4 defaults 0 2


9. Reload and verify

bash
Copy code
sudo mount -a
sudo systemctl daemon-reload
df -h


🛠 Step 2 — Prepare the Database Server
Repeat the same LVM setup steps as above.
Create a logical volume named db-lv and mount it at /db.

🛠 Step 3 — Install WordPress on Web Server
1. Install packages

bash
Copy code
sudo yum -y update
sudo yum -y install wget httpd php php-mysqlnd php-fpm php-json
sudo systemctl enable httpd --now
2. Enable PHP Remi repo & extensions

bash
Copy code
sudo yum install epel-release yum-utils -y
sudo yum install http://rpms.remirepo.net/enterprise/remi-release-8.rpm -y
sudo yum module reset php -y
sudo yum module enable php:remi-7.4 -y
sudo yum install php php-opcache php-gd php-curl php-mysqlnd -y
3. Configure Apache & PHP

bash
Copy code
sudo systemctl enable php-fpm --now
sudo setsebool -P httpd_execmem 1
sudo systemctl restart httpd
sudo systemctl status httpd
<img width="1306" height="432" alt="httpd running" src="https://github.com/user-attachments/assets/30b1f598-54d8-400f-8cea-03698a749bc3" />
4. Download and configure WordPress

bash
Copy code
mkdir wordpress && cd wordpress
sudo wget http://wordpress.org/latest.tar.gz
sudo tar -xzvf latest.tar.gz
sudo rm -f latest.tar.gz
sudo cp wordpress/wp-config-sample.php wordpress/wp-config.php
sudo cp -r wordpress /var/www/html/
sudo chown -R apache:apache /var/www/html/wordpress
sudo chcon -t httpd_sys_rw_content_t /var/www/html/wordpress -R
sudo setsebool -P httpd_can_network_connect=1
<img width="1381" height="569" alt="wordpress install" src="https://github.com/user-attachments/assets/3ed88df5-bdab-4563-951c-49d39f6c445c" />
🛠 Step 4 — Install MySQL on DB Server
bash
Copy code
sudo yum update -y
sudo yum install mysql-server -y
sudo systemctl enable mysqld --now
🛠 Step 5 — Configure Database for WordPress
bash
Copy code
sudo mysql
Inside MySQL:

sql
Copy code
CREATE DATABASE wordpress;
CREATE USER 'myuser'@'<web-server-private-ip>' IDENTIFIED BY 'mypass';
GRANT ALL ON wordpress.* TO 'myuser'@'<web-server-private-ip>';
FLUSH PRIVILEGES;
EXIT;
🛠 Step 6 — Connect WordPress to Remote Database
1. Open DB Port (3306) restricted to Web Server IP.

<img width="1588" height="231" alt="SG rule" src="https://github.com/user-attachments/assets/31c4e6be-0329-4cb6-abdb-e8cdafd02464" />
2. Test connection from Web Server

bash
Copy code
sudo yum install mysql -y
mysql -u myuser -p -h <db-server-private-ip>
SHOW DATABASES;
3. Ensure port 80 open on Web Server security group.

<img width="1637" height="384" alt="HTTP SG" src="https://github.com/user-attachments/assets/61b18f26-c37a-4fdc-8f3c-5a1c4c0f9a5e" />
4. Access WordPress setup

perl
Copy code
http://<web-server-public-ip>/wordpress/
<img width="1838" height="824" alt="wordpress setup" src="https://github.com/user-attachments/assets/3348932f-1d6c-4aeb-8c65-5569dbe2dec5" />
Enter DB credentials and complete installation.

5. Verify WordPress is running

<img width="1873" height="979" alt="custom site" src="https://github.com/user-attachments/assets/0b808d47-024a-439c-8a27-53b108787607" />
⚠️ Notes & Troubleshooting
💸 Stop EC2 instances when not in use to avoid costs.

🔑 Replace all placeholder IPs, UUIDs, and credentials with your own.

🔒 Restrict ports for security.

Issues faced on RedHat 10:

Some tools like gdisk and mysql were not available directly in the default repos → required enabling CRB & EPEL.

WordPress database connection failed until wp-config.php was updated with correct DB user and private IP.
