# Host a WordPress Website on AWS

## Overview

This project demonstrates how to deploy a WordPress website on Amazon Web Services (AWS) using a scalable, multi-tier architecture. It explores cloud networking, web hosting, shared file storage, managed databases, load balancing, and secure access to application servers.

The architecture is designed to separate the web application from the database and use shared storage for WordPress files.

### AWS Services Used

| Service | Purpose |
|---|---|
| Amazon VPC | Provides an isolated network for the application |
| Public and private subnets | Separates internet-facing resources from application and data resources |
| Internet Gateway | Provides internet connectivity for public resources |
| NAT Gateway | Allows private application instances to initiate outbound internet connections |
| Amazon EC2 | Hosts the Apache web server and WordPress application |
| Amazon EFS | Provides shared file storage for WordPress files |
| Amazon RDS for MySQL | Stores WordPress application data |
| Application Load Balancer (ALB) | Distributes incoming HTTP/HTTPS requests across healthy application instances |
| EC2 Instance Connect Endpoint | Enables access to supported private EC2 instances without requiring a public IP address |
| Amazon Route 53 | Provides DNS hosting and domain routing |
| AWS Certificate Manager (ACM) | Provides TLS certificates for HTTPS on supported AWS load balancers |
| Amazon CloudWatch and SNS | Can be used for monitoring, alarms, and notifications |

## Architecture and Security

A typical deployment uses the following layout:

- **Public subnets:** Host the Application Load Balancer and NAT Gateway across Availability Zones.
- **Private application subnets:** Host EC2 instances running Apache, PHP, and WordPress.
- **Private data subnets:** Host the RDS database. EFS mount targets should be created in the appropriate Availability Zones.
- **Shared storage:** EFS allows multiple application instances to access shared WordPress files.
- **Database:** RDS stores WordPress content, user accounts, settings, and other application data.

### Recommended Security Group Rules

| Security group | Inbound access |
|---|---|
| ALB | HTTP (80) and/or HTTPS (443) from intended client networks, commonly the internet for a public website |
| Web application | HTTP (80) only from the ALB security group |
| RDS | MySQL (3306) only from the web application security group |
| EFS | NFS (2049) only from the web application security group |
| EC2 Instance Connect Endpoint | Configure endpoint and instance security rules according to AWS EC2 Instance Connect requirements |

Avoid opening SSH or database ports to the entire internet. Use IAM roles for EC2 access to AWS services instead of storing long-lived AWS access keys on instances.

## Prerequisites

Before starting, you should have:

- An AWS account with appropriate permissions to create the required resources.
- Basic familiarity with VPCs, subnets, route tables, and security groups.
- An EC2 instance running a compatible Amazon Linux release.
- An EFS file system with mount targets and correctly configured security groups.
- An RDS for MySQL database in private subnets.
- A target group and Application Load Balancer.
- A domain name, if you want to configure a custom domain.
- A plan for securely storing database credentials.

The resources in this project may incur AWS charges. Review the pricing of EC2, RDS, EFS, NAT Gateway, and the load balancer before deployment.

## Deployment Guide

### Step 1: Create the VPC and Subnets

1. Create a VPC and enable DNS resolution and DNS hostnames.
2. Create public subnets in at least two Availability Zones.
3. Create private application subnets in at least two Availability Zones.
4. Create private database subnets in at least two Availability Zones.
5. Attach an Internet Gateway to the VPC.
6. Associate public subnets with a route table containing a default route to the Internet Gateway.
7. Configure private application subnet routing through a NAT Gateway if outbound internet access is required.

Database subnets should remain private. A NAT Gateway is generally not required for database subnets unless there is a specific networking requirement.

### Step 2: Configure Security Groups

Create security groups for the ALB, web application, database, and EFS.

Reference the appropriate source security group for application-to-database and application-to-EFS communication. Avoid relying solely on broad VPC CIDR rules when a security group reference is suitable.

### Step 3: Create EFS

1. Create an Amazon EFS file system.
2. Create mount targets in the Availability Zones used by the application.
3. Allow inbound NFS traffic on port 2049 from the web application security group.
4. Verify that the EC2 instances can reach the EFS mount targets.

### Step 4: Create the RDS Database

1. Create an RDS for MySQL database.
2. Place it in a private DB subnet group.
3. Disable public accessibility.
4. Configure the database security group to accept MySQL connections only from the web application security group.
5. Create the WordPress database and database user using secure credentials.

Keep the database endpoint, database name, username, and password available for WordPress configuration. Do not commit passwords or other secrets to GitHub.

### Step 5: Launch the EC2 Application Instance

Launch an EC2 instance in a private application subnet using a supported Amazon Linux release.

Install and configure Apache, PHP, and the PHP extensions required by your chosen WordPress version. Configure the instance to mount EFS and connect to RDS.

For production-style deployments, consider using a launch template and Auto Scaling group so that instances can be replaced and scaled as needed.

### Step 6: Install WordPress

The following is an illustrative installation script for Amazon Linux 2023. Review it against your environment before running it. It assumes that the EFS file system is empty or already prepared for this application.

Replace `fs-REPLACE_WITH_YOUR_ID` and `us-east-1` with your actual EFS file system ID and AWS Region.

