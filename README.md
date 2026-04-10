AWS-Based Photo Album Application

I built a fully functional, secure, and scalable photo album web application and deployed it entirely on Amazon Web Services (AWS). This project demonstrates my real-world cloud infrastructure setup using core AWS services — VPC, EC2, RDS, S3, and phpMyAdmin.

---

What This Project Does

I built this application to let users view a photo album through a web browser. I stored the photos in AWS S3 (cloud storage), and saved information about each photo (like title, description, and keywords) in a MySQL database hosted on AWS RDS. The website runs on an Apache web server inside an EC2 virtual machine, all wrapped inside a secure private network (VPC) that I configured.

Here is how I mapped each service:
- S3  = my photo storage drive in the cloud
- RDS = my database that remembers photo details
- EC2 = the computer running my website
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

1. I went to AWS Console → VPC → Create VPC
2. I named it `ASrivastavaVPC` and set the CIDR block to `10.0.0.0/16`
3. I created 4 subnets across 2 Availability Zones:
   - `Public Subnet 1` → `10.0.1.0/24` (us-east-1a)
   - `Private Subnet 1` → `10.0.2.0/24` (us-east-1a)
   - `Public Subnet 2` → `10.0.3.0/24` (us-east-1b)
   - `Private Subnet 2` → `10.0.4.0/24` (us-east-1b)
4. I created an Internet Gateway, named it `ASrivastavaIGW`, and attached it to the VPC
5. I created a Public Route Table, added a route `0.0.0.0/0 → IGW`, and associated it with both public subnets

> Why two availability zones? If one data center goes down, the other keeps the app running. This is called high availability.

---

Step 2 — Launch the EC2 Web Server

1. I went to EC2 → Launch Instance
2. I named it `WebServerInstance`
3. I chose Amazon Linux 2023 AMI and instance type t2.micro (Free Tier eligible)
4. I placed it in Public Subnet 2 so it gets a public IP
5. I attached a key pair (e.g., `Assignment-1b.ppk`) for SSH access
6. I assigned a Security Group (`WebServerSG`) allowing:
   - Port `22` (SSH) for admin access
   - Port `80` (HTTP) for web traffic
   - Port `443` (HTTPS) for secure web traffic
7. After launch, I allocated an Elastic IP and associated it with this instance — this gives a permanent public IP

Once the instance was running, I SSH'd into it and installed the required software:

```bash
# Update the system
sudo dnf update -y

# Install Apache web server
sudo dnf install httpd -y
sudo systemctl start httpd
sudo systemctl enable httpd

# Install PHP
sudo dnf install php php-mysqlnd -y

# Install phpMyAdmin
sudo dnf install phpMyAdmin -y
```

---

Step 3 — Launch the Test Instance (Optional but Recommended)

1. I launched a second EC2 instance named `TestInstance`
2. I placed it in Private Subnet 1 (no internet access — this is intentional)
3. I used it to test internal network connectivity:

```bash
# Ping the web server's private IP from the test instance
ping 10.0.2.65

# Test SSH connectivity internally
ssh -i Assignment-1b.ppk ec2-user@10.0.2.65
```

> This simulates how backend/internal servers behave in real company architectures — they never talk directly to the internet.

---

Step 4 — Set Up the RDS MySQL Database

1. I went to RDS → Create Database
2. I chose MySQL, version `8.0.39`
3. I selected the Free Tier template
4. I set Public access = No — this is critical for security
5. I created a DB Subnet Group (`asrivastavadbsubnetgroup`) using Private Subnet 1 and Private Subnet 2
6. I attached Security Group `RDSSG` that only allows:
   - Port `3306` (MySQL) from `WebServerSG` only

> By setting public access to No, my database is invisible to the internet. Only my web server can talk to it. This is a standard security best practice used in every real-world application.

---

Step 5 — Configure phpMyAdmin

phpMyAdmin gives a visual interface to manage the database without writing SQL manually.

1. I edited the phpMyAdmin config file to point to my RDS endpoint:

```bash
sudo nano /etc/phpMyAdmin/config.inc.php
```

2. I set the host to my RDS endpoint URL (found in the RDS console under Connectivity & security)
3. I ran into a permissions error and fixed it with:

```bash
sudo chown -R apache:apache /usr/share/phpMyAdmin
sudo chmod -R 755 /usr/share/phpMyAdmin
```

4. I accessed phpMyAdmin via browser: `http://<my-elastic-ip>/phpMyAdmin`

---

Step 6 — Create the S3 Bucket for Photos

1. I went to S3 → Create Bucket
2. I named it `photoalbum-images-bucket`
3. I unchecked "Block all public access" (needed so browsers can load the photos)
4. I applied this bucket policy to allow public read access:

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

5. I uploaded my image files (`.jpg`, `.png`, etc.) into the bucket
6. Each image got a public URL like:
   `https://photoalbum-images-bucket.s3.amazonaws.com/P1.jpg`

---

Step 7 — Create the Database Table

