AWS-Based Photo Album Application

A fully functional, secure, and scalable photo album web application built and deployed entirely on Amazon Web Services (AWS). This project demonstrates real-world cloud infrastructure setup using core AWS services — VPC, EC2, RDS, S3, and phpMyAdmin.

---

What This Project Does

This application lets users view a photo album through a web browser. Photos are stored in AWS S3 (cloud storage), and information about each photo (like title, description, and keywords) is stored in a MySQL database hosted on AWS RDS. The website itself runs on an Apache web server inside an EC2 virtual machine, all wrapped inside a secure private network (VPC).

Think of it like this:
- S3  = your photo storage drive in the cloud
- RDS = your database that remembers photo details
- EC2 = the computer running your website
- VPC = the secure private network connecting everything

---

Architecture Overview

```
Internet
   │
   ▼
Internet Gateway (ASrivastavaIGW)
   │
   ▼
VPC: ASrivastavaVPC (10.0.0.0/16) — Region: us-east-1
   │
   ├── Availability Zone A (us-east-1a)
   │     ├── Public Subnet 1  (10.0.1.0/24)
   │     └── Private Subnet 1 (10.0.2.0/24)  ← Test Instance
   │
   └── Availability Zone B (us-east-1b)
         ├── Public Subnet 2  (10.0.3.0/24)  ← WebServerInstance (EC2)
         └── Private Subnet 2 (10.0.4.0/24)  ← RDS MySQL Database
```

---

AWS Services Used

```
| Service | Purpose |
|--------|---------|
| VPC | Creates an isolated private network for all resources |
| EC2 | Virtual machine that runs the Apache web server + PHP app |
| RDS (MySQL) | Managed database storing photo metadata |
| S3 | Object storage for actual image files |
| phpMyAdmin | Web UI to manage the MySQL database |
| Internet Gateway | Allows public internet traffic into the VPC |
| Elastic IP | Static public IP address for the web server |
| Network ACLs | Subnet-level firewall rules |
| Security Groups | Instance-level firewall rules |
```
---

Step-by-Step Setup Guide

Step 1 — Set Up the VPC and Subnets

1. Go to AWS Console → VPC → Create VPC
2. Name it `ASrivastavaVPC`, set CIDR block to `10.0.0.0/16`
3. Create 4 subnets across 2 Availability Zones:
   - `Public Subnet 1` → `10.0.1.0/24` (us-east-1a)
   - `Private Subnet 1` → `10.0.2.0/24` (us-east-1a)
   - `Public Subnet 2` → `10.0.3.0/24` (us-east-1b)
   - `Private Subnet 2` → `10.0.4.0/24` (us-east-1b)
4. Create an Internet Gateway, name it `ASrivastavaIGW`, and attach it to the VPC
5. Create a Public Route Table, add a route `0.0.0.0/0 → IGW`, and associate it with both public subnets

> Why two availability zones? If one data center goes down, the other keeps your app running. This is called high availability.

---

Step 2 — Launch the EC2 Web Server

1. Go to EC2 → Launch Instance
2. Name it `WebServerInstance`
3. Choose Amazon Linux 2023 AMI and instance type t2.micro (Free Tier eligible)
4. Place it in Public Subnet 2 so it gets a public IP
5. Attach or create a key pair (e.g., `Assignment-1b.ppk`) for SSH access
6. Assign a Security Group (`WebServerSG`) allowing:
   - Port `22` (SSH) for admin access
   - Port `80` (HTTP) for web traffic
   - Port `443` (HTTPS) for secure web traffic
7. After launch, allocate an Elastic IP and associate it with this instance — this gives you a permanent public IP

Once the instance is running, SSH into it and install the required software:

```bash
Update the system
sudo dnf update -y

Install Apache web server
sudo dnf install httpd -y
sudo systemctl start httpd
sudo systemctl enable httpd

Install PHP
sudo dnf install php php-mysqlnd -y

Install phpMyAdmin
sudo dnf install phpMyAdmin -y
```

---

Step 3 — Launch the Test Instance (Optional but Recommended)

1. Launch a second EC2 instance named `TestInstance`
2. Place it in Private Subnet 1 (no internet access — this is intentional)
3. Use it to test internal network connectivity:

```bash
 Ping the web server's private IP from the test instance
ping 10.0.2.65

 Test SSH connectivity internally
ssh -i Assignment-1b.ppk ec2-user@10.0.2.65
```

> This simulates how backend/internal servers behave in real company architectures — they never talk directly to the internet.

---

Step 4 — Set Up the RDS MySQL Database

1. Go to RDS → Create Database
2. Choose MySQL, version `8.0.39`
3. Select Free Tier template
4. Set Public access = No — this is critical for security
5. Create a DB Subnet Group (`asrivastavadbsubnetgroup`) using Private Subnet 1 and Private Subnet 2
6. Attach Security Group `RDSSG` that only allows:
   - Port `3306` (MySQL) from `WebServerSG` only

> By setting public access to No, your database is invisible to the internet. Only your web server can talk to it. This is a standard security best practice used in every real-world application.

---

Step 5 — Configure phpMyAdmin

