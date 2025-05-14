Project Overview
Cloud Provider: AWS (or Azure alternative provided below)
App: Simple Flask app in a Docker container
CI/CD: GitHub Actions
Container Hosting: AWS ECS with Fargate (or Azure Web App for Containers)
Monitoring: Zabbix on EC2 (or Azure VM)
________________________________________
 Step-by-Step Deployment on AWS
________________________________________
Log into AWS Account and create VPC as shown in below pattern
Step-by-Step Guide to Create a VPC Manually in AWS
Step 1: Login to AWS Console
Go to https://aws.amazon.com
Sign in to the AWS Management Console.
________________________________________
Step 2: Navigate to VPC Dashboard
In the top search bar, type “VPC” and select VPC under “Services”.
Click “Your VPCs” from the left sidebar.
________________________________________
Step 3: Create a Custom VPC
Click “Create VPC”.
Choose “VPC only” option.
Fill in the details:
Name tag: MyCustomVPC
IPv4 CIDR block: 10.0.0.0/16 (or any private IP range)
Leave IPv6 settings (optional)
Tenancy: Default (or Dedicated, if needed)
Click “Create VPC”.
________________________________________
Step 4: Create Subnets
You'll typically create:
1 Public Subnet (for internet-facing resources like EC2)
1 Private Subnet (for internal services like databases)
Go to Subnets > Click “Create subnet”.
Choose your VPC (MyCustomVPC).
Add subnet details:
Public Subnet:
Name: PublicSubnet
AZ: e.g., us-east-1a
CIDR: 10.0.1.0/24
Private Subnet:
Name: PrivateSubnet
AZ: us-east-1a
CIDR: 10.0.2.0/24
Click “Create subnet”.
________________________________________
Step 5: Create and Attach Internet Gateway
Go to Internet Gateways > Click “Create internet gateway”.
Name it: MyIGW.
Click “Create internet gateway”.
After creation, select it and click “Attach to VPC” > Select MyCustomVPC.
________________________________________
Step 6: Create Route Tables
Go to Route Tables > Click “Create route table”.
Name it: PublicRouteTable.
Select MyCustomVPC > Click “Create”.
Add Route to the Internet:
Select PublicRouteTable > Routes tab > Edit routes > Add route:
Destination: 0.0.0.0/0
Target: Internet Gateway (MyIGW)
Save the route.
Associate Public Subnet:
Under Subnet Associations tab > Click Edit subnet associations.
Select PublicSubnet and Save.
________________________________________
Step 7: Modify Public Subnet to Auto-Assign IP
Go to Subnets > Select PublicSubnet.
Click Actions > Modify auto-assign IP settings.
Enable “Auto-assign IPv4 address”.
Save.
________________________________________
Step 8: (Optional) Create NAT Gateway (For Private Subnet Internet Access)
Go to NAT Gateways > Click Create NAT Gateway.
Choose:
Subnet: PublicSubnet
Allocate or create a new Elastic IP.
Click Create NAT Gateway.
Add Private Route Table:
Create a new route table: PrivateRouteTable.
Add route:
Destination: 0.0.0.0/0
Target: NAT Gateway
Associate it with PrivateSubnet.
Then next go for creating the EC2 instance as per the requirement to build nodejs app and also for the purpose of dockerization as below 
Step 1: Log In to AWS
Visit https://aws.amazon.com
Sign in to the AWS Management Console.
________________________________________
Step 2: Go to EC2 Dashboard
In the search bar at the top, type EC2.
Click EC2 under “Services”.
________________________________________
Step 3: Launch an Instance
Click “Instances” in the left sidebar.
Click the “Launch Instances” button.
________________________________________
Step 4: Configure Instance Details
1. Name and Tags
Enter a Name for your instance (e.g., Zabbbix_server).
2. Application and OS Image (Amazon Machine Image - AMI)
Select an AMI (e.g., Amazon Linux 2023
Example: Amazon Linux 2023 AMI (HVM), SSD Volume Type
3. Instance Type
Choose an instance type (e.g., t2.micro – Free Tier eligible).
4. Key Pair (login)
Select an existing key pair or create a new one (download the .pem file).
Important: You need this file to SSH into your instance.
Example: zabbix.pem
5. Network Settings
Choose your VPC and subnet (e.g., MyCustomVPC, PublicSubnet).
Enable Auto-assign public IP (for internet access).
Create or select a Security Group:
Add rule to allow SSH (port 22) from your IP.
You can also allow HTTP/HTTPS if you're hosting a web server.
6. Configure Storage
Default is usually 8 GiB for Amazon Linux.
You can increase if needed.
________________________________________
Step 5: Launch the Instance
Review settings.
Click “Launch Instance”.
Wait for a few seconds. You will see a message:
“Your instances are now launching”.
Click “View all instances”.
________________________________________
Step 6: Connect to Your EC2 Instance
Using SSH (Linux/Mac/WSL)
bash
CopyEdit
chmod 400 MyKeyPair.pem
ssh -i "zabbix.pem" ec2-user@<Public-IP>
Replace <Public-IP> with the actual public IP of your instance.
The username depends on the AMI:
Amazon Linux: ec2-user
Ubuntu: ubuntu
RedHat: ec2-user
CentOS: centos
 Using EC2 Instance Connect (via browser)
Select the instance.
Click “Connect” > “EC2 Instance Connect” > Click “Connect”.

After logging into the server install the necessary packages like git, docker, node js bundles, pip installers and all the other dependent tools including aws development kit for pushing the data into aws ECR.
# yum install git -y
#yum install docker-ce -y
#yum install python3 -y
#python3 --version
#pip3 –version
# pip3 install Flask
# pip3 install Flask flask_sqlalchemy flask_cors
# mkdir flask-app
# cd flask-app
1. Create a Simple Flask App
# vi app.py
app.py:
python
CopyEdit
from flask import Flask
app = Flask(__name__)

@app.route('/')
def hello():
    return "Hello from Flask in Docker!"

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=80)
Dockerfile:
Dockerfile
CopyEdit
FROM python:3.9-slim
WORKDIR /app
COPY . /app
RUN pip install flask
EXPOSE 80
CMD ["python", "app.py"]
________________________________________
2. Push App to GitHub Repo
Create a GitHub repo and push your code.
________________________________________
3. Build & Push Docker Image to ECR
Set up ECR:
bash
CopyEdit
aws ecr create-repository --repository-name flask-app
Authenticate and push:
bash
CopyEdit
aws ecr get-login-password | docker login --username AWS --password-stdin <account-id>.dkr.ecr.<region>.amazonaws.com
docker build -t flask-app .
docker tag flask-app:latest <account-id>.dkr.ecr.<region>.amazonaws.com/flask-app:latest
docker push <account-id>.dkr.ecr.<region>.amazonaws.com/flask-app:latest
________________________________________
4. Deploy to AWS ECS with Fargate
Create ECS Cluster (Fargate).
Create Task Definition with your ECR image.
Create Service using the Task.
Attach Application Load Balancer (optional but useful for public access).
Use AWS Console or terraform for IaC.
________________________________________
5. Set Up CI/CD with GitHub Actions
.github/workflows/deploy.yml:
yaml
CopyEdit
name: CI/CD Pipeline

