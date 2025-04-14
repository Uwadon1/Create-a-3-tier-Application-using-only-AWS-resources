
# AWS 3-Tier Architecture Implementation

A comprehensive guide to building a secure 3-Tier Architecture application on AWS, separating presentation, application, and database layers with proper security controls and custom domain integration via Route 53.


## Architecture Overview

### Component Breakdown

| Tier          | Components                                                                 |
|---------------|----------------------------------------------------------------------------|
| Presentation  | - EC2 in Public Subnet<br>- IAM Roles<br>- Auto Scaling<br>- Internet-Facing ALB |
| Application   | - EC2 in Private Subnet<br>- IAM Roles<br>- Internal Application Load Balancer |
| Database      | - RDS MySQL in Private Subnet                                              |
| Security      | - VPC with Public/Private Subnets<br>- Security Groups<br>- NAT Gateway    |

## Infrastructure Setup

### Step 1: VPC Creation

**Create Custom VPC:**
1. Navigate to VPC service → "Create VPC"
2. Select "VPC and more" option
3. Configuration:
   - Name: `my-demo-3-tier-vpc`
   - IPv4 CIDR: `192.168.0.0/22`
   - Availability Zones: 2
   - Public Subnets: 2
   - Private Subnets: 4 (2 for app, 2 for DB)
   - NAT Gateway: 1

**Subnet Naming:**
- Rename private subnets:
  - App-1, App-2 (application tier)
  - Db-1, Db-2 (database tier)

**Security Groups Setup:**

| SG Name            | Allowed Traffic                          | Ports  | Source                     |
|--------------------|------------------------------------------|--------|----------------------------|
| Web-tier-ALB-SG    | HTTP from internet                       | 80     | 0.0.0.0/0                  |
| Web-tier-SG        | HTTP from ALB + VPC                      | 80     | Web-tier-ALB-SG + VPC CIDR |
| App-tier-SG        | App traffic from VPC                     | 4000   | VPC CIDR                   |
| App-tier-ALB-SG    | Internal HTTP traffic                    | 80     | VPC CIDR                   |
| RDS-SG             | MySQL from VPC                           | 3306   | VPC CIDR                   |

---

### Step 2: S3 Bucket and IAM Role Setup

**S3 Bucket Creation:**
```bash
git clone https://github.com/Uwadon1/3TierArchitectureApp.git
```
1. Create bucket (e.g., `demo-3tier-bucket-007`)
2. Upload cloned application code

**IAM Role for EC2 SSM Access:**
1. Create role for EC2 service
2. Attach `AmazonEC2RoleforSSM` policy
3. Name: `Demo-EC2-Role`

---

### Step 3: Database Configuration

**RDS MySQL Setup:**
1. Create DB subnet group:
   - Include Db-1, Db-2 subnets
2. Launch RDS instance:
   - Engine: MySQL 8.0.35
   - Instance: db.t3.micro (Free Tier)
   - Credentials:
     - DB identifier: `database-1`
     - Master username: `admin`
     - Custom password
   - Connectivity: Custom VPC with RDS-SG
   - Disable enhanced monitoring/backups

**Database Initialization:**
```sql
-- Connect to RDS
mysql -h <RDS-Endpoint> -u admin -p

-- Create database and tables
CREATE DATABASE webappdb;
USE webappdb;

CREATE TABLE IF NOT EXISTS transactions(
  id INT NOT NULL AUTO_INCREMENT, 
  amount DECIMAL(10,2), 
  description VARCHAR(100), 
  PRIMARY KEY(id)
);

-- Sample data
INSERT INTO transactions (amount, description) VALUES ('400', 'groceries');
```

---

### Step 4: Application Tier Setup

**EC2 Instance Configuration:**
1. Launch instance:
   - Name: `App-Tier-Instance`
   - AMI: Amazon Linux 2
   - Type: t2.micro
   - Subnet: App-1 or App-2
   - Security Group: App-tier-SG
   - IAM Role: Demo-EC2-Role

**Application Deployment:**
```bash
# Install dependencies
sudo yum install mysql -y

# Node.js setup
curl -o- https://raw.githubusercontent.com/avizway1/aws_3tier_architecture/main/install.sh | bash
source ~/.bashrc
nvm install 16
nvm use 16
npm install -g pm2

# Deploy app
cd ~/
sudo aws s3 cp s3://<your-bucket>/application-code/app-tier/ app-tier --recursive
cd app-tier
npm install

# Update DbConfig.js with RDS endpoint/credentials

# Start application
pm2 start index.js
pm2 startup
pm2 save

# Verify
curl http://localhost:4000/health
```