phpMyAdmin gives you a visual interface to manage your database without writing SQL manually.

1. Edit the phpMyAdmin config file to point to your RDS endpoint:

```bash
sudo nano /etc/phpMyAdmin/config.inc.php
```

2. Set the host to your RDS endpoint URL (found in the RDS console under Connectivity & security)
3. If you get a permissions error, fix it with:

```bash
sudo chown -R apache:apache /usr/share/phpMyAdmin
sudo chmod -R 755 /usr/share/phpMyAdmin
```

4. Access phpMyAdmin via browser: `http://<your-elastic-ip>/phpMyAdmin`

---

Step 6 — Create the S3 Bucket for Photos

1. Go to S3 → Create Bucket
2. Name it `photoalbum-images-bucket`
3. Uncheck "Block all public access" (needed so browsers can load the photos)
4. Apply this bucket policy to allow public read access:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowPublicRead",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::photoalbum-images-bucket/*"
    }
  ]
}
```

5. Upload your image files (`.jpg`, `.png`, etc.) into the bucket
6. Each image will get a public URL like:
   `https://photoalbum-images-bucket.s3.amazonaws.com/P1.jpg`

---

 Step 7 — Create the Database Table

In phpMyAdmin, run the following SQL to create the photos table:

```sql
CREATE TABLE photos (
    id           INT AUTO_INCREMENT PRIMARY KEY,
    photo_title  VARCHAR(255),
    description  VARCHAR(255),
    creation_date DATE,
    keywords     VARCHAR(255),
    s3_reference VARCHAR(255)
);
```

Insert sample data:

```sql
INSERT INTO photos (photo_title, description, creation_date, keywords, s3_reference)
VALUES
  ('Altona Beach', 'Beach sunset', '2025-04-13', 'beach, sunset', 'https://photoalbum-images-bucket.s3.us-east-1.amazonaws.com/P1.jpg'),
  ('St Kilda Beach', 'Beach sunset', '2025-04-13', 'Sunset Beach', 'https://photoalbum-images-bucket.s3.us-east-1.amazonaws.com/P2.jpg'),
  ('Swinburne', 'Swinburne College Building', '2025-04-13', 'Building, College', 'https://photoalbum-images-bucket.s3.us-east-1.amazonaws.com/P3.jpg');
```

---

Step 8 — Configure the Application

Edit the `constants.php` file in your application code to connect to RDS and S3:

```php
<?php
define('DB_HOST', 'your-rds-endpoint.rds.amazonaws.com');
define('DB_USER', 'admin');
define('DB_PASS', 'your-password');
define('DB_NAME', 'photo_album');
define('S3_BUCKET_URL', 'https://photoalbum-images-bucket.s3.amazonaws.com/');
?>
```

---

 Step 9 — Configure Network ACLs

A Network ACL (`PublicSubnet2NACL`) was added to Public Subnet 2 as an extra layer of security:

| Rule  | Type | Protocol | Port | Source | Allow/Deny |
|--------|------|----------|------|--------|------------|
| 100 | SSH | TCP | 22 | 0.0.0.0/0 | Allow |
| 110 | ICMP | All ICMP | All | 10.0.2.0/24 | Allow |
| 120 | HTTP | TCP | 80 | 0.0.0.0/0 | Allow |
| 130 | HTTPS | TCP | 443 | 0.0.0.0/0 | Allow |
| * | All traffic | All | All | 0.0.0.0/0 | Deny |

> 💡 Unlike Security Groups (which remember connections), NACLs are stateless — you must explicitly allow both inbound AND outbound traffic for every rule.

---

Accessing the Application

Once everything is set up, open a browser and go to:

```
http://18.233.133.57/cos80001/photoalbum/album.php
```

You should see the photo album with images loaded from S3 and details pulled from the RDS database.

---

Security Design Summary

This project follows a defence-in-depth approach — multiple layers of security:

1. VPC Isolation — all resources live inside a private network, not exposed directly to the internet
2. Public vs Private Subnets — web server is public-facing; database is locked in a private subnet
3. Security Groups — instance-level rules; RDS only accepts connections from the web server
4. Network ACLs — subnet-level rules for an extra firewall layer
5. No Public RDS Access — database has no public IP at all
6. S3 Bucket Policy — only read access is granted publicly; no one can delete or modify files

---

Project Structure

```
photo-album-aws/
│
├── constants.php           DB and S3 connection settings
├── album.php               Main webpage displaying photos
├── /phpMyAdmin             Installed on EC2 for DB management
│
└── AWS Infrastructure
    ├── VPC (ASrivastavaVPC)
    ├── EC2 (WebServerInstance + TestInstance)
    ├── RDS (photoalbum-db, MySQL 8.0.39)
    └── S3  (photoalbum-images-bucket)
```

---

Tech Stack

- Cloud Provider: Amazon Web Services (AWS)
- Web Server: Apache HTTP Server
- Backend Language: PHP
- Database: MySQL 8.0.39 (via AWS RDS)
- Object Storage: AWS S3
- DB Management UI: phpMyAdmin
- OS: Amazon Linux 2023
- Instance Type: t2.micro (Free Tier)

