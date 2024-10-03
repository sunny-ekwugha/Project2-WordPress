```markdown
# Host a WordPress Website on AWS

This repository provides a comprehensive guide to hosting a WordPress website on Amazon Web Services (AWS) using a combination of services including EC2, RDS, EFS, and an Application Load Balancer (ALB).

## Table of Contents

- [Prerequisites](#prerequisites)
- [Setup Instructions](#setup-instructions)
- [Installation Script](#installation-script)
- [Accessing WordPress](#accessing-wordpress)
- [Domain Registration](#domain-registration)
- [License](#license)

## Prerequisites

1. **AWS CLI**: Ensure that the AWS Command Line Interface (CLI) is installed and configured on your machine.
2. **IAM User**: Create an IAM user if you don't already have one.
3. **Access Keys**: Generate Access and Secret Keys for the IAM user.
4. **Configuration**: Configure your IAM credentials on your local machine:
   ```bash
   aws configure
   ```
5. **Test Configuration**: Verify your setup:
   ```bash
   aws iam get-user
   ```

## Setup Instructions

### Step 1: Create VPC

1. Create a Virtual Private Cloud (VPC) and enable DNS hostname in the VPC settings.
2. Create an Internet Gateway and attach it to the VPC.

### Step 2: Create Subnets

- **Public Subnets** (in Availability Zones 1 and 2) for the Application Load Balancer and NAT Gateway.
- **Private App Subnets** (in Availability Zones 1 and 2) for the EC2 instances.
- **Private Data Subnets** (in Availability Zones 1 and 2) for RDS and EFS.

### Step 3: Create Route Tables

1. **Public Route Table**:
   - Destination: `0.0.0.0/0`
   - Target: Internet Gateway
   - Associate with Public Subnets.

2. **Private Route Table**:
   - Destination: `0.0.0.0/0`
   - Target: NAT Gateway
   - Associate with Private App and Data Subnets.

### Step 4: Create NAT Gateway

- Enable instances in private subnets to connect securely to the internet.

### Step 5: Set Up Security Groups

1. **ALB Security Group**: Allow HTTP (80) and HTTPS (443) access from anywhere, SSH (22) access restricted to EICE security group.
2. **EICE Security Group**: SSH access restricted to VPC CIDR block.
3. **Webapp Security Group**: Allow HTTP and HTTPS access restricted to ALB security group.
4. **Database Security Group**: Allow MySQL/Aurora access restricted to Webapp security group.
5. **EFS Security Group**: Allow NFS (2049) and SSH (22) access restricted to appropriate groups.

### Step 6: Create Resources

1. **EC2 Instance Connect Endpoint (EICE)**: For secure SSH access from the public to the private subnet.
2. **Elastic File System (EFS)**: For scalable NFS file storage.
3. **Relational Database Service (RDS)**: Set up an RDS instance.
4. **EC2 Instance**: Launch an EC2 instance.
5. **Target Group**: Route requests to registered EC2 targets.
6. **Application Load Balancer (ALB)**: Distribute incoming traffic.

## Installation Script

Use the following script to install WordPress on the EC2 instance:

```bash
# Update software packages
sudo yum update -y

# Create HTML directory
sudo mkdir -p /var/www/html

# Mount EFS
# Change the fs number to your fs number
sudo mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport fs-<your-efs-id>.efs.us-east-1.amazonaws.com:/ /var/www/html

# Install Apache web server
sudo yum install -y httpd
sudo systemctl enable httpd
sudo systemctl start httpd

# Install PHP 8 and necessary extensions
sudo dnf install -y \
php \
php-cli \
php-cgi \
php-curl \
php-mbstring \
php-gd \
php-mysqlnd \
php-gettext \
php-json \
php-xml \
php-fpm \
php-intl \
php-zip \
php-bcmath \
php-ctype \
php-fileinfo \
php-openssl \
php-pdo \
php-tokenizer

# Install MySQL Server
sudo wget https://dev.mysql.com/get/mysql80-community-release-el9-1.noarch.rpm
sudo dnf install -y mysql80-community-release-el9-1.noarch.rpm
sudo rpm --import https://repo.mysql.com/RPM-GPG-KEY-mysql-2023
sudo dnf install -y mysql-community-server

# Start MySQL server
sudo systemctl start mysqld
sudo systemctl enable mysqld

# Set permissions
sudo usermod -a -G apache ec2-user
sudo chown -R ec2-user:apache /var/www
sudo chmod 2775 /var/www
sudo find /var/www -type d -exec sudo chmod 2775 {} \;
sudo find /var/www -type f -exec chmod 664 {} +
sudo chown apache:apache -R /var/www/html

# Download WordPress
wget https://wordpress.org/latest.tar.gz
tar -xzf latest.tar.gz
sudo cp -r wordpress/* /var/www/html/

# Configure WordPress
sudo cp /var/www/html/wp-config-sample.php /var/www/html/wp-config.php
sudo vi /var/www/html/wp-config.php

# Restart the web server
sudo service httpd restart
```

## Accessing WordPress

1. SSH into the EC2 instance using the EC2 instance endpoint connect:
   ```bash
   aws ec2-instance-connect ssh --instance-id <instance-id>
   ```
2. After installation, restart Apache to enable the changes:
   ```bash
   sudo service httpd restart
   ```
3. Open a web browser and navigate to the DNS name of the Load Balancer to access your WordPress site.

## Domain Registration

Register a domain name in AWS Route 53 to point to your Load Balancer's DNS name for easier access.

```

### Notes:
- Make sure to replace placeholders like `<your-efs-id>` and `<instance-id>` with actual values.
- You can add more sections based on your specific needs or additional features of your setup.
- Consider including a troubleshooting section if necessary. 

