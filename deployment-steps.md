# Deployment Steps

## 1. Architecture Overview

This project demonstrates a production-style **3-Tier Architecture** deployed on **AWS Free Tier**.

### Architecture Flow

User
↓
Route 53 (DNS)
↓
Application Load Balancer (External)
↓
Web Tier (EC2 + Nginx + React)
↓
Application Load Balancer (Internal)
↓
App Tier (EC2 + Node.js + Express + PM2)
↓
Amazon RDS (MySQL)


## 2. Create Networking - Use VPC More
   • VPC - 3-tier-vpc-project
   • Public Subnets - 2 for Bastion
   • Private Subnets - 4 web-tiar and app-tiar
   • Route Tables
   • Internet Gateway
   • NAT Gateway - Zonal
     VPC Endpoint - None

## 3. Configure Security Groups

| Security Group  | Attached To                        | Inbound Rules                           | Purpose                                                                                                |
| --------------- | ---------------------------------- | --------------------------------------- | ------------------------------------------------------------------------------------------------------ |
| **WebALB-SG**   | External Application Load Balancer | HTTP (80), HTTPS (443) from `0.0.0.0/0` | Internet se user traffic ko External ALB tak allow karta hai.                                          |
| **Web-SG**      | Web Server (EC2)                   | HTTP (80) from **WebALB-SG**            | Sirf External ALB ko Web Server access karne ki permission deta hai. Direct internet access nahi.      |
| **AppALB-SG**   | Internal Application Load Balancer | HTTP (80) from **Web-SG**               | Web Server se API requests receive karta hai aur App Server ko forward karta hai.                      |
| **App-SG**      | Application Server (EC2)           | Custom TCP **4000** from **AppALB-SG**  | Node.js backend application (Port 4000) ko sirf Internal ALB access kar sake.                          |
| **Database-SG** | RDS MySQL                          | MySQL/Aurora **3306** from **App-SG**   | Sirf App Server ko database access karne ki permission deta hai. RDS internet se accessible nahi hota. |


## 4. Create Private S3 Bucket
   • Upload application-code 
   • Modify configuration files - DbConfig.js (RDS endpoint) and nginx.conf (LB DNSName) 

## 5. Create IAM Role
  Create a IAM ROLE (3-tier-role) and later we attach to the EC2 instance, Permissions = SSM or Admin

## 6. Create RDS MySQL
   • DB Subnet Group 
   • Database
   • Attach DB Security Group 
   • Create Schema

NOTE--> Here Confirm that your Endpoint,Username,Password and update DbConfig.js

## 7. Deploy Application Tier
   • Launch EC2 Amazone
   • Install Packages
   • Install Node
   • Install PM2
   • Copy Code
   • Start Application
   • Verify

## Launch EC2 (privet)

Name : App-Server

AMI : Amazon Linux 2023

Instance Type : t3.micro

Subnet : APP1

Security Group : App-SG

IAM Role : attach 3-tier-role

---

## Install MariaDB Client

TO connect to App Server which is in Private Subnet, we need Bastion/Jump Server. Instead now, we use SSM to connect directly to App Server or use VPC EC2 Endpoints

```bash
sudo -s
cd /home/ec2-user
sudo dnf install -y mariadb105  (install MySQL Client)
```

---

## Connect RDS

```bash
mysql -h <RDS-ENDPOINT> -u admin -p
```

---

## Create Database

```sql
CREATE DATABASE webappdb;

SHOW DATABASES;

USE webappdb;

CREATE TABLE transactions (
 id INT AUTO_INCREMENT PRIMARY KEY,
 amount DECIMAL(10,2),
 description VARCHAR(100)
);

SHOW TABLES;
INSERT INTO transactions (amount, description) VALUES ('400', 'awsbill');
SELECT * FROM transactions;

```

---

## Install Node.js

```bash
curl -o- https://raw.githubusercontent.com/ReyazShaik/3tier-app-deployment-aws/main/install.sh | bash

source ~/.bashrc

nvm install 16

nvm use 16
```

---

## Install PM2

```bash
npm install -g pm2   
```

---

## Download Application

```bash
aws s3 cp s3://<bucket-name>/application-code/app-tier/ app-tier --recursive
```

---

## Install Packages

```bash
cd app-tier

npm install
```

---

## Start Backend

```bash
pm2 start index.js
pm2 status
pm2 list
pm2 logs
pm2 startup   [to make it startup after reboot]
```

---

## Verify

```bash
curl http://localhost:4000/health
```

Expected Output

```
"This is the health check"
```

## 8. Configure Internal Load Balancer Target Group