```bash
#!/bin/bash
set -euo pipefail

EFS_ID="fs-REPLACE_WITH_YOUR_ID"
AWS_REGION="us-east-1"
WEB_ROOT="/var/www/html"

# Install the web server, PHP, and EFS utilities.
dnf update -y
dnf install -y httpd php php-fpm php-mysqlnd \
  php-curl php-gd php-mbstring php-xml \
  php-intl php-zip amazon-efs-utils tar

# Start the web server and PHP-FPM.
systemctl enable --now php-fpm
systemctl enable --now httpd

# Mount EFS as the shared WordPress document root.
mkdir -p "$WEB_ROOT"

if ! mountpoint -q "$WEB_ROOT"; then
  mount -t efs -o tls \
    "${EFS_ID}:/" "$WEB_ROOT"
fi

# Persist the mount across reboots.
grep -qF "${EFS_ID}:/ ${WEB_ROOT} efs" /etc/fstab || \
  echo "${EFS_ID}:/ ${WEB_ROOT} efs _netdev,tls 0 0" \
  >> /etc/fstab

# Download and extract WordPress into a temporary directory.
TMP_DIR="$(mktemp -d)"
trap 'rm -rf "$TMP_DIR"' EXIT

curl -fsSL https://wordpress.org/latest.tar.gz \
  -o "$TMP_DIR/wordpress.tar.gz"

tar -xzf "$TMP_DIR/wordpress.tar.gz" -C "$TMP_DIR"

# Copy WordPress files to the mounted EFS document root.
cp -a "$TMP_DIR/wordpress/." "$WEB_ROOT/"

# Set basic ownership and permissions.
chown -R apache:apache "$WEB_ROOT"
find "$WEB_ROOT" -type d -exec chmod 755 {} +
find "$WEB_ROOT" -type f -exec chmod 644 {} +

systemctl restart php-fpm
systemctl restart httpd

echo "WordPress files have been installed."
echo "Configure wp-config.php with your RDS details."
```

**Important:** This is a starting point, not a complete production deployment. Verify the EFS mount, PHP-FPM/Apache integration, file permissions, and EFS access-point configuration for your environment. Ensure the EFS document root does not contain files you need to preserve before copying WordPress into it. Run the installation only once per shared file system, or use a controlled deployment process for subsequent instances.

### Step 7: Configure WordPress and RDS

1. Open the WordPress installation page through the ALB after confirming the web server is reachable.
2. Follow the WordPress setup wizard, or configure `wp-config.php` with your RDS database name, username, password, and endpoint.
3. Use a secure method to provide database credentials. For production, consider AWS Secrets Manager or another approved secret-management approach.
4. Verify that WordPress can connect to RDS and create its database tables.
5. Confirm that shared files remain available when application instances are replaced.

Do not install a separate local MySQL server on the EC2 instance when RDS is the intended database service.

### Step 8: Configure the Load Balancer and Domain

1. Create a target group for the WordPress EC2 instances.
2. Configure a health check that matches the application's expected response.
3. Create an Application Load Balancer in the public subnets.
4. Register the application instances and verify that they become healthy.
5. Configure an HTTPS listener using an ACM certificate in the same Region as the load balancer.
6. If you have a custom domain, configure the appropriate Route 53 record to point to the ALB.

WordPress should be configured with the correct site URL and HTTPS settings when HTTPS is enabled.

## Accessing the Application

For private EC2 instances, connect through an appropriately configured EC2 Instance Connect Endpoint or another approved management method.

The `aws ec2-instance-connect ssh` command requires the AWS CLI and EC2 Instance Connect support to be configured correctly. The endpoint and instance security groups must permit the required connection.

Once the ALB target is healthy, open its DNS name in a browser to test the site. After configuring DNS and HTTPS, test the custom domain as well.

## Troubleshooting

- **The ALB target is unhealthy:** Check the target port, health-check path, Apache status, and security group rules.
- **The site cannot connect to the database:** Verify the RDS endpoint, database credentials, database security group, and private network connectivity.
- **EFS will not mount:** Check the file system ID, mount targets, NFS port 2049, security groups, and EFS utilities.
- **PHP pages do not execute correctly:** Check PHP-FPM status, Apache configuration, and installed PHP extensions.
- **WordPress cannot upload files:** Review EFS access permissions, ownership, and the directory used for uploads.
- **HTTPS or redirects fail:** Check the ACM certificate, ALB listener, WordPress site URL, and HTTPS/proxy configuration.

## Cost Management and Cleanup

AWS resources may continue generating charges after the website is no longer in use. When you have finished testing, review and remove resources you no longer need, including:

- EC2 instances and unused EBS volumes.
- Application Load Balancers and target groups.
- NAT Gateways and associated public IPv4 addresses.
- RDS database instances and unused snapshots.
- EFS file systems and mount targets.
- Route 53 hosted zones and records that are no longer required.

Back up any data you want to retain before deleting resources. Do not delete shared or production resources without confirming their ownership and purpose.

## What This Project Demonstrates

- AWS VPC design and subnet segmentation.
- Public and private application architecture.
- EC2-based web hosting with Apache and PHP.
- Shared storage with Amazon EFS.
- Managed relational databases with Amazon RDS.
- Load balancing, health checks, and availability across Availability Zones.
- DNS and HTTPS configuration.
- Security group design and cloud resource cost awareness.

## License

Add a license file if you intend to distribute this project under a specific open-source license. Otherwise, state the repository's actual licensing terms here.
