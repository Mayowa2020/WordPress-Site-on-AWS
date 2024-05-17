# DigitalBoost WordPress Solution Setup Guide

## Overview

This guide outlines the steps to set up a high-availability WordPress solution using AWS services. The setup includes configuring a VPC with public and private subnets, deploying MySQL on RDS, integrating Elastic File System (EFS), setting up an Application Load Balancer with Auto Scaling, and configuring a Bastion host for secure access.

## Table of Contents

1. [VPC Setup](#1-vpc-setup)
2. [Public and Private Subnets with NAT Gateway](#2-public-and-private-subnets-with-nat-gateway)
3. [AWS MySQL RDS Setup](#3-aws-mysql-rds-setup)
4. [EFS Setup for WordPress Files](#4-efs-setup-for-wordpress-files)
5. [Application Load Balancer and Auto Scaling](#5-application-load-balancer-and-auto-scaling)
6. [Project Execution](#guided-tour-of-project-execution)

## 1. VPC Setup

- **Create a VPC** with both public and private subnets across 2 availability zones for high availability and fault tolerance.
- **Set up an Internet Gateway** to enable communication between instances within the VPC and the Internet.
- **Allocate web servers and database servers** to private subnets.
- **Configure route tables** for each subnet.

## 2. Public and Private Subnets with NAT Gateway

- **Implement a secure network architecture**:
  - Create a public subnet for resources accessible from the internet.
  - Create a private subnet for resources without direct internet access.
  - Set up a NAT Gateway to allow private subnet internet access.

## 3. AWS MySQL RDS Setup

- **Deploy a managed MySQL database** using Amazon RDS to store WordPress data.
- **Configure security groups** for the RDS instance.
- **Connect WordPress** to the RDS database.

## 4. EFS Setup for WordPress Files

- **Create an Elastic File System (EFS)**.
- **Mount the EFS file system** on WordPress instances.
- **Configure WordPress** to utilize the shared file system.

## 5. Application Load Balancer and Auto Scaling

- **Create an Application Load Balancer**.
- **Configure listener rules** to route traffic to instances.
- **Integrate the Load Balancer** with the Auto Scaling group.

## Project Execution

### 1. Create a Custom VPC

#### 1.1. Create the VPC

- Navigate to the VPC service section and select "Your VPCs".
- Click "Create VPC" with the following details:
  - **Name tag**: DigitalBoost-VPC
  - **IPv4 CIDR block**: 10.0.0.0/16
  - **IPv6 CIDR block**: Leave empty (unless needed).
  - **Tenancy**: Default
- Enable DNS hostnames for the VPC.

### 2. Create Subnets

#### 2.1. Public Subnets

- Create two public subnets across two Availability Zones (AZs):
  - **Public Subnet AZ1 (us-east-1a)**: 10.0.0.0/24
  - **Public Subnet AZ2 (us-east-1b)**: 10.0.1.0/24
- Enable auto-assign public IPv4 addresses for both public subnets.

#### 2.2. Private Subnets

- Create two private subnets for the app tier and two for the database tier:
  - **Private App Subnet AZ1 (us-east-1a)**: 10.0.2.0/24
  - **Private App Subnet AZ2 (us-east-1b)**: 10.0.3.0/24
  - **Private Data Subnet AZ1 (us-east-1a)**: 10.0.4.0/24
  - **Private Data Subnet AZ2 (us-east-1b)**: 10.0.5.0/24

### 2.3. Internet Gateway and Route Tables

#### 2.3.1. Create an Internet Gateway (IGW)

- **Name tag**: DigitalBoost-IGW
- Attach the IGW to the VPC.

#### 2.3.2. Create a Public Route Table (Public-RT)

- Associate the Public-RT with the public subnets.
- Add a route for Internet traffic (0.0.0.0/0) via the IGW.

### 2.4. NAT Gateways and Private Route Tables

#### 2.4.1. Create NAT Gateways

- **NAT Gateway AZ1**:
  - **Subnet**: Public Subnet AZ1
  - Allocate an Elastic IP and create the NAT gateway.
- **NAT Gateway AZ2**:
  - **Subnet**: Public Subnet AZ2
  - Allocate an Elastic IP and create the NAT gateway.

#### 2.4.2. Create Private Route Tables

- **Private RT AZ1**:
  - Create a route table and name it Private RT AZ1.
  - Add a route to send traffic to NAT Gateway AZ1.
  - Associate with Private App Subnet AZ1 and Private Data Subnet AZ1.
- **Private RT AZ2**:
  - Create a route table and name it Private RT AZ2.
  - Add a route to send traffic to NAT Gateway AZ2.
  - Associate with Private App Subnet AZ2 and Private Data Subnet AZ2.

### 2.5. Configure Security Groups

Security groups act as virtual firewalls for EC2 instances, controlling inbound and outbound traffic.

1. **Application Load Balancer Security Group (ALB Security Group)**:
   - Create and configure inbound rules to allow HTTP (Port 80) and HTTPS (Port 443) from `0.0.0.0/0`.

2. **SSH Security Group**:
   - Allow SSH (Port 22) from your IP address.

3. **Bastion Security Group**:
   - Allow SSH (Port 22) from the "SSH Security Group".

4. **Webserver Security Group**:
   - Allow SSH (Port 22) from the "Bastion Security Group".
   - Allow HTTP (Port 80) and HTTPS (Port 443) from the "ALB Security Group".

5. **Database Security Group**:
   - Allow MYSQL/Aurora (Port 3306) from the "Webserver Security Group".

6. **EFS Security Group**:
   - Allow NFS (Port 2049) from the "Webserver Security Group".
   - Allow SSH (Port 22) from the "SSH Security Group".

### 3. AWS MySQL RDS Setup

#### Create a DB Subnet Group

1. On the Amazon RDS Dashboard:
   - Navigate to "Subnet groups" and click "Create DB subnet group".
   - Provide the following details:
     - **Name**: database subnets
     - **Description**: database subnet
     - **VPC**: DigitalBoost-VPC
   - Add Subnets:
     - **Availability Zones**: us-east-1a, us-east-1b
     - **Subnets**: Select 10.0.4.0/24, 10.0.5.0/24
   - Create the DB subnet group.

#### Create an Amazon RDS Instance

1. On the RDS Dashboard:
   - Click "Create database" and select "Standard Create".
   - Choose MySQL 8.0.32.
   - **Templates**: Free tier.
   - **DB instance identifier**: wordpressdb
   - **Master username**: admin
   - **Password**: {your_password}
   - **VPC**: DigitalBoost-VPC
   - **DB Subnet group**: database subnets
   - **Publicly accessible**: No
   - **VPC security group**: Database Security Group
   - **Database port**: 3306
   - **Initial database name**: wordpress_database
   - Create the database.

### 4. Deploy a Bastion Host in a Public Subnet

1. On the EC2 Dashboard, launch a new instance:
   - **Name**: Bastion Host
   - **AMI**: Amazon Linux 2
   - **Instance Type**: t2.micro
   - **Key Pair**: bastion-key
   - **VPC**: DigitalBoost-VPC
   - **Subnet**: Public Subnet AZ1
   - **Auto-assign public IP**: Enable
   - **Security Group**: SSH Security Group. Configure this security group to allow SSH access from your IP.

### 5. Deploy a Temporary WordPress Instance in a Private Subnet

#### 5.1. On the EC2 Dashboard, launch a new instance

- **Name**: WordPress Server
- **AMI**: Amazon Linux 2
- **Instance Type**: t2.micro
- **Key Pair**: wp-app-key
- **VPC**: DigitalBoost-VPC
- **Subnet**: Private App Subnet AZ1
- **Auto-assign public IP**: Disable
- **Security Group**: Webserver Security Group

Sure, here are the corrections and some improvements for the next sections:

### 5.2 SSH into the Bastion Host and then into the WordPress Server

To access the WordPress Server securely, first, SSH into the Bastion Host and then into the WordPress Server:

<!-- From your local machine to the Bastion Host -->
```bash
ssh -i bastion-key.pem ec2-user@<BastionHostPublicIP>
```

<!-- From the Bastion Host to the WordPress Server -->
```bash
ssh -i wp-app-key.pem ec2-user@<WordPressPrivateIP>
```

### 5.3 Install WordPress on the WordPress Server

Use the following script to install WordPress on the WordPress Server:

```bash
#!/bin/bash
sudo yum update -y
sudo yum install -y httpd php php-mysqlnd amazon-linux-extras
sudo amazon-linux-extras install php7.4 -y
sudo systemctl start httpd
sudo systemctl enable httpd

cd /var/www/html
sudo wget https://wordpress.org/latest.tar.gz
sudo tar -xzf latest.tar.gz
sudo mv wordpress/* .
sudo rm -rf wordpress latest.tar.gz
sudo chown -R apache:apache *

sudo cp wp-config-sample.php wp-config.php

# Configure WordPress Database Settings:
sudo nano /var/www/html/wp-config.php
```

In `wp-config.php`, update the following lines with your database information:

```php
define( 'DB_NAME', 'database_name_here' );
define( 'DB_USER', 'username_here' );
define( 'DB_PASSWORD', 'password_here' );
define( 'DB_HOST', 'localhost' ); // or your RDS endpoint
```

Set Authentication Keys and Salts by generating and pasting unique keys and salts from the WordPress secret-key service.

### 6. Mount EFS on the WordPress Instance

#### 6.1 Create an EFS file system

- **Name**: wordpress-efs
- **VPC**: DigitalBoost-VPC
- **Security Group**: EFS Security Group
- Click "Customize" and uncheck Encryption of data at rest if you don't need it to avoid additional charges.

#### 6.2 Configure the file system settings

- **VPC**: DigitalBoost-VPC
- **Subnet**: Private Data Subnet AZ1, Private Data Subnet AZ2 (select the two private subnets in the database tier).
- **Availability zones**: us-east-1a, us-east-1b
- **Security groups**: EFS Security Group
- Leave other settings as default and click on create.

Make note of the DNS name of the EFS file system: `fs-015561da8b710c03c.efs.us-east-2.amazonaws.com`

#### 6.3 Attach the EFS file system to the EC2 instances

```bash
sudo yum install -y amazon-efs-utils
sudo mkdir /mnt/efs
sudo mount -t efs fs-015561da8b710c03c:/ /mnt/efs
echo "fs-015561da8b710c03c:/ /mnt/efs efs defaults,_netdev 0 0" | sudo tee -a /etc/fstab
```

#### 6.4 Copy WordPress files to EFS and create a symbolic link

```bash
sudo cp -r /var/www/html/* /mnt/efs/
sudo rm -rf /var/www/html/*
sudo ln -s /mnt/efs/* /var/www/html/
```

### 7. Create a Launch Template for Auto Scaling

To ensure seamless Auto Scaling in the future, let's create a launch template. This template will be used to launch instances for scaling purposes, maintaining high availability for your WordPress application.

#### 7.1 Create an AMI from the WordPress Instance

1. **Navigate to the EC2 Dashboard:**
   - Log in to the AWS Management Console.
   - Go to the **EC2 Dashboard**.

2. **Create an AMI:**
   - Select your WordPress instance from the list of running instances.
   - Click on **Actions** > **Image and templates** > **Create image**.

3. **Configure the AMI:**
   - **Image name:** `wordpress-app-ami`
   - **Image description:** `App tier AMI for Auto Scaling`
   - **No reboot:** Enable (unless specifically required to reboot for consistency).

4. **Tag the Image and Snapshots:**
   - Enable **Tag image and snapshots together**.
   - Add a tag:
     - **Key:** `Name`
     - **Value:** `wordpress-app-ami`

5. **Create the Image:**
   - Click **Create image** and wait for the process to complete.

#### 7.2 Create a Launch Template

1. **Navigate to the Launch Templates Section:**
   - In the EC2 Dashboard, go to **Launch Templates**.

2. **Create a New Launch Template:**
   - Click **Create launch template**.

3. **Configure the Launch Template:**
   - **Launch template name:** `wordpress-app-template`
   - **Template version description:** `First edition of wordpress-app-template`

4. **Application and OS Images (Amazon Machine Image):**
   - **AMI source:** Select **My AMIs**.
   - **Owned by me:** Ensure this filter is selected.
   - **AMI:** Choose the `wordpress-app-ami` you created earlier.

5. **Instance Type:**
   - **Instance type:** `t2.micro` (Free tier eligible).

6. **Key Pair:**
   - **Key pair:** Select your existing key pair used for the WordPress instance, `wp-app-key`.

7. **Network Settings:**
   - **Security Groups:** Choose the existing `Webserver Security Group`.

8. **Storage (Volumes):**
   - Use default volume settings unless customizations are needed.

9. **Advanced Details (Optional):**
   - Configure additional instance details if required.

10. **Tag Specifications (Optional):**
    - Add necessary tags. Example:
      - **Key:** `Name`
      - **Value:** `wordpress-app-instance`

11. **Create the Launch Template:**
    - Review settings and click **Create launch template**.

#### 7.3 Verify the Launch Template

1. **Verification:**
   - Ensure the launch template appears in the list.
   - Check details like AMI, instance type, key pair, and security group settings.

### Summary

1. **Create an AMI from the WordPress Instance:**
   - Navigate to the EC2 Dashboard, select your WordPress instance.
   - Create an AMI named `wordpress-app-ami` for Auto Scaling.

2. **Create a Launch Template:**
   - Go to the Launch Templates section in the EC2 Dashboard.
   - Create a new launch template named `wordpress-app-template`.
   - Configure instance type, key pair, and security group for Auto Scaling.

This setup guarantees that your WordPress instance is ready for Auto Scaling, maintaining high availability and fault tolerance.

### 8 — Deploy an Auto Scaling Group (using the launch template created in Step 7)

#### 8.1 Navigate to Auto Scaling Groups

1. **Open the EC2 Console:**
   - Log in to the AWS Management Console.
   - Go to the **EC2 Dashboard**.

2. **Access Auto Scaling Groups:**
   - In the left-hand menu, under **Auto Scaling**, click on **Auto Scaling Groups**.

#### 8.2 Create a New Auto Scaling Group

1. **Start the Creation Process:**
   - Click **Create Auto Scaling group**.

2. **Configure the Auto Scaling Group:**

   - **Auto Scaling group name:** `wordpress-app-asg`

   - **Launch template:**
     - **Launch template:** Select `wordpress-app-template`.
     - **Version:** Choose `Default (1)`.

3. **Configure Network Settings:**

   - **VPC:** Select `DigitalBoost-VPC`.
   - **Availability Zones and Subnets:**
     - Choose both private subnets:
       - `Private App Subnet AZ1`
       - `Private App Subnet AZ2`

#### 8.3 Configure Group Size and Scaling Policies

1. **Set Group Size:**

   - **Desired capacity:** `2` (ensures that 2 instances are launched, one in each private subnet).
   - **Minimum capacity:** `2`
   - **Maximum capacity:** `2`

2. **Configure Scaling Policies:**
   - For simplicity, leave the scaling policy as **None** to rely on the default instance health check.

3. **Configure Instance Refresh (Optional):**
   - Set up instance refresh settings if you want instances to be replaced automatically with new ones when a new template version is created.

#### 8.4 Configure Load Balancing (Optional but Recommended)

1. **Attach to an Application Load Balancer:**
   - **Load Balancer Type:** Select `Application Load Balancer`.
   - **Target Group:** Choose the appropriate target group, like `wordpress-targets`.

#### 8.5 Advanced Configuration (Optional)

1. **Additional Settings:**
   - Configure advanced settings such as health check grace period, monitoring, notifications, tags, etc., as per your requirements.

#### 8.6 Review and Create

1. **Review Settings:**
   - Double-check all configurations to ensure accuracy.

2. **Create Auto Scaling Group:**
   - Click **Create Auto Scaling group**.

#### 8.7 Verify Auto Scaling Group

1. **Monitor the Provisioning Process:**
   - Monitor the Auto Scaling group as it provisions new instances.
   - Check the **Instances** section in the EC2 Dashboard for the status of new instances.

2. **Confirm Instances:**
   - Verify that there are four instances running:
     - Two instances from the Auto Scaling group.
     - One Bastion host.
     - One temporary WordPress instance from Step 3.

#### 8.8 Terminate Temporary WordPress Instance

1. **Locate the Temporary Instance:**
   - Identify the temporary WordPress instance created earlier.

2. **Terminate the Instance:**
   - Select the temporary instance.
   - Click **Actions** > **Instance State** > **Terminate**.
   - Confirm the termination.

### Summary

1. **Create Auto Scaling Group:**
   - Name: `wordpress-app-asg`
   - Use launch template `wordpress-app-template`.
   - Configure network settings for private subnets in `AZ1` and `AZ2`.
   - Set desired, minimum, and maximum capacity to `2`.

2. **Configure Scaling Policies:**
   - Opt not to set additional scaling policies for simplicity.

3. **Attach Load Balancer (Optional):**
   - Connect the Application Load Balancer for efficient traffic distribution.

4. **Verify and Monitor:**
   - Ensure Auto Scaling group provisions new WordPress instances.
   - Confirm a total of four instances in the EC2 dashboard.
   - Terminate the temporary WordPress instance.

This setup guarantees high availability and scalability for your WordPress instances, adapting to demand effectively.

### 9 — Create an Application Load Balancer and Connect with the Auto Scaling Group

#### 9.1 Create a Target Group

1. **Navigate to the EC2 Dashboard:**
   - Log in to the AWS Management Console.
   - Go to the **EC2 Dashboard**.

2. **Create a Target Group:**
   - In the left-hand menu, under **Load Balancing**, click on **Target Groups**.
   - Click **Create target group**.

3. **Configure the Target Group:**
   - **Choose a target type:** Select `Instances`.
   - **Target group name:** Enter `wordpress-target-grp`.
   - **Protocol:** Select `HTTP`.
   - **Port:** Enter `80`.
   - **VPC:** Select `DigitalBoost-VPC`.

4. **Health Checks:**
   - **Protocol:** HTTP
   - **Path:** `/` (default root path of WordPress)
   - Leave other settings as default unless you have specific health check requirements.

5. **Register Targets:**
   - **Do not register any targets at this point**. Click **Next**, then **Create target group**.

#### 9.2 Create a Security Group for the Load Balancer

1. **Navigate to Security Groups:**
   - In the left-hand menu, under **Network & Security**, click on **Security Groups**.
   - Click **Create security group**.

2. **Configure the Security Group:**
   - **Security group name:** `ALB Security Group`.
   - **Description:** Security group for ALB allowing HTTP and HTTPS traffic.
   - **VPC:** Select `DigitalBoost-VPC`.

3. **Inbound Rules:**
   - Add the following inbound rules:
     - **Type:** HTTP
       - **Protocol:** TCP
       - **Port Range:** 80
       - **Source:** `0.0.0.0/0` and `::/0`
     - **Type:** HTTPS
       - **Protocol:** TCP
       - **Port Range:** 443
       - **Source:** `0.0.0.0/0` and `::/0`

4. **Create Security Group:**
   - Click **Create security group**.

#### 9.3 Create an Application Load Balancer (ALB)

1. **Navigate to Load Balancers:**
   - In the left-hand menu, under **Load Balancing**, click on **Load Balancers**.
   - Click **Create Load Balancer**.

2. **Select Load Balancer Type:**
   - Choose **Application Load Balancer**.

3. **Configure the Load Balancer:**
   - **Name:** Enter `wordpress-alb`.
   - **Scheme:** Select `Internet-facing`.
   - **IP address type:** Select `ipv4`.

4. **Network Mapping:**
   - **VPC:** Select `DigitalBoost-VPC`.
   - **Availability Zones:** Select both public subnets:
     - `Public Subnet AZ1`
     - `Public Subnet AZ2`

5. **Security Groups:**
   - **Select a security group:** Choose the newly created `ALB-SG`.

6. **Listeners:**
   - **Protocol:** HTTP
   - **Port:** 80
   - Click **Add listener**, and configure another listener for HTTPS:
     - **Protocol:** HTTPS
     - **Port:** 443

7. **Default Action:**
   - For each listener, select the target group `wordpress-target-grp` created earlier.

8. **Review and Create:**
   - Review all configurations to ensure they are correct.
   - Click **Create load balancer**.

#### 9.4 Attach Auto Scaling Group to the Load Balancer

1. **Navigate to Auto Scaling Groups:**
   - In the left-hand menu, under **Auto Scaling**, click on **Auto Scaling Groups**.

2. **Select Auto Scaling Group:**
   - Choose the `wordpress-app-asg`.

3. **Edit Load Balancer:**
   - Click **Edit** under **Load balancing**.
   - **Target groups:** Select `wordpress-target-grp`.
   - Save the changes.

#### 9.5 Update Security Group for Auto Scaling Group

1. **Navigate to Security Groups:**
   - In the left-hand menu, under **Network & Security**, click on **Security Groups**.
   - Select the security group associated with the Auto Scaling group (`Webserver Security Group`).

2. **Update Inbound Rules:**
   - Add the following inbound rules:
     - **Type:** HTTP
       - **Protocol:** TCP
       - **Port Range:** 80
       - **Source:** `ALB Security GroupG`
     - **Type:** HTTPS
       - **Protocol:** TCP
       - **Port Range:** 443
       - **Source:** `ALB-SG`

#### 9.6 Test the Load Balancer

1. **Get the DNS Name:**
   - Navigate to the **Load Balancers** section in the EC2 Dashboard.
   - Select `wordpress-alb` and copy its DNS name.

2. **Verify Access:**
   - Open a web browser and enter the DNS name.
   - You should see your WordPress site indicating that the ALB is properly routing traffic to the Auto Scaling group.

### Summary

1. **Create Target Group:**
   - Name: `wordpress-target-grp`
   - Type: `Instances`
   - Protocol: `HTTP`
   - Port: `80`
   - VPC: `DigitalBoost-VPC`

2. **Create Security Group:**
   - Name: `ALB Security Group
   - Inbound Rules: Allow HTTP (80) and HTTPS (443) traffic from all sources.

3. **Create Application Load Balancer:**
   - Name: `wordpress-alb`
   - Scheme: `Internet-facing`
   - VPC: `DigitalBoost-VPC`
   - Security Group: `ALB Security Group`
   - Listeners: HTTP (80) and HTTPS (443)
   - Target Group: `wordpress-target-grp`

4. **Attach Auto Scaling Group:**
   - Attach `wordpress-target-grp` to `wordpress-app-asg`.

5. **Update Security Group:**
   - Allow HTTP and HTTPS traffic from `ALB Security Group` to the Auto Scaling group's security group.

6. **Test Load Balancer:**
   - Verify access to WordPress using the Load Balancer’s DNS name.

This setup ensures your WordPress application is highly available and scalable, with traffic efficiently routed through the Application Load Balancer to the Auto Scaling group.

### 10. Integration with Auto Scaling

The objective is to correctly integrate the Load Balancer with the Auto Scaling group to ensure high availability and automatic scaling of your WordPress application.

#### Step 1: Create Launch Configuration or Template

1. **Navigate to EC2 Dashboard:**
   - Log in to the AWS Management Console.
   - Go to the **EC2 Dashboard**.

2. **Create Launch Configuration/Template:**
   - Go to **Launch Configurations** or **Launch Templates** under the EC2 menu.
   - Click **Create launch configuration** or **Create launch template**.

3. **Configure the Launch Configuration/Template:**
   - **AMI:** Choose the AMI used for your WordPress instances (e.g., Amazon Linux 2).
   - **Instance Type:** Select `t2.micro` (eligible for Free Tier).
   - **Key Pair:** Select your key pair (e.g., `wp-app-key`).
   - **Security Group:** Attach the `Webserver Security Group` which allows necessary inbound traffic for WordPress.

4. **Launch Template Settings:**
   - **Launch template name:** `wordpress-app-template`
   - **Template version:** `1`
   - **Instance Configuration:** Use your selected AMI, instance type, key pair, and security group.

5. **Save the Configuration/Template:**
   - Review your settings and save the launch configuration/template.

#### Step 2: Create Auto Scaling Group

1. **Navigate to Auto Scaling Groups:**
   - In the EC2 Dashboard, click on **Auto Scaling Groups** under the **Auto Scaling** section.

2. **Create Auto Scaling Group:**
   - Click **Create Auto Scaling group**.
   - **Name:** Enter `wordpress-asg`.

3. **Select Launch Configuration/Template:**
   - Choose the launch configuration or template created in Step 1.

4. **Network Configuration:**
   - **VPC:** Select `DigitalBoost-VPC`.
   - **Subnets:** Choose the private subnets (e.g., `Private App Subnet AZ1` and `Private App Subnet AZ2`).

#### Step 3: Attach Load Balancer

1. **Load Balancer Attachment:**
   - In the Auto Scaling group configuration, navigate to the **Load Balancing** section.
   - **Load Balancer:** Select your ALB (`wordpress-alb`).
   - **Target Group:** Select the target group (`wordpress-target-grp`).

#### Step 4: Configure Scaling Policies

1. **Set Desired, Minimum, and Maximum Capacity:**
   - **Desired Capacity:** `2`
   - **Minimum Capacity:** `2`
   - **Maximum Capacity:** `4`

2. **Scaling Policies:**
   - Set up policies based on metrics like CPU utilization:
     - **Policy Type:** Target Tracking Scaling.
     - **Metric Type:** Average CPU Utilization.
     - **Target Value:** For example, 50%.

3. **Additional Settings:**
   - Configure cooldown periods and other advanced settings as needed.

#### Step 5: Review and Create

1. **Review Settings:**
   - Review all configurations for correctness.

2. **Create Auto Scaling Group:**
   - Click **Create Auto Scaling group**.

#### Verification

1. **Monitor Instances:**
   - Ensure the Auto Scaling group launches instances as specified.
   - Check instances are registered with the ALB target group and receive traffic.

2. **Health Checks:**
   - Verify instance health according to target group checks.

3. **Traffic Distribution:**
   - Confirm the Load Balancer distributes traffic correctly.

#### Final Steps

1. **Configure Load Balancer Listener:**
   - Add listener rules for HTTP and HTTPS traffic to the target group.

2. **Testing:**
   - Test the Load Balancer's DNS in a browser to access WordPress.
   - Ensure traffic balances across instances and scaling works based on load.

### Conclusion

By following these steps, you've integrated an Application Load Balancer with an Auto Scaling group for your WordPress app on AWS, ensuring high availability, security, and scalability. Regularly monitor for optimal performance and troubleshoot any issues.

### 11 — Create a CloudFront Distribution for the WordPress website hosted on AWS

#### Step-by-Step Guide

1. **Navigate to CloudFront in the AWS Console:**
   - Log in to the AWS Management Console.
   - In the search bar, type "CloudFront" and select it from the dropdown.

2. **Create a New CloudFront Distribution:**
   - Click on the "Create Distribution" button.
   - Choose "Web" for the delivery method.

3. **Configure the Origin Settings:**
   - **Origin Domain Name:** Select your Application Load Balancer (ALB) from the dropdown list.
   - **Origin Path:** Leave this blank unless you have a specific subdirectory.
   - **Origin ID:** This will be auto-filled. You can leave it as is.

4. **Configure Default Cache Behavior Settings:**
   - **Viewer Protocol Policy:** Choose either "Redirect HTTP to HTTPS" (if you have an SSL certificate) or "HTTP and HTTPS" (if you don't want to enforce HTTPS yet).
   - **Allowed HTTP Methods:** Select "GET, HEAD, OPTIONS, PUT, POST, PATCH, DELETE". This ensures that all necessary methods for WordPress functionality are enabled.
   - **Cache Based on Selected Request Headers:** Choose "All" to ensure dynamic content from WordPress is not cached inappropriately.
   - **Forward Cookies:** Select "All" to ensure proper functionality of your WordPress site.

5. **Configure Distribution Settings:**
   - **Price Class:** Choose a price class based on your needs (e.g., "Use Only U.S., Canada, and Europe" for lower cost).
   - **Alternate Domain Names (CNAMEs):** If you want to use your custom domain, enter it here.
   - **SSL Certificate:** If you have chosen to redirect HTTP to HTTPS, select "Custom SSL Certificate" and choose your certificate from ACM (AWS Certificate Manager). Otherwise, you can use the default CloudFront certificate.

6. **Configure Default Root Object:**
   - **Default Root Object:** Enter `index.php` to ensure that CloudFront directs to the correct root object of your WordPress site.

7. **Review and Create Distribution:**
   - Review all the settings to ensure they are correct.
   - Click on the "Create Distribution" button.

8. **Configure Route 53 (Optional):**
   - If you want to use a custom domain, navigate to Route 53 in the AWS Management Console.
   - Select your hosted zone.
   - Create a new record set:
     - **Name:** Enter your custom domain name (e.g., `www.yourdomain.com`).
     - **Type:** Select "A - IPv4 address".
     - **Alias:** Choose "Yes".
     - **Alias Target:** Select your CloudFront distribution from the list.

9. **Wait for Distribution to Deploy:**
   - It can take a few minutes for the CloudFront distribution to be fully deployed and propagate across the edge locations.
   - Once deployed, you will receive a CloudFront distribution domain name (e.g., `d1234567890.cloudfront.net`).

10. **Test Your Setup:**
    - Open a web browser and navigate to the CloudFront distribution domain name or your custom domain if configured.
    - You should see your WordPress site, and if configured for HTTPS, ensure that the site is served over HTTPS.

### Summary

By following these steps, you will have set up a CloudFront distribution that caches and serves your WordPress site content, potentially improving load times and reliability for your users. If you chose to redirect HTTP to HTTPS, ensure you have an SSL certificate in place to secure your site. Finally, using Route 53 allows you to seamlessly integrate your custom domain with CloudFront.

### 12 — Register CloudFront Distribution with Route 53

1. **Navigate to Route 53 in the AWS Console:**
   - Log in to the AWS Management Console.
   - In the search bar, type "Route 53" and select it from the dropdown.

2. **Choose Your Domain Management Option:**
   - **Register a Domain with Route 53:**
     - If you don't have a domain, you can register one through Route 53.
     - Navigate to **Registered Domains** and click on **Register Domain**.
     - Search for the desired domain name, select it, and follow the prompts to complete the registration.
   - **Using a Third-Party Domain:**
     - If you already have a domain registered with a third-party provider, ensure that you have access to the domain's DNS settings.

3. **Create a Hosted Zone in Route 53:**
   - In Route 53, go to **Hosted Zones** and click on **Create Hosted Zone**.
   - **Domain Name:** Enter your domain name (e.g., `yourdomain.com`).
   - **Type:** Select **Public Hosted Zone**.
   - **Comment:** Optional, but you can add a description for clarity.
   - Click on **Create Hosted Zone**.

4. **Update DNS Settings with Third-Party Provider (if applicable):**
   - If your domain is registered with a third-party provider, update the nameservers to those provided by Route 53.
   - In your third-party provider's DNS settings, replace the existing nameservers with the four Route 53 nameservers listed in the Hosted Zone details.

5. **Create an A Record for CloudFront Distribution:**
   - Select your newly created Hosted Zone.
   - Click on **Create Record** and choose the following settings:
     - **Record Name:** Leave it blank or enter `www` if you want to use `www.yourdomain.com`.
     - **Record Type:** Choose **A - IPv4 address**.
     - **Alias:** Select **Yes**.
     - **Alias Target:** Choose your CloudFront distribution from the list.
     - **Routing Policy:** Choose **Simple**.
     - **Evaluate Target Health:** Select **Yes** or **No** based on your preference.

6. **Create Additional Records (Optional):**
   - If you want both `yourdomain.com` and `www.yourdomain.com` to point to your CloudFront distribution, create another A record for the root domain:
     - **Record Name:** Leave it blank.
     - **Record Type:** Choose **A - IPv4 address**.
     - **Alias:** Select **Yes**.
     - **Alias Target:** Choose your CloudFront distribution.
     - **Routing Policy:** Choose **Simple**.
     - **Evaluate Target Health:** Select **Yes** or **No**.

7. **Save the Records:**
   - Click on **Create Records** to save your new DNS settings.

8. **Verify the Setup:**
   - Allow some time for DNS propagation (this can take a few minutes to several hours).
   - Open a web browser and navigate to your domain (e.g., `yourdomain.com` or `www.yourdomain.com`).
   - You should see your WordPress site being served through CloudFront.

### Summary

By following these steps, you will have successfully registered your CloudFront distribution with Route 53. This integration allows you to manage your domain's DNS settings within AWS, providing seamless routing of traffic to your WordPress site via CloudFront. Make sure to verify that your DNS records are correctly pointing to the CloudFront distribution and that your site is accessible through your custom domain.