Create target group
Target Type = Instance
Target Group Name = App-TG
Protocol = http, port = 4000
VPC = 3-tier-vpc-project
Health Check = /health  [curl http://localhost:4000/health as application is using /health for health check]
Select App-Server
Create Target Group
---------------------------------------------------
Now Create ALB
Name = app-internal-alb
Scheme = Internal
VPC = 3-tier-vpc-project
AZ = 1a = APP1
AZ = 1b = APP2
Security Group = AppALB-SG
Protocol http = 80 = Select target group = App-TG

After Internal ALB is created , update nginx.conf in application-code --> New ALB DNS

        location /api/ {
            proxy_pass http://internal-app-internal-alb-432718895.ap-south-1.elb.amazonaws.com:80/;
        }

Upload it back to S3 .


## 9. Deploy Web Tier - Public Ec2
   • Launch Amazon Linux 2023 Instance
   • Name = Web-Server
   • Security Group = Web-SG
   • IAM role attach Instance role = 3-tier-role

## Connect to this instance with session manager 

sudo -s
cd /home/ec2-user

curl -o- https://raw.githubusercontent.com/ReyazShaik/3tier-app-deployment-aws/main/install.sh | bash
source ~/.bashrc
nvm install 16
nvm use 16

aws s3 cp s3://3-tier-project-demo-6pm/application-code/web-tier/ web-tier --recursive

cd ~/web-tier
npm install
npm run build  [if you get build error, run the below command. this is due to some version mismatch]

npm install eslint-plugin-jest@27 --save-dev

sudo dnf install -y nginx

## Update Nginx configuration:

cd /etc/nginx

ls

sudo rm nginx.conf   [Remove the default file and download our nginx.conf from S3 where it has our own configuration]

sudo aws s3 cp s3://3-tier-project-demo-6pm/application-code/nginx.conf .

sudo systemctl restart nginx

chmod -R 755 /home/ec2-user

sudo chkconfig nginx on




## 10. Configure External Load Balancer and Target Group

Create target group
Target Type = Instance
Target Group Name = Web-TG
Protocol = http, port = 80
VPC = 3-tier-vpc-project
Health Check = /
Select Web-Server
Create Target Group

## Create a Internet facing load balancer

Name = app-external-alb
Scheme = internet
VPC = 3-tier-vpc-project
AZ = 1a = Public1
AZ = 1b = Public2
Security Group = WebALB-SG
Protocol http = 80 = Select target group = Web-TG

Go to external elb --> listeners and rules --> add listener --> https and select ACM certificate





## 11. Configure Route53
Go in Rout53 and create Records 

## 12. Configure ACM
Select Region --> N.Verginia Request ACM for Your Domain

## 13. Enable HTTPS

Go to external elb --> listeners and rules --> add listener --> https and select ACM certificate 

Now Your Domain Will be Working.

## 14. Configure Auto Scaling (Optional)

Create an AMI from the configured Web Server and App Server.
Create Web and app Launch Templatet
Create Auto Scaling Groups Web-ASG and App-ASG

## 15. Validation Tests

## Infrastructure Validation
VPC created successfully
Public and Private Subnets created
Security Groups configured correctly
IAM Role attached to EC2
RDS endpoint reachable from App Server
Internal ALB healthy
External ALB healthy

## Application Validation
Web application opens successfully
User can add transactions
Transactions are stored in MySQL RDS
Transactions can be fetched successfully
Health Check returns HTTP 200
API requests are forwarded through Nginx
Internal ALB routes traffic correctly
HTTPS works successfully using ACM and Route53

## Commands Used

curl http://localhost/health

curl http://localhost/api/transaction

curl http://localhost:4000/health

pm2 status

systemctl status nginx

mysql -h <RDS-ENDPOINT> -u admin -p

## 16. Troubleshooting Reference

| Issue                            | Cause                             | Solution                                   |
| -------------------------------- | --------------------------------- | ------------------------------------------ |
| Web Target Group Unhealthy (500) | React build missing               | Run `npm run build` and restart Nginx      |
| Web Target Group Unhealthy       | Nginx configuration error         | Validate using `sudo nginx -t`             |
| 502 Bad Gateway                  | Backend service stopped           | Restart PM2 application                    |
| 504 Gateway Timeout              | App Server unreachable            | Verify Internal ALB and Security Groups    |
| Cannot GET /transaction          | Wrong API endpoint                | Verify Nginx `proxy_pass` configuration    |
| Database connection failed       | Wrong RDS endpoint or credentials | Update `DbConfig.js`                       |
| HTTPS not working                | ACM not attached to ALB           | Attach ACM certificate to HTTPS Listener   |
| HTTPS Connection Refused         | Security Group missing Port 443   | Allow HTTPS (443) on External ALB SG       |
| Internal ALB Unhealthy           | Health check path incorrect       | Use `/health`                              |
| App Server Unhealthy             | Node.js application not running   | Check `pm2 logs`                           |
| React page shows 500             | Build folder missing              | Run `npm install` then `npm run build`     |
| Route53 not resolving            | DNS record incorrect              | Verify Alias record points to External ALB |
| RDS not reachable                | Security Group issue              | Allow MySQL 3306 from App-SG               |
| Nginx failed to start            | Syntax error                      | Run `sudo nginx -t`                        |

