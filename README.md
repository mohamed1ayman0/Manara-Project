# Manara-Project

## 1. VPC and Network Setup

### Create VPC
1. Navigate to VPC Dashboard in AWS Console
2. Click "Create VPC"
3. Configure:
   - Name: `project`
   - IPv4 CIDR block: `10.0.0.0/16`
   - Tenancy: Default
4. Click "Create VPC"

### Create Subnets
1. Navigate to "Subnets" in VPC Dashboard
2. Create public subnets:
   - Name: `public-subnet-1a`
   - VPC: `project`
   - Availability Zone: `us-east-1a`
   - IPv4 CIDR: `10.0.1.0/24`
   - Repeat for `public-subnet-1b` with CIDR `10.0.2.0/24` in AZ `us-east-1b`
3. Create private subnets for EC2:
   - Name: `private-subnet-ec2-1a`
   - VPC: `project`
   - Availability Zone: `us-east-1a`
   - IPv4 CIDR: `10.0.3.0/24`
   - Repeat for `private-subnet-ec2-1b` with CIDR `10.0.4.0/24` in AZ `us-east-1b`
4. Create private subnets for RDS:
   - Name: `private-subnet-rds-1a`
   - VPC: `project`
   - Availability Zone: `us-east-1a`
   - IPv4 CIDR: `10.0.5.0/24`
   - Repeat for `private-subnet-rds-1b` with CIDR `10.0.6.0/24` in AZ `us-east-1b`

### Create Internet Gateway
1. Navigate to "Internet Gateways" in VPC Dashboard
2. Click "Create internet gateway"
3. Name: `project-igw`
4. Click "Create"
5. Select the new IGW and click "Actions" > "Attach to VPC"
6. Select `project-vpc` and click "Attach"

### Create NAT Gateways
1. Navigate to "NAT Gateways" in VPC Dashboard
2. Click "Create NAT gateway"
3. Configure:
   - Name: `nat-gateway-1a`
   - Subnet: `public-subnet-1a`
   - Connectivity type: Public
   - Elastic IP allocation: Click "Allocate Elastic IP"
4. Click "Create NAT gateway"
5. Repeat for `nat-gateway-1b` in `public-subnet-1b`

### Create Route Tables
1. Navigate to "Route Tables" in VPC Dashboard
2. Create public route table:
   - Name: `public-1`
   - VPC: `project-vpc`
   - Add route: Destination `0.0.0.0/0`, Target `project-igw`
   - Associate with public subnets
3. Create private route table for AZ-1a:
   - Name: `private-1a`
   - VPC: `project-vpc`
   - Add route: Destination `0.0.0.0/0`, Target `nat-gateway-1a`
   - Associate with private subnets in AZ-1a
4. Create private route table for AZ-1b:
   - Name: `private-1b`
   - VPC: `project-vpc`
   - Add route: Destination `0.0.0.0/0`, Target `nat-gateway-1b`
   - Associate with private subnets in AZ-1b

## 2. Security Groups Setup

### Create ALB Security Group
1. Navigate to "Security Groups" in VPC Dashboard
2. Click "Create security group"
3. Configure:
   - Name: `alb-sg`
   - Description: "Security group for ALB"
   - VPC: `project-vpc`
   - Inbound rules:
     - Type: HTTP, Source: `0.0.0.0/0`
     - Type: HTTPS, Source: `0.0.0.0/0`
   - Outbound rules: Allow all traffic
4. Click "Create security group"

### Create EC2 Security Group
1. Click "Create security group"
2. Configure:
   - Name: `ec2-sg`
   - Description: "Security group for EC2 instances"
   - VPC: `project-vpc`
   - Inbound rules:
     - Type: HTTP, Source: `alb-sg`
     - Type: SSH, Source: Your IP (for administration)
   - Outbound rules: Allow all traffic
3. Click "Create security group"

### Create RDS Security Group
1. Click "Create security group"
2. Configure:
   - Name: `rds-sg`
   - Description: "Security group for RDS instances"
   - VPC: `project-vpc`
   - Inbound rules:
     - Type: MySQL/Aurora , Source: `ec2-sg`
   - Outbound rules: Allow all traffic
3. Click "Create security group"

## 3. IAM Role Setup

### Create EC2 IAM Role
1. Navigate to IAM Dashboard
2. Click "Roles" > "Create role"
3. Select "AWS service" as the trusted entity and "EC2" as the service
4. Attach policies:
   - AmazonSSMManagedInstanceCore (for Session Manager access)
   - CloudWatchAgentServerPolicy (for CloudWatch metrics)
5. Name: `project-ec2-role0`
6. Click "Create role"

## 4. Create RDS Database (Optional)

### Create DB Subnet Group
1. Navigate to RDS Dashboard
2. Click "Subnet groups" > "Create DB Subnet Group"
3. Configure:
   - Name: `project-db-subnet-group`
   - Description: "Subnet group for project database"
   - VPC: `project-vpc`
   - Add subnets: Select both RDS private subnets