**Internal Load Balancer:**
1. Create Target Group:
   - Name: `internal-ALB-TG`
   - Protocol: HTTP:4000
   - Health check: `/health`
2. Create ALB:
   - Name: `App-internal-LB`
   - Scheme: internal
   - Subnets: App-1, App-2
   - Security Group: App-tier-ALB-SG
3. Update nginx.conf:
   ```nginx
   location /api/ {
     proxy_pass http://<INTERNAL-LB-DNS>:80/;
   }
   ```

### Step 5: Web-Tier Setup
Creation of Web tier resources including External Load Balancer: We will create an instance and give it the name Web-Tier-Intance and choose the Linux Linus 2 AMI, choose t2.micro and proceed without selecting a key pair.
We will need to edit the network settings to select our vpc and subnet details. Choose the Web-tier Security Group and ensure to attach the IAM role we created earlier to the advance details, else we won’t be able to connect to this instance. After that we will click on launch instance.

Once the instance is in running state, we will connect to this instance, using the sessions manager.
To connect to this instance, click on the web-tier instance check box and click on connect, and select session manager

What we will do next is to switch to root user and navigate to the /home/ec2-user;
sudo -su 
cd /home/ec2-user


We will run this command to install from the github; 
curl -o- https://raw.githubusercontent.com/avizway1/aws_3tier_architecture/main/install.sh | bash
source ~/.bashrc


The next set of commands we will run are;
nvm install 16 &
nvm use 16

Now, we will copy our application code from our S3 bucket using the command below;
aws s3 cp s3://<S3 Bucker Name>/application-code/web-tier/ web-tier --recursive
Ex: aws s3 cp s3://demo-3tier-bucket-007/application-code/web-tier/ web-tier --recursive


Type ls and you’ll see 'web-tier'
We will move into the web-tier and install NPM

cd web-tier

npm install

npm run build


Now, we will install nginx into our web-tier server using this command;
sudo amazon-linux-extras install nginx1 -y

We are going to update Nginx configuration in the nginx.conf file we just installed, first we will need to move into the nginx directory and remove the file and add the updated nginx.conf in our local machine :
cd /etc/nginx (You are in nginx path)
Pwd to confirm we are now in the nginx directory, then type ls, you will see 'nginx.conf' file


We will use this command to remove the nginx.conf file in my web-tier instance;
sudo rm nginx.conf

This command will copy the nginx.conf file we updated to our bucket directly into the nginx directory in the web-tier instance;
sudo aws s3 cp s3://<S3 Bucker Name>/application-code/nginx.conf .
Ex: sudo aws s3 cp s3://demo-3tier-bucket-007/application-code/nginx.conf .
sudo service nginx restart

chmod -R 755 /home/ec2-user: This command chmod -R 755 /home/ec2-user is used in Linux and Unix-like operating systems (including those used by AWS EC2 instances) to change the permissions of files and directories. Let's break it down:
755: This is the numerical representation of the permissions being set. It defines the read, write, and execute permissions for three categories of users:


The owner (user): The first digit (7) applies to the owner of the file or directory (ec2-user in this context, as it's their home directory).
The group: The second digit (5) applies to the group that the file or directory belongs to.
Others: The third digit (5) applies to all other users on the system.
Let's understand what the digits 7 and 5 mean in terms of permissions:


7 (binary 111): This grants the owner:


Read permission (4): The owner can open and read the contents of files, or list the contents of directories.
Write permission (2): The owner can modify the contents of files, or create, delete, and rename files within directories.
Execute permission (1): The owner can execute files (if they are programs) or enter directories (to access their contents).
5 (binary 101): This grants the group and others:


Read permission (4): They can read files or list directory contents.
No write permission (0): They cannot modify files or the contents of directories.
Execute permission (1): They can execute files or enter directories.



In summary, the command chmod -R 755 /home/ec2-user recursively sets the permissions for the /home/ec2-user directory and all the files and directories within it so that:
The user ec2-user (the owner) has read, write, and execute permissions.
The group associated with these files and directories has read and execute permissions, but not write permission.
All other users on the system have read and execute permissions, but not write permission.

sudo chkconfig nginx on: The command sudo chkconfig nginx on is used on older Linux systems that utilize the SysVinit or Upstart init systems (as opposed to the more modern systemd). It's specifically related to the nginx web server service. The command sudo chkconfig nginx on is used to configure the nginx web server to start automatically whenever the server boots up. It ensures that nginx will be running after a system restart without requiring manual intervention.
The preferred way to manage services on systemd is using the systemctl command. The equivalent command on a systemd-based system would be:
sudo systemctl enable nginx


To check the output of the App, we can check using the Web-Tier-Instance public IP. But before checking let’s open port no 80 with http, Anywhere IPv4, 0.0.0.0/0. Save rules, Now paste the public IP of Web-Tier-Instance in the new tab of your browser.
You will see the app, 

Click on the 3 menu bar to enter the data in the app, click on the Dbdemo to manipulate the data as you wish.


In real life, we don’t give the public IP address of the instance to the end-users nor do we give them the DNS of the load balancer, what we do is create a custom domain by using Route 53 in AWS.

Creation of External/Internet- Load Balancer for Web-Tier
Now we will create an external/Internet-facing Load balancer, this will be in front of the web-tier server, but before then we need to first create a target group and give it the necessary details.
For the instance type, select instances and give your target group a name; external-web-TG and select the protocol to be HTTP and port number 80. We will select our custom VPC and HTTP1 protocol version.
We will leave the health check protocol as default and use / as the health-check path.

Click on next and you will see the option to add an instance to the target group, we will select the web-tier-instance to this target group and include as pending below.

You will see the target group has been created, next we will attach this target group to the load balancer we want to create next.

So we will go to the load balancer section and create a load balancer, choose the application load balancer and give it a name; Web-External-LB. Ensure to select the internet-facing option because we are creating an internet-facing load balancer for our web-tier, that people can access. Next we will select our custom VPC and choose our web-tier subnet group; Web1 and Web2. Then we select the Web-Tier-SG Security Group and attach the web-alb-internal target group to this ALB and click on create.
You will see the prompt Successfully created load balancer: Web-external-LB  and, the ALB status will show ‘active’ You will also notice the DNS name.


Now, we may decide to access our website via the external load balancer DNS name which is not advisable; Web-External-LB-1688633489.us-west-2.elb.amazonaws.com.  We will encounter an error, that error can be corrected by creating a certificate using ACM in AWS. Instead what we need to do is to route the traffic to the custom domain by mapping the external Load Balancer DNS name to the custom domain via Route 53.

### Step 6: SSL Certification and Domain Mapping

We will search for the Route 53 service and navigate to the dashboard.
So, if you already have a domain name, you can transfer it to AWS and do the necessary steps, or if you don’t have you will need to register a new domain name


Once, you’ve configured the necessary verification steps, your domain will be approved and will show in the registered domain section.


To map our domain, we will need to go to the hosted zones and click on the domain name that was just created and click on create record.


In creating the record, we will select A record which will route traffic to an ipv4 address and to other AWS resources.
We will tick the box alias to input our ALB endpoint, resource region and the ALB endpoint.

Now, we need to fix the error we encountered earlier by configuring a TLS/SSL certificate in the Aws Certificate Manager.

We will navigate to the ACM and request a certificate


Now, we will input our domain name and select the DNS validation method, leave the other options as default and click on request. AWS will have to confirm the domain belongs to you, so it might take some time. You will see the prompt that ‘you have successfully requested certificate’ the status will show ‘pending validation’.


We need to create a DNS record in Route 53. Once a CNAME record is created, it would take a while then you will see issued, then the status would show ‘success’.


Now, we will go to the load Balancer’s dashboard and add listener rules to the web-tier ALB, change the protocol port to HTTPS and the port to 443. Click forward to a target group and select the External-web-TG


We will check ‘From ACM’ as the certificate source and select the certificate we just registered, then we will scroll down and click on add.

Having successfully created and added the listener rules, we can copy our domain name now and paste into a browser and our website will come on as shown below. Note: Please ensure to open HTTPS (443) in the Web-Tier-S.G, else, your website won’t open.


Congratulations you have created a 3-tier architecture with Custom Domain using only AWS resources.




So as not to incur any unnecessary costs, remember to delete everything once you’re done.
First, delete both LBs, next delete both TGs, next delete both AMIs, next delete both Snapshots of AMIs, next delete DB, Next delete S3 Bucket, next delete Certificate, next delete Route 53 record, next delete NAT GW, next delete Elastic IP. This came because of NATGW, Finally, delete VPC