In phpMyAdmin, I ran the following SQL to create the photos table:

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

I then inserted sample data:

```sql
INSERT INTO photos (photo_title, description, creation_date, keywords, s3_reference)
VALUES
  ('Altona Beach', 'Beach sunset', '2025-04-13', 'beach, sunset', 'https://photoalbum-images-bucket.s3.us-east-1.amazonaws.com/P1.jpg'),
  ('St Kilda Beach', 'Beach sunset', '2025-04-13', 'Sunset Beach', 'https://photoalbum-images-bucket.s3.us-east-1.amazonaws.com/P2.jpg'),
  ('Swinburne', 'Swinburne College Building', '2025-04-13', 'Building, College', 'https://photoalbum-images-bucket.s3.us-east-1.amazonaws.com/P3.jpg');
```

---

Step 8 — Configure the Application

I edited the `constants.php` file in my application code to connect to RDS and S3:

```php
<?php

// [ACTION REQUIRED] your full name
define('STUDENT_NAME', 'Anmol Srivastava');
// [ACTION REQUIRED] your Student ID
define('STUDENT_ID', '105026098');
// [ACTION REQUIRED] your tutorial session
define('TUTORIAL_SESSION', 'Tuesday 06:00PM');

// [ACTION REQUIRED] name of the S3 bucket that stores images
define('BUCKET_NAME', 'photoalbum-images-bucket');
// [ACTION REQUIRED] region of the above bucket
define('REGION', 'us-east-1');
// no need to update this const
define('S3_BASE_URL','https://'.BUCKET_NAME.'.s3.amazonaws.com/');

// [ACTION REQUIRED] name of the database that stores photo meta-data (note that this is not the DB identifier of the RDS instance)
define('DB_NAME', 'photo_album');
// [ACTION REQUIRED] endpoint of RDS instance
define('DB_ENDPOINT', 'photoalbum-db.ceqvjmn1uerm.us-east-1.rds.amazonaws.com');
// [ACTION REQUIRED] username of your RDS instance 
define('DB_USERNAME', 'admin');
// [ACTION REQUIRED] password of your RDS instance
define('DB_PWD', 'Swin2025!');

// [ACTION REQUIRED] name of the DB table that stores photo's meta-data
define('DB_PHOTO_TABLE_NAME', 'photos');
// The table above has 5 columns:
// [ACTION REQUIRED] name of the column in the above table that stores photo's titles
define('DB_PHOTO_TITLE_COL_NAME', 'photo_title');
// [ACTION REQUIRED] name of the column in the above table that stores photo's descriptions
define('DB_PHOTO_DESCRIPTION_COL_NAME', 'description');
// [ACTION REQUIRED] name of the column in the above table that stores photo's creation dates
define('DB_PHOTO_CREATIONDATE_COL_NAME', 'creation_date');
// [ACTION REQUIRED] name of the column in the above table that stores photo's keywords
define('DB_PHOTO_KEYWORDS_COL_NAME', 'keywords');
// [ACTION REQUIRED] name of the column in the above table that stores photo's links in S3 
define('DB_PHOTO_S3REFERENCE_COL_NAME', 's3_reference');
?>
```

---

Step 9 — Configure Network ACLs

I added a Network ACL (`PublicSubnet2NACL`) to Public Subnet 2 as an extra layer of security:

| Rule  | Type | Protocol | Port | Source | Allow/Deny |
|--------|------|----------|------|--------|------------|
| 100 | SSH | TCP | 22 | 0.0.0.0/0 | Allow |
| 110 | ICMP | All ICMP | All | 10.0.2.0/24 | Allow |
| 120 | HTTP | TCP | 80 | 0.0.0.0/0 | Allow |
| 130 | HTTPS | TCP | 443 | 0.0.0.0/0 | Allow |
| * | All traffic | All | All | 0.0.0.0/0 | Deny |

> Unlike Security Groups (which remember connections), NACLs are stateless — I had to explicitly allow both inbound AND outbound traffic for every rule.

---

Accessing the Application

Once I had everything set up, I opened a browser and navigated to:

```
http://18.233.133.57/cos80001/photoalbum/album.php
```

I could see the photo album with images loaded from S3 and details pulled from the RDS database.

---

Security Design Summary

I followed a defence-in-depth approach — multiple layers of security:

1. VPC Isolation — I kept all resources inside a private network, not exposed directly to the internet
2. Public vs Private Subnets — I made the web server public-facing and locked the database in a private subnet
3. Security Groups — I configured instance-level rules so RDS only accepts connections from the web server
4. Network ACLs — I added subnet-level rules for an extra firewall layer
5. No Public RDS Access — I ensured the database has no public IP at all
6. S3 Bucket Policy — I granted only read access publicly so no one can delete or modify files

---

Project Structure

```
photo-album/
│
├── constants.php           DB and S3 connection settings
├── album.php               Main webpage displaying photos
├── defaultstyle.css         
├── mydb.php
├── photo.php
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