4. Click "Create"

### Create RDS Instance
1. Click "Databases" > "Create database"
2. Choose "Standard create"
3. Select MySQL 
4. Configure:
   - DB instance identifier: `prohect-db`
   - Master username: `admin`
   - Master password: Generate a secure password
   - DB instance class: `db.t2.micro` (for testing)
   - Storage: General Purpose SSD, 20 GB
   - Multi-AZ deployment: Yes
   - VPC: `project-vpc`
   - Subnet group: `project-db-subnet-group`
   - Public access: No
   - VPC security group: `rds-sg`
   - Initial database name: `projectdb`
   - Backup: Enable automated backups
5. Click "Create database"

## 5. Create Launch Template

### Create Launch Template
1. Navigate to EC2 Dashboard
2. Click "Launch Templates" > "Create launch template"
3. Configure:
   - Name: `project-launch-template`
   - Description: "Launch template for web application"
   - AMI: Amazon Linux 2
   - Instance type: t3.micro (for testing)
   - Key pair: Create or select existing
   - Security groups: `ec2-sg`
   - Storage: Default (8 GB gp2)
   - Resource tags: Add Name tag
   - IAM instance profile: `project-ec2-role`
   - User data:
```bash
#!/bin/bash
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd
echo "<h1>Hello from $(hostname -f)</h1>" > /var/www/html/index.html
```
4. Click "Create launch template"

## 6. Create Application Load Balancer

### Create Target Group
1. Navigate to EC2 Dashboard > "Target Groups"
2. Click "Create target group"
3. Configure:
   - Target type: Instances
   - Name: `project-target-group`
   - Protocol: HTTP, Port: 80
   - VPC: `project-vpc`
   - Health check:
     - Protocol: HTTP
     - Path: /
     - Advanced settings: Default
4. Click "Next" > "Create target group"

### Create Load Balancer
1. Navigate to EC2 Dashboard > "Load Balancers"
2. Click "Create load balancer" > "Application Load Balancer"
3. Configure:
   - Name: `project-alb`
   - Scheme: Internet-facing
   - IP address type: IPv4
   - VPC: `project-vpc`
   - Mappings: Select both public subnets
   - Security groups: `alb-sg`
   - Listeners:
     - HTTP on port 80, forward to `project-target-group`
   - Tags: Add Name tag
4. Click "Create load balancer"

## 7. Create Auto Scaling Group

### Create Auto Scaling Group
1. Navigate to EC2 Dashboard > "Auto Scaling Groups"
2. Click "Create Auto Scaling group"
3. Configure:
   - Name: `project-asg`
   - Launch template: `project-launch-template`
   - VPC: `project-vpc`
   - Subnets: Select both EC2 private subnets
   - Load balancing: Attach to existing load balancer
   - Choose target group: `project-target-group`
   - Health checks:
     - Enable ELB health checks
     - Health check grace period: 300 seconds
   - Group size:
     - Desired: 2
     - Minimum: 2
     - Maximum: 4
   - Scaling policies:
     - Target tracking policy
     - Metric: Average CPU utilization
     - Target value: 70%
   - Tags: Add Name tag
4. Click "Create Auto Scaling group"

## 8. Set Up CloudWatch Monitoring

### Create CloudWatch Dashboard
1. Navigate to CloudWatch Dashboard
2. Click "Create dashboard"
3. Name: `project-dashboard`
4. Add widgets for:
   - ALB metrics (RequestCount, HTTPCode_ELB_5XX_Count)
   - EC2 metrics (CPUUtilization, NetworkIn/Out)
   - RDS metrics (if applicable)

### Create CloudWatch Alarms
1. Navigate to CloudWatch Alarms
2. Create alarm for high CPU:
   - Select metric: EC2 > Per-Instance Metrics > CPUUtilization
   - Conditions: Greater than 80% for 5 minutes
   - Notification: Create new SNS topic
3. Create alarm for ALB 5XX errors:
   - Select metric: ApplicationELB > Per-AppELB Metrics > HTTPCode_ELB_5XX_Count
   - Conditions: Greater than 10 for 5 minutes
   - Notification: Use same SNS topic

## 9. Testing the Deployment

1. Wait for the Auto Scaling Group to launch instances
2. Access the ALB DNS name in a web browser
3. Verify that the web page loads correctly
4. Test scaling by:
   - Using a load testing tool to increase CPU usage
   - Manually terminating an instance to verify auto-recovery

## 10. Clean Up 

If you need to clean up resources 

1. Delete Auto Scaling Group
2. Delete Load Balancer and Target Group
3. Delete Launch Template
4. Delete RDS Instance 
5. Delete NAT Gateways
6. Delete Security Groups
7. Delete Subnets and Route Tables
8. Delete Internet Gateway
9. Delete VPC