on:
  push:
    branches: [ main ]

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout code
      uses: actions/checkout@v3

    - name: Log in to Amazon ECR
      uses: aws-actions/amazon-ecr-login@v1

    - name: Build and Push to ECR
      run: |
        docker build -t flask-app .
        docker tag flask-app:latest ${{ secrets.AWS_ACCOUNT_ID }}.dkr.ecr.${{ secrets.AWS_REGION }}.amazonaws.com/flask-app:latest
        docker push ${{ secrets.AWS_ACCOUNT_ID }}.dkr.ecr.${{ secrets.AWS_REGION }}.amazonaws.com/flask-app:latest

    - name: Deploy to ECS
      uses: aws-actions/amazon-ecs-deploy-task-definition@v1
      with:
        task-definition: ecs-task-def.json
        service: flask-service
        cluster: flask-cluster
        wait-for-service-stability: true
Store secrets: AWS keys, region, account ID.
________________________________________
6. Deploy Zabbix for Monitoring
Zabbix on EC2:
Step 1: Update and Enable Required Extras
bash
CopyEdit
sudo yum update -y
sudo amazon-linux-extras enable php7.4 mariadb10.5
sudo yum clean metadata
________________________________________
🔹 Step 2: Install Zabbix Repository
bash
CopyEdit
sudo rpm -Uvh https://repo.zabbix.com/zabbix/6.0/rhel/7/x86_64/zabbix-release-6.0-4.el7.noarch.rpm
sudo yum clean all
________________________________________
🔹 Step 3: Install Zabbix Server, Agent, and Web Packages
bash
CopyEdit
sudo yum install -y zabbix-server-mysql zabbix-agent zabbix-web-mysql zabbix-apache-conf
Also install MariaDB and Apache:
bash
CopyEdit
sudo yum install -y mariadb-server httpd
________________________________________
🔹 Step 4: Start and Enable Services
bash
CopyEdit
sudo systemctl enable --now mariadb httpd
________________________________________
🔹 Step 5: Set Up the Zabbix Database
Run:
bash
CopyEdit
sudo mysql_secure_installation
Then log into MariaDB:
bash
CopyEdit
mysql -u root -p
Create the Zabbix DB and user:
sql
CopyEdit
CREATE DATABASE zabbix CHARACTER SET utf8mb4 COLLATE utf8mb4_bin;
CREATE USER zabbix@localhost IDENTIFIED BY 'ZabbixPass123!';
GRANT ALL PRIVILEGES ON zabbix.* TO zabbix@localhost;
FLUSH PRIVILEGES;
EXIT;
________________________________________
🔹 Step 6: Import Initial Schema
bash
CopyEdit
zcat /usr/share/doc/zabbix-server-mysql*/create.sql.gz | mysql -uzabbix -p zabbix
________________________________________
🔹 Step 7: Configure Zabbix Server
Edit:
bash
CopyEdit
sudo nano /etc/zabbix/zabbix_server.conf
Set the DB password:
ini
CopyEdit
DBPassword=ZabbixPass123!
________________________________________
🔹 Step 8: Configure PHP for Zabbix Frontend
Edit:
bash
CopyEdit
sudo nano /etc/httpd/conf.d/zabbix.conf
Set the timezone:
lua
CopyEdit
php_value date.timezone America/New_York
Change to your correct timezone from: https://www.php.net/manual/en/timezones.php
________________________________________
🔹 Step 9: Start and Enable Zabbix Server & Agent
bash
CopyEdit
sudo systemctl restart zabbix-server zabbix-agent httpd
sudo systemctl enable zabbix-server zabbix-agent httpd
________________________________________
🔹 Step 10: Access the Zabbix Web Interface
Open in browser:
arduino
CopyEdit
http://<your-ec2-public-ip>/zabbix
Use:
Username: Admin
Password: zabbix
________________________________________
🔹 Step 11: Open Firewall (If using firewalld)
bash
CopyEdit
sudo firewall-cmd --add-port=80/tcp --permanent
sudo firewall-cmd --add-port=10051/tcp --permanent
sudo firewall-cmd --reload
Set up MySQL, configure DB, and start services.
To monitor ECS apps, install Zabbix agent on an EC2 instance that can communicate with your Fargate service or implement custom HTTP health-check monitoring via Zabbix web scenarios.
________________________________________
Add Zabbix Monitoring to Flask App
Use Zabbix’s HTTP Agent checks:
Set up a web scenario to periodically check http://<flask-app-url>/.
Set trigger if it returns non-200.


 



 





 

 
 
 

