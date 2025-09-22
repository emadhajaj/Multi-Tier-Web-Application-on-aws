# Multi-Tier Web Application Architecture on AWS
## Requirements

1. The application will be launched in a VPC with CIDR block `10.0.0.0/16` with:
    - 2 Public subnets in two different AZs (`10.0.10.0/24`, `10.0.20.0/24`).
    - 2 Private subnets in the same AZs (`10.0.100.0/24`, `10.0.200.0/24`).
    
2. Design the VPC security to ensure access control at Layer 4 at the subnet and compute levels.
3. Launch two EBS-backed EC2 instances, one in each AZ above; these will serve as the web and application tiers.
4. Administrators and developers will need programmatic access to AWS services. The application on EC2 instances will also require access to AWS services.
5. The domain name will be registered with AWS.
6. The application will have users across the globe. Ensure the solution has a way of ensuring good performance for remote users as well.
7. Launch an RDS database in the VPC. Ensure failover to another AZ in case of a failure of the primary RDS instance.
8. As traffic increases, the solution must have a component that decouples the web/app tier from the database tier to avoid overwhelming the database.
9. Ensure that the data is encrypted as it is stored.
10. Ensure that the web/app tier is highly available across the two availability zones. Load should be distributed evenly across the web/app instances.




# Designer's Notes


The goal is to have the most cost-effective AWS architecture design using Free Tier benefits and lowest-cost options to meet all specified requirements.

## Services Choice
1. VPC & Networking Foundation
	**Selected Service**: Amazon VPC (Free)
	**Cost Impact**: $0/month (VPC creation is free; only pay for additional services)
	
2. Security (Layer 4 Access Control)
	**Selected Services**:
	- **Security Groups**: Free (managed at instance level)
	- **Network ACLs**: Free (managed at subnet level)
	**Cost Impact**: $0/month
	
3. Compute Layer (Web/App Tier)
	**Selected Service**: EC2 t2.micro instances (Free Tier Eligible)
	- **Configuration**: 2 instances (1 per AZ)
	- **Instance Type**: t2.micro (1 vCPU, 1 GiB RAM)
	- **Storage**: 30 GB gp2 EBS per instance (Free Tier: 30 GB total)
	**Cost Breakdown**:
	- EC2 instances: **$0.00/month** (Free Tier: 750 hours/month covers 2 instances)
	- EBS storage: **$0.00/month** (Free Tier: 30 GB gp2 - using 20 GB total)
	- **Subtotal**: **$0.00/month** (for first 12 months)
	
	**Post-Free Tier Cost** (after 12 months):
	- EC2 instances: 2 × $8.47/month = $16.94/month
	- EBS storage: 2 × 10 GB × $0.10/GB = $2.00/month
	- **Post-Free Tier Subtotal**: $18.94/month
	
1. Load Balancing & High Availability
	**Selected Service**: Application Load Balancer (ALB)
	- **Configuration**: 1 ALB across 2 AZs
	- **Load Balancer Capacity Units (LCUs)**: ~5 LCUs/hour estimated
	
	**Cost**: ~$22.27/month (base) + $5.76/month (5 LCUs) = $28.03/month
	
	**Why ALB over Network Load Balancer**:
	
	- Layer 7 routing capabilities
	- Better for web applications
	- Similar pricing to NLB
	- Built-in SSL termination
	
1. Database Layer
	**Selected Service**: RDS MySQL Multi-AZ (Lowest Cost Option)
	- **Instance**: db.t2.micro (1 vCPU, 1 GiB RAM) - **Free Tier Eligible**
	- **Storage**: 20 GB gp2 (Free Tier eligible)
	- **Multi-AZ**: Enabled for failover requirement
	
	**Cost Breakdown**:
	- **Free Tier (First 12 months)**: $0.00/month
	    - 750 hours db.t2.micro instance (covers Multi-AZ primary)
	    - 20 GB gp2 storage included
	    - Multi-AZ standby is NOT covered by Free Tier
	- **Actual Cost with Multi-AZ**: ~$15.18/month (standby instance cost)
	
	**Post-Free Tier Cost** (after 12 months):
	- db.t2.micro Multi-AZ: $24.82/month
	- Storage (20 GB): $4.60/month
	- **Total**: $29.42/month
	
1. Message Queue (Database Decoupling)
	**Selected Service**: Amazon SQS Standard Queue
	- **Expected Usage**: 1M requests/month
	- **Message Retention**: 14 days (default)
	**Cost**: ~$0.40/month
	
| Service     | Use Case                   | Monthly Cost (1M messages) | Best For                |
| ----------- | -------------------------- | -------------------------- | ----------------------- |
| **SQS**     | **Queue-based decoupling** | **$0.40**                  | **Async processing**    |
| SNS         | Pub/sub messaging          | $0.50                      | Real-time notifications |
| EventBridge | Event routing              | $1.00                      | Complex event patterns  |
	
1. Global Content Delivery
	**Selected Service**: Amazon CloudFront
	- **Origin**: ALB
	- **Expected Traffic**: 100 GB/month outbound
	- **Requests**: ~1M requests/month
	
	**Cost**: $8.50/month (data transfer) + $0.75/month (requests) = $9.25/month
	
1. Domain Registration
	**Selected Service**: Route 53 Domain Registration
	- **Domain**: .com registration
	- **DNS Hosting**: Route 53 hosted zone
	
	**Cost**: $12.00/year ($1.00/month) + $0.50/month (hosted zone) = $1.50/month
	
1. Identity & Access Management
	**Selected Services**:
	- **IAM Users/Groups**: Free (for admin/developer access)
	- **IAM Roles**: Free (for EC2 to AWS services access)
	
	**Cost**: $0/month
	
1. Encryption at Rest
	**Selected Service**: AWS KMS (Customer Managed Keys)
	- **Keys**: 2 keys (RDS + EBS)
	- **Key Usage**: ~1000 requests/month
	
	**Cost**: 2 × $1.00/month + $0.03/month (requests) = $2.03/month

---

## Total Cost Summary

| Year 1 (Free Tier Benefits) | Monthly Cost | Annual Cost |
| --------------------------- | ------------ | ----------- |
| EC2 Instances (2x t2.micro) | $0.00        | $0.00       |
| EBS Storage                 | $0.00        | $0.00       |
| Application Load Balancer   | $28.03       | $336.36     |
| RDS Multi-AZ (db.t2.micro)  | $15.18       | $182.16     |
| SQS                         | $0.40        | $4.80       |
| CloudFront                  | $9.25        | $111.00     |
| Route 53                    | $1.50        | $18.00      |
| KMS                         | $2.03        | $24.36      |
| **YEAR 1 TOTAL**            | **$314.89**  | **$676.68** |

| Post-Free Tier (Year 2+)    | Monthly Cost | Annual Cost   |
| --------------------------- | ------------ | ------------- |
| EC2 Instances (2x t2.micro) | $16.94       | $203.28       |
| EBS Storage                 | $2.00        | $24.00        |
| Application Load Balancer   | $28.03       | $336.36       |
| RDS Multi-AZ (db.t2.micro)  | $29.42       | $353.04       |
| SQS                         | $0.40        | $4.80         |
| CloudFront                  | $9.25        | $111.00       |
| Route 53                    | $1.50        | $18.00        |
| KMS                         | $2.03        | $24.36        |
| **YEAR 2+ TOTAL**           | **$348.07**  | **$1,074.84** |




**Cost Optimization Recommendations**

| Reserved Instances Strategy | Upfront | Monthly Cost | Total Annual | Savings vs On-Demand   |
| --------------------------- | ------- | ------------ | ------------ | ---------------------- |
| **On-Demand**               | $0      | $16.94       | $203.28      | Baseline               |
| **1-Year No Upfront**       | $0      | $11.86       | $142.32      | 30% ($60.96/year)      |
| **1-Year Partial Upfront**  | $51.00  | $10.42       | $176.04      | 13% ($27.24/year)      |
| **1-Year All Upfront**      | $124.00 | $0           | $124.00      | **39% ($79.28/year)**  |
| **3-Year All Upfront**      | $198.00 | $0           | $66.00/year  | **68% ($137.28/year)** |



---

## Architecture Compliance Matrix

| Requirement                | Solution                    | Cost Impact (Year 1) |
| -------------------------- | --------------------------- | -------------------- |
| 1. VPC with subnets        | VPC + 4 subnets in 2 AZs    | $0                   |
| 2. Layer 4 security        | Security Groups + NACLs     | $0                   |
| 3. 2 EC2 instances         | t2.micro in each AZ         | $0 (Free Tier)       |
| 4. Programmatic access     | IAM users, roles            | $0                   |
| 5. Domain registration     | Route 53                    | $1.50/mo             |
| 6. Global performance      | CloudFront CDN              | $9.25/mo             |
| 7. RDS Multi-AZ            | RDS MySQL Multi-AZ          | $15.18/mo            |
| 8. Database decoupling     | Amazon SQS                  | $0.40/mo             |
| 9. Encryption at rest      | KMS + encrypted storage     | $2.03/mo             |
| 10. Load balancing HA      | Application Load Balancer   | $28.03/mo            |












# Implementation


## Networking and Security

### 1. Create VPC and subnets

- Create VPC with CIDR 10.0.0.0/16
- Create Subnets
- Create IGW and attach it to the VPC
- Create Route table and associate it to the subnets
	- Make sure to make the public route table route 0.0.0.0/0 to the IGW
- Create Nat Gateway for the private instance to update and route it

<img width="1196" height="462" alt="image" src="https://github.com/user-attachments/assets/b66f59d1-4edc-499e-ac84-52dbb8711a00" />



### 2. Create the admins IAM Group

We will use the SSM to connect to the instance, this way the SSH port will be closed and the connection will be established by the SSM which is more secure.


Attach this policy to your **admin group** 

> Make sure to change the account id to yours

```json
{
	"Version": "2012-10-17",
	"Statement": [
		{
			"Effect": "Allow",
			
			//The required permissions only for SSH using SSM
			"Action": [
				"ssm:StartSession",
				"ssm:ResumeSession",
				"ssm:TerminateSession"
			],
			
			//Selecting the resources (all EC2)
			"Resource": [
				"arn:aws:ec2:us-east-1:198945929565:instance/*",
				"arn:aws:ssm:us-east-1:*:document/*"
			]
		},
		{
			"Effect": "Allow",
			"Action": [
				"ssm:DescribeSessions",
				"ssm:DescribeInstanceInformation",
				"ssm:GetConnectionStatus"
			],
			"Resource": "*"
		},
		{
			"Effect": "Allow",
			
			//we will need that in order to see the instances in the consle
			"Action": [
				"ec2:DescribeInstances",
				"ec2:DescribeTags",
				"ec2:DescribeSecurityGroups"
			],
			"Resource": "*"
		}
	]
}
```



 
### 3. Create your IAM users 

- add them to the group we created
- enable **MFA** for more security
- Make sure to save the credentials, do not make my mistake 😅



### 4. Create the **EC2 role** (instance profile) for SSM

The instance needs a role so the **SSM Agent** can register. Make sure to reboot the instances after attaching the role.

1. **Create role**
	- Attach managed policy **AmazonSSMManagedInstanceCore**.
    
2. **(Optional) Add CloudWatch logging** later if you want SSM session logs.

---

### 5. Create Security Groups (Give minimal access)


**ALB-WEB-SG (on the ALB)**
- Inbound: 80/443 from `0.0.0.0/0`
- Outbound: 80/443 → **Web-SG** 


**ALB-APP-SG (on the ALB)**
- Inbound: 80/443 from **Web-SG** 
- Outbound: TCP 8080 → **App-SG**

**Web-SG (on public web EC2s)** 
- Inbound:
    - 80 from **ALB-WEB-SG**
    - 433 from 0.0.0.0/0
    
- Outbound:
    - HTTP 80 → **ALB-App-SG**
    - HTTPS 443 → `0.0.0.0/0` for SSM
	- **Do NOT** allow 3306 outbound to DB anymore


**App-SG (on private app EC2s)** 
- Inbound:
    - TCP 8080 from **ALB-APP-SG**
    - HTTPS 433 from 0.0.0.0/0 for SSM
    
- Outbound:
    - MySQL 3306 → **RDS-SG** 
    - HTTPS 443 → `0.0.0.0/0` or to VPC endpoints for SSM/updates


**RDS-SG (on the RDS instance)**
- Inbound:
    - MySQL 3306 from **App-SG** only 
    
- Outbound:
    - Leave default 


**Endpoint-SG
- inbound:
	- TCP 433 from Web-SG
	- TCP 433 from App-SG
- Outbound:
	- Default



> If you later add **RDS Proxy**, swap the last hop:  
> App-SG → **RDS-Proxy-SG** (3306), and **RDS-SG** only trusts **RDS-Proxy-SG**.





## Computing


### 6. Create App Tier Launch Template

- **Name**: `app-tier-template`
- **Security groups**: Select **App-SG**
- **Auto-assign public IP**: Disable (private subnets)
- **IAM instance profile**: Select your EC2-SSM-Role
- **User data**:

```bash
#!/bin/bash
yum update -y
yum install -y python3 python3-pip
pip3 install flask pymysql

# Create a simple Flask app
cat <<EOF > /home/ec2-user/app.py
from flask import Flask, jsonify
import pymysql
import os
from datetime import datetime

app = Flask(__name__)

@app.route('/')
def home():
    return jsonify({
        'message': 'App Tier Server',
        'instance_id': os.popen('curl -s http://169.254.169.254/latest/meta-data/instance-id').read(),
        'timestamp': datetime.now().isoformat(),
        'status': 'healthy'
    })

@app.route('/health')
def health():
    return jsonify({'status': 'healthy'}), 200

@app.route('/db-test')
def db_test():
    try:
        # Replace with your RDS endpoint when ready
        # connection = pymysql.connect(host='your-rds-endpoint',
        #                            user='admin',
        #                            password='your-password',
        #                            database='testdb')
        return jsonify({'database': 'connection ready', 'status': 'configured'})
    except Exception as e:
        return jsonify({'database': 'not connected', 'error': str(e)}), 500

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8080)
EOF

# Create systemd service for the app
cat <<EOF > /etc/systemd/system/appserver.service
[Unit]
Description=Flask App Server
After=network.target

[Service]
Type=simple
User=ec2-user
WorkingDirectory=/home/ec2-user
Environment=PATH=/usr/local/bin:/usr/bin:/bin
ExecStart=/usr/bin/python3 /home/ec2-user/app.py
Restart=always

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable appserver
systemctl start appserver
```

---



### 7. Create App Target Group

- **Target group name**: `app-tier-tg`
- **Protocol**: HTTP
- **Port**: 8080
- **VPC**: Select your VPC

**Health checks:**
- **Health check path**: `/health`
- **Port**: 8080
- **Success codes**: 200

---





### 8. Create App Application Load Balancer (Internal)

- **Name**: `app-alb`
- **Scheme**: Internal
- **IP address type**: IPv4

**Network mapping:**
- **VPC**: Select your VPC
- **Mappings**:
    - Select both private subnets (10.0.100.0/24, 10.0.200.0/24)

**Security groups:**
- Select **ALB-APP-SG**

**Listeners and routing:**
- **Protocol**: HTTP
- **Port**: 80
- **Default action**: Forward to `app-tier-tg`

---



### 9. Create App Tier Auto Scaling Group

- **Name**: `app-tier-asg`
- **Launch template**: Select `app-tier-template`
- **VPC**: Select your VPC
- **Subnets**: Select both private subnets (10.0.100.0/24, 10.0.200.0/24)
- **Load balancing**: Attach to existing load balancer
- **Target groups**: Select `app-tier-tg`
- **Health checks**: ELB health checks enabled
- **Desired capacity**: 1
- **Minimum capacity**: 1
- **Maximum capacity**: 3
- **Target tracking scaling policy**:
    - **Scaling policy name**: `app-cpu-scaling`
    - **Metric type**: Average CPU Utilization
    - **Target value**: 70

---


### 10. Create Web Tier Launch Template

- **Name**: `web-tier-template`
- **Subnet**: Don't specify (ASG will handle)
- **Security groups**: Select **Web-SG** (created earlier)
- **Auto-assign public IP**: Enable
- **IAM instance profile**: Select your EC2-SSM-Role
- **User data**: just make sure to put the right APP-ALB DNS

```bash
#!/bin/bash
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd

# Create a simple web page that identifies the instance
TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
INSTANCE_ID=$(curl -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/instance-id)
AZ=$(curl -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/placement/availability-zone)

cat <<EOF > /var/www/html/index.html
<!DOCTYPE html>
<html>
<head><title>Web Server</title></head>
<body>
<h1>Web Tier Server</h1>
<p>Instance ID: $INSTANCE_ID</p>
<p>Availability Zone: $AZ</p>
<p>Server Time: $(date)</p>
<hr>
<p><a href="/app">Connect to App Tier</a></p>
</body>
</html>
EOF

# Configure web server to proxy to app tier (will be configured later)
cat <<EOF > /var/www/html/app.html
<!DOCTYPE html>
<html>
<head><title>App Connection</title></head>
<body>
<h1>Connecting to App Tier...</h1>
<p>This will connect to the internal App Load Balancer</p>
</body>
</html>
EOF
```



### 11. Create Web Target Group

- **Choose target type**: Instances
- **Target group name**: `web-tier-tg`
- **Protocol**: HTTP
- **Port**: 80
- **VPC**: Select your VPC
- **Protocol version**: HTTP1

**Health checks:**
- **Health check protocol**: HTTP
- **Health check path**: `/`
- **Port**: Traffic port
- **Healthy threshold**: 2
- **Unhealthy threshold**: 2
- **Timeout**: 5 seconds
- **Interval**: 30 seconds
- **Success codes**: 200

**Register targets**: Skip for now (ASG will handle)


### 12. Create Web Application Load Balancer (Internet-facing)

- **Name**: `web-alb`
- **Scheme**: Internet-facing
- **IP address type**: IPv4

**Network mapping:**
- **VPC**: Select your VPC
- **Mappings**:
    - Select both public subnets (10.0.10.0/24, 10.0.20.0/24)

**Security groups:**
- Remove default
- Select **ALB-WEB-SG**

**Listeners and routing:**
- **Protocol**: HTTP
- **Port**: 80
- **Default action**: Forward to `web-tier-tg`



### 13. Create Web Tier Auto Scaling Group


- **Name**: `web-tier-asg`
- **Launch template**: Select `web-tier-template`
- **Version**: Latest
- **VPC**: Select your VPC
- **Availability Zones and subnets**:
    - Select both public subnets (10.0.10.0/24, 10.0.20.0/24)
- **Load balancing**: Attach to an existing load balancer
- **Existing load balancer target groups**: Select `web-tier-tg`
- **Health checks**:
    - **ELB health checks**: Enable
    - **Health check grace period**: 300 seconds
- **Desired capacity**: 1
- **Minimum capacity**: 1
- **Maximum capacity**: 3
- **Target tracking scaling policy**:
    - **Scaling policy name**: `web-cpu-scaling`
    - **Metric type**: Average CPU Utilization
    - **Target value**: 70
    - **Instances need**: 300 seconds warm up




---


### 14. Test Web ALB

1. Get the Web ALB DNS name from the console
2. Access `http://your-web-alb-dns-name` in browser
   <img width="608" height="399" alt="image" src="https://github.com/user-attachments/assets/7a106df7-21f1-4fcf-8d76-6e0bef35fb68" />

3. Test the buttons in the End



---

### 15CloudFront Integration for global performance:

**Console: CloudFront → Create Distribution**

- **Origin domain**: Your Web ALB DNS name
- **Origin protocol policy**: HTTP only (or HTTPS if configured)
- **Viewer protocol policy**: Redirect HTTP to HTTPS
- **Allowed HTTP methods**: GET, HEAD, OPTIONS, PUT, POST, PATCH, DELETE
- **Cache policy**: Managed-CachingDisabled (for dynamic content)

---


### 16. Create VPC Endpoints for SSM

Your private EC2 instances need to communicate with AWS SSM service. Instead of routing through NAT Gateway, VPC endpoints provide secure, direct access.

**SSM Endpoint:**

- Name: `ssm-endpoint`
- Service category: AWS services
- Service name: `com.amazonaws.us-east-1.ssm`
- VPC: Select your VPC
- Route tables: Select private route tables
- Security group: **Endpoint-SG**

**SSM Messages Endpoint:**
- Name: `ssm-messages-endpoint`
- Service name: `com.amazonaws.us-east-1.ssmmessages`
- Same configuration as above

**EC2 Messages Endpoint:**
- Name: `ec2-messages-endpoint`
- Service name: `com.amazonaws.us-east-1.ec2messages`
- Same configuration as above

> Make sure to put them all in the endpoints security group


---



## Database


### 17. Create RDS Database

**Configuration:**
- Creation method: **Standard create**
- Engine: **MySQL**
- Engine version: **8.0.35** (latest)
- Template: **Free tier** (if eligible)
- **Multi-AZ deployment**: Yes 

**Settings:**
- DB instance identifier: `webapp-db`
- Master username: `admin`
- Master password: `YourSecurePassword123!` (save this!)

**Instance configuration:**
- DB instance class: `db.t3.micro`

**Storage:**
- Storage type: **gp2**
- Allocated storage: **20 GB**
- Enable storage encryption: **Yes** 
- KMS key: **Default**

**Connectivity:**
- VPC: Select your VPC
- Subnet group: **Create new** → Select both private subnets
- Public access: **No**
- VPC security group: **Choose existing** → Select **RDS-SG**

**Database authentication:** Password authentication

**Monitoring:** Enable Enhanced monitoring (optional)

**Backup:**
- Enable automated backups: **Yes**
- Backup retention: **7 days**
- Backup window: Select preferred time

---


### 18. Create Enhanced SQS Setup


```
Web/App Tier → SQS Queue → Lambda (Auto-triggered) → Database
                    ↓
            Dead Letter Queue (DLQ) for failed messages
                    ↓
            CloudWatch Logs + Metrics
```

#### Dead Letter Queue:
- **Queue Name**: `webapp-processing-dlq`
- **Visibility Timeout**: `30 seconds`
- **Message Retention**: `14 days`
- **Max Receive Count**: `3


#### Main Processing Queue:

**Configuration:**
- Type: **Standard queue**
- Name: `webapp-processing-queue`
- Visibility timeout: **30 seconds**
- Message retention period: **14 days**
- Maximum message size: **256 KB**
- Delivery delay: **0 seconds**
- Receive message wait time: **0 seconds** (short polling)

**Access policy:**
- Use default (can restrict later based on needs)

**Encryption:**

- Server-side encryption: **Enabled**
- AWS KMS key: **Amazon SQS key (SSE-SQS)**

---
**Redrive policy:**

- **Dead letter queue**: Select `webapp-processing-dlq`
- **Maximum receives**: **3**

**Tags:**

- Environment: Production
- Project: WebApp
- Component: MessageQueue

---

### 19. Create Lambda Function for SQS Processing

**Basic Information:**
- **Function name**: `Multi-tier-Lambda`
- **Runtime**: Python 3.11
- **Architecture**: x86_64

**Execution role:**
- **Create a new role with basic Lambda permissions**
- **Attach additional policies**:
    - `AWSLambdaVPCAccessExecutionRole` (for VPC access)
    - `AWSLambdaSQSQueueExecutionRole` (for SQS access)
    - Custom policy for RDS access

**Create Custom IAM Policy for Lambda:**

Before creating the function, create this custom policy and attach it to the Lambda execution role:

**Policy Name**: `LambdaSQSRDSPolicy`

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "sqs:ReceiveMessage",
                "sqs:DeleteMessage",
                "sqs:GetQueueAttributes"
            ],
            "Resource": "arn:aws:sqs:us-east-1:YOUR-ACCOUNT-ID:webapp-processing-queue"
        },
        {
            "Effect": "Allow",
            "Action": [
                "rds:DescribeDBInstances",
                "rds:DescribeDBClusters"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "logs:CreateLogGroup",
                "logs:CreateLogStream",
                "logs:PutLogEvents"
            ],
            "Resource": "arn:aws:logs:us-east-1:YOUR-ACCOUNT-ID:*"
        }
    ]
}
```

**Alternative: Use AWS Managed Policy** Instead of the custom policy above, you can attach the AWS managed policy:

- `AWSLambdaSQSQueueExecutionRole`

**Function code:**

```python
import json
import pymysql
import boto3
import os
from datetime import datetime

def lambda_handler(event, context):
    # RDS connection details
    rds_host = os.environ['RDS_ENDPOINT']
    username = os.environ['DB_USERNAME'] 
    password = os.environ['DB_PASSWORD']
    db_name = os.environ['DB_NAME']
    
    connection = None
    
    try:
        # Connect to RDS
        connection = pymysql.connect(
            host=rds_host,
            user=username,
            password=password,
            database=db_name,
            connect_timeout=5
        )
        
        # Process each SQS record
        for record in event['Records']:
            # Parse message body
            message_body = json.loads(record['body'])
            
            # Process the message (example: insert into database)
            with connection.cursor() as cursor:
                sql = "INSERT INTO processed_messages (message_id, content, processed_at) VALUES (%s, %s, %s)"
                cursor.execute(sql, (
                    record['messageId'],
                    json.dumps(message_body),
                    datetime.now()
                ))
                connection.commit()
        
        return {
            'statusCode': 200,
            'body': json.dumps(f'Successfully processed {len(event["Records"])} messages')
        }
        
    except Exception as e:
        print(f"Error processing messages: {str(e)}")
        raise e
        
    finally:
        if connection:
            connection.close()
```

**Environment variables:**

- `RDS_ENDPOINT`: Your RDS endpoint
- `DB_USERNAME`: admin
- `DB_PASSWORD`: YourSecurePassword123!
- `DB_NAME`: your database name

**VPC Configuration:**
- **VPC**: Select your VPC
- **Subnets**: Select both private subnets
- **Security groups**: Create **Lambda-SG**

**Lambda-SG Configuration:**

- **Inbound**: None needed
- **Outbound**:
    - MySQL 3306 → **RDS-SG**
    - HTTPS 443 → `0.0.0.0/0` (for AWS service calls)

**Trigger:**

- **Add trigger**: SQS
- **SQS queue**: The main Queue
- **Batch size**: 10
- **Enable trigger**: Yes

---

### 20. Create Database

**On the Private EC2 create the table, before that get the RDS endpoint first**

```bash
# Update the system
sudo yum update -y

# Install MySQL client
sudo yum install -y mariadb105

# Verify installation
mariadb --version

# Replace YOUR_RDS_ENDPOINT with actual endpoint
mysql -h multi-tier-db.ckr0goam663v.us-east-1.rds.amazonaws.com -u admin -p
```



**Inside the Database**

```sql
-- Create database if it doesn't exist
CREATE DATABASE IF NOT EXISTS webapp;
USE webapp;

-- Create users table
CREATE TABLE users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

-- Create processed_messages table (for SQS processing)
CREATE TABLE processed_messages (
    id INT AUTO_INCREMENT PRIMARY KEY,
    message_id VARCHAR(255) UNIQUE NOT NULL,
    content JSON,
    processed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Create application logs table
CREATE TABLE app_logs (
    id INT AUTO_INCREMENT PRIMARY KEY,
    log_level VARCHAR(10) NOT NULL,
    message TEXT NOT NULL,
    instance_id VARCHAR(20),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Insert sample data
INSERT INTO users (username, email) VALUES 
('admin', 'admin@company.com'),
('developer', 'dev@company.com'),
('testuser', 'test@company.com');
```


### 21. Edit the user data of the Web lunch templates

**Get the actual values of these and replace them**
1. Web ALB DNS
2. App ALB DNS
3. SQS Queue URL
4. RDS Endpoint

```bash
#!/bin/bash
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd

# Get instance metadata
TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
INSTANCE_ID=$(curl -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/instance-id)
AZ=$(curl -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/placement/availability-zone)

# Your actual App ALB DNS name
APP_ALB_DNS="internal-AppALB-1017959364.us-east-1.elb.amazonaws.com"

cat <<EOF > /var/www/html/index.html
<!DOCTYPE html>
<html>
<head>
    <title>Multi-Tier Web Application</title>
    <link href="https://fonts.googleapis.com/css2?family=Poppins:wght@400;600&display=swap" rel="stylesheet">
    <style>
        body {
            font-family: 'Poppins', sans-serif;
            margin: 0;
            padding: 40px;
            background: linear-gradient(to right, #0f2027, #203a43, #2c5364); /* Blue gradient */
            color: #f4faff;
        }

        .container {
            max-width: 900px;
            margin: 0 auto;
            background: #ffffff10; /* translucent white */
            padding: 30px;
            border-radius: 12px;
            box-shadow: 0 8px 20px rgba(0, 0, 0, 0.25);
            backdrop-filter: blur(12px);
            border: 1px solid rgba(255, 255, 255, 0.1);
        }

        .header {
            background-color: #1e3c72;
            color: white;
            padding: 20px;
            border-radius: 8px;
            margin-bottom: 30px;
            text-shadow: 1px 1px 2px #000;
        }

        .header h1 {
            margin: 0 0 10px 0;
        }

        .section {
            margin: 25px 0;
            padding: 20px;
            border: 1px solid #3b5b92;
            background-color: #ffffff0d;
            border-radius: 8px;
        }

        .users-table {
            width: 100%;
            border-collapse: collapse;
            margin-top: 15px;
            background: #ffffff10;
            color: #fff;
        }

        .users-table th, .users-table td {
            border: 1px solid #3b5b92;
            padding: 10px;
            text-align: left;
        }

        .users-table th {
            background-color: #1e3c72;
        }

        button {
            background-color: #007BFF;
            color: white;
            border: none;
            padding: 10px 16px;
            border-radius: 5px;
            cursor: pointer;
            margin: 5px 0;
            transition: background-color 0.3s ease;
            font-family: 'Poppins', sans-serif;
        }

        button:hover {
            background-color: #0056b3;
        }

        input[type="text"], input[type="email"] {
            width: 220px;
            padding: 10px;
            border: 1px solid #4a76a8;
            border-radius: 5px;
            margin: 5px 10px 10px 0;
            background-color: #ffffffcc;
            color: #000;
            font-family: 'Poppins', sans-serif;
        }

        .status {
            padding: 12px;
            border-radius: 6px;
            margin: 15px 0;
            font-weight: bold;
        }

        .success {
            background-color: #d1f4e0;
            color: #155724;
            border: 1px solid #a3e6b5;
        }

        .error {
            background-color: #f8d7da;
            color: #721c24;
            border: 1px solid #f5c6cb;
        }

        .info {
            background-color: #d1ecf1;
            color: #0c5460;
            border: 1px solid #bee5eb;
        }

        h2, h3 {
            color: #dceeff;
        }

        a {
            color: #a3d0ff;
        }

        ::placeholder {
            color: #666;
        }

        footer {
            text-align: center;
            margin-top: 40px;
            padding: 15px;
            color: #a3d0ff;
            font-size: 14px;
            border-top: 1px solid rgba(255,255,255,0.2);
        }
    </style>
</head>
<body>
    <div class="container">
        <div class="header">
            <h1>🌐 Multi-Tier Web Application</h1>
            <p>Instance ID: $INSTANCE_ID | Availability Zone: $AZ</p>
            <p>Server Time: $(date)</p>
        </div>

        <div class="section">
            <h2>🔍 System Health Check</h2>
            <button onclick="checkHealth()">Check App Health</button>
            <button onclick="checkDatabase()">Test Database Connection</button>
            <div id="health-status"></div>
        </div>

        <div class="section">
            <h2>👥 User Management</h2>
            <div>
                <h3>Add New User</h3>
                <input type="text" id="username" placeholder="Username" />
                <input type="email" id="email" placeholder="Email" />
                <button onclick="addUser()">Add User</button>
            </div>
            <div>
                <h3>Current Users</h3>
                <button onclick="loadUsers()">Refresh User List</button>
                <div id="users-container">
                    <p>Click "Refresh User List" to load users from database</p>
                </div>
            </div>
        </div>

        <div class="section">
            <h2>⚙️ Architecture Information</h2>
            <p><strong>Web ALB:</strong> WebALB-1847819320.us-east-1.elb.amazonaws.com</p>
            <p><strong>App ALB:</strong> internal-AppALB-1017959364.us-east-1.elb.amazonaws.com</p>
            <p><strong>Database:</strong> multi-tier-db.ckr0goam663v.us-east-1.rds.amazonaws.com</p>
            <p><strong>SQS Queue:</strong> Multi-Tier-Q</p>
        </div>
    </div>

    <script>
        function showStatus(message, type = 'info') {
            const statusDiv = document.getElementById('health-status');
            statusDiv.innerHTML = '<div class="status ' + type + '">' + message + '</div>';
        }

        function checkHealth() {
            showStatus('Checking application health...', 'info');
            fetch('/health')
                .then(response => response.json())
                .then(data => {
                    showStatus('✅ Application is healthy! Status: ' + data.status, 'success');
                })
                .catch(error => {
                    showStatus('❌ Health check failed: ' + error.message, 'error');
                });
        }

        function checkDatabase() {
            showStatus('Testing database connection...', 'info');
            fetch('/db-test')
                .then(response => response.json())
                .then(data => {
                    if (data.database === 'connected') {
                        showStatus('✅ Database connected! User count: ' + data.user_count, 'success');
                    } else {
                        showStatus('❌ Database connection failed: ' + data.error, 'error');
                    }
                })
                .catch(error => {
                    showStatus('❌ Database test failed: ' + error.message, 'error');
                });
        }

        function loadUsers() {
            const container = document.getElementById('users-container');
            container.innerHTML = '<p>Loading users...</p>';
            
            fetch('/users')
                .then(response => response.json())
                .then(data => {
                    if (data.users && data.users.length > 0) {
                        let tableHTML = '<table class="users-table">';
                        tableHTML += '<tr><th>ID</th><th>Username</th><th>Email</th><th>Created</th><th>Actions</th></tr>';
                        data.users.forEach(user => {
                            const createdDate = new Date(user.created_at).toLocaleDateString();
                            tableHTML += '<tr>';
                            tableHTML += '<td>' + user.id + '</td>';
                            tableHTML += '<td>' + user.username + '</td>';
                            tableHTML += '<td>' + user.email + '</td>';
                            tableHTML += '<td>' + createdDate + '</td>';
                            tableHTML += '<td><button onclick="deleteUser(' + user.id + ', \'' + user.username + '\')">Delete</button></td>';
                            tableHTML += '</tr>';
                        });
                        tableHTML += '</table>';
                        container.innerHTML = tableHTML;
                    } else {
                        container.innerHTML = '<p>No users found in database.</p>';
                    }
                })
                .catch(error => {
                    container.innerHTML = '<div class="status error">Failed to load users: ' + error.message + '</div>';
                });
        }

        function addUser() {
            const username = document.getElementById('username').value;
            const email = document.getElementById('email').value;
            
            if (!username || !email) {
                showStatus('❌ Please enter both username and email', 'error');
                return;
            }
            
            showStatus('Adding user...', 'info');
            
            fetch('/users', {
                method: 'POST',
                headers: {
                    'Content-Type': 'application/json',
                },
                body: JSON.stringify({
                    username: username,
                    email: email
                })
            })
            .then(response => response.json())
            .then(data => {
                if (data.message) {
                    showStatus('✅ ' + data.message, 'success');
                    document.getElementById('username').value = '';
                    document.getElementById('email').value = '';
                    // Auto-refresh users after adding
                    setTimeout(loadUsers, 2000);
                } else {
                    showStatus('❌ Failed to add user: ' + (data.error || 'Unknown error'), 'error');
                }
            })
            .catch(error => {
                showStatus('❌ Error adding user: ' + error.message, 'error');
            });
        }

        function deleteUser(userId, username) {
            if (!confirm('Are you sure you want to delete user "' + username + '"?')) {
                return;
            }
            
            showStatus('Deleting user...', 'info');
            
            fetch('/users/' + userId, {
                method: 'DELETE'
            })
            .then(response => response.json())
            .then(data => {
                if (data.message) {
                    showStatus('✅ ' + data.message, 'success');
                    loadUsers(); // Refresh the list
                } else {
                    showStatus('❌ Failed to delete user: ' + (data.error || 'Unknown error'), 'error');
                }
            })
            .catch(error => {
                showStatus('❌ Error deleting user: ' + error.message, 'error');
            });
        }

        // Load users when page loads
        document.addEventListener('DOMContentLoaded', function() {
            loadUsers();
        });
    </script>
    <footer>
        Made by <strong>EMAD HAJAJ</strong>
    </footer>
</body>
</html>
EOF

# Configure Apache to proxy to app tier
cat <<EOF > /etc/httpd/conf.d/proxy.conf
LoadModule proxy_module modules/mod_proxy.so
LoadModule proxy_http_module modules/mod_proxy_http.so

# Proxy all API endpoints to App ALB
ProxyPass /health http://$APP_ALB_DNS/health
ProxyPassReverse /health http://$APP_ALB_DNS/health

ProxyPass /db-test http://$APP_ALB_DNS/db-test
ProxyPassReverse /db-test http://$APP_ALB_DNS/db-test

ProxyPass /users http://$APP_ALB_DNS/users
ProxyPassReverse /users http://$APP_ALB_DNS/users

# Enable proxy status for debugging
ProxyStatus On
<Location "/proxy-status">
    SetHandler server-status
</Location>
EOF

systemctl restart httpd
```

### 22. Edit the user data of the App lunch templates

**Edit these values**
1. DB_HOST  
2. DB_USER 
3. DB_PASSWORD 
4. DB_NAME

```bash
#!/bin/bash
yum update -y
yum install -y python3 python3-pip mariadb105
pip3 install flask pymysql boto3


# Create a comprehensive Flask application
cat <<EOF > /home/ec2-user/app.py
from flask import Flask, jsonify, request
import pymysql
import boto3
import json
import os
from datetime import datetime

app = Flask(__name__)

DB_HOST = 'multi-tier-db.ckr0goam663v.us-east-1.rds.amazonaws.com' 
DB_USER = 'admin' 
DB_PASSWORD = '' 
DB_NAME = 'webapp'
# Your actual SQS Queue URL
SQS_QUEUE_URL = 'https://sqs.us-east-1.amazonaws.com/198945929565/Multi-Tier-Q'

def get_db_connection():
    try:
        connection = pymysql.connect(
            host=DB_HOST,
            user=DB_USER,
            password=DB_PASSWORD,
            database=DB_NAME,
            connect_timeout=10,
            cursorclass=pymysql.cursors.DictCursor
        )
        return connection
    except Exception as e:
        print(f"Database connection error: {str(e)}")
        raise e

@app.route('/')
def home():
    try:
        instance_id = os.popen('curl -s http://169.254.169.254/latest/meta-data/instance-id').read().strip()
    except:
        instance_id = 'unknown'
    
    return jsonify({
        'message': 'App Tier Server',
        'instance_id': instance_id,
        'timestamp': datetime.now().isoformat(),
        'status': 'healthy',
        'database': DB_HOST,
        'queue': 'Multi-Tier-Q'
    })

@app.route('/health')
def health():
    return jsonify({
        'status': 'healthy',
        'timestamp': datetime.now().isoformat(),
        'service': 'app-tier'
    }), 200

@app.route('/users', methods=['GET'])
def get_users():
    try:
        connection = get_db_connection()
        with connection.cursor() as cursor:
            cursor.execute("SELECT id, username, email, created_at FROM users ORDER BY created_at DESC")
            users = cursor.fetchall()
        connection.close()
        
        # Convert datetime objects to strings for JSON serialization
        for user in users:
            if user['created_at']:
                user['created_at'] = user['created_at'].isoformat()
        
        return jsonify({
            'users': users,
            'count': len(users),
            'timestamp': datetime.now().isoformat()
        })
    except Exception as e:
        return jsonify({
            'error': f'Failed to fetch users: {str(e)}',
            'timestamp': datetime.now().isoformat()
        }), 500

@app.route('/users', methods=['POST'])
def create_user():
    try:
        data = request.get_json()
        
        if not data or 'username' not in data or 'email' not in data:
            return jsonify({'error': 'Username and email are required'}), 400
        
        username = data['username']
        email = data['email']
        
        # Validate input
        if not username.strip() or not email.strip():
            return jsonify({'error': 'Username and email cannot be empty'}), 400
        
        # Insert directly into database (synchronous for immediate feedback)
        connection = get_db_connection()
        with connection.cursor() as cursor:
            # Check if user already exists
            cursor.execute("SELECT id FROM users WHERE username = %s OR email = %s", (username, email))
            if cursor.fetchone():
                connection.close()
                return jsonify({'error': 'User with this username or email already exists'}), 409
            
            # Insert new user
            cursor.execute(
                "INSERT INTO users (username, email, created_at) VALUES (%s, %s, %s)",
                (username, email, datetime.now())
            )
            connection.commit()
            new_user_id = cursor.lastrowid
        
        connection.close()
        
        # Also send to SQS for logging/processing
        try:
            sqs = boto3.client('sqs', region_name='us-east-1')
            message = {
                'action': 'user_created',
                'user_id': new_user_id,
                'username': username,
                'email': email,
                'timestamp': datetime.now().isoformat()
            }
            
            sqs.send_message(
                QueueUrl=SQS_QUEUE_URL,
                MessageBody=json.dumps(message)
            )
        except Exception as sqs_error:
            print(f"SQS notification failed: {str(sqs_error)}")
            # Don't fail the request if SQS fails
        
        return jsonify({
            'message': f'User "{username}" created successfully',
            'user_id': new_user_id,
            'timestamp': datetime.now().isoformat()
        }), 201
        
    except Exception as e:
        return jsonify({
            'error': f'Failed to create user: {str(e)}',
            'timestamp': datetime.now().isoformat()
        }), 500

@app.route('/users/<int:user_id>', methods=['DELETE'])
def delete_user(user_id):
    try:
        connection = get_db_connection()
        with connection.cursor() as cursor:
            # Check if user exists
            cursor.execute("SELECT username FROM users WHERE id = %s", (user_id,))
            user = cursor.fetchone()
            
            if not user:
                connection.close()
                return jsonify({'error': 'User not found'}), 404
            
            username = user['username']
            
            # Delete user
            cursor.execute("DELETE FROM users WHERE id = %s", (user_id,))
            connection.commit()
        
        connection.close()
        
        # Send to SQS for logging
        try:
            sqs = boto3.client('sqs', region_name='us-east-1')
            message = {
                'action': 'user_deleted',
                'user_id': user_id,
                'username': username,
                'timestamp': datetime.now().isoformat()
            }
            
            sqs.send_message(
                QueueUrl=SQS_QUEUE_URL,
                MessageBody=json.dumps(message)
            )
        except Exception as sqs_error:
            print(f"SQS notification failed: {str(sqs_error)}")
        
        return jsonify({
            'message': f'User "{username}" deleted successfully',
            'timestamp': datetime.now().isoformat()
        }), 200
        
    except Exception as e:
        return jsonify({
            'error': f'Failed to delete user: {str(e)}',
            'timestamp': datetime.now().isoformat()
        }), 500

@app.route('/db-test')
def db_test():
    try:
        connection = get_db_connection()
        with connection.cursor() as cursor:
            cursor.execute("SELECT COUNT(*) as user_count FROM users")
            result = cursor.fetchone()
            
            cursor.execute("SELECT VERSION() as version")
            version_result = cursor.fetchone()
        
        connection.close()
        
        return jsonify({
            'database': 'connected',
            'host': DB_HOST,
            'database_name': DB_NAME,
            'user_count': result['user_count'],
            'mysql_version': version_result['version'],
            'timestamp': datetime.now().isoformat()
        })
    except Exception as e:
        return jsonify({
            'database': 'connection failed',
            'error': str(e),
            'host': DB_HOST,
            'timestamp': datetime.now().isoformat()
        }), 500

# Error handlers
@app.errorhandler(404)
def not_found(error):
    return jsonify({'error': 'Endpoint not found'}), 404

@app.errorhandler(500)
def internal_error(error):
    return jsonify({'error': 'Internal server error'}), 500

if __name__ == '__main__':
    print(f"Starting Flask app on port 8080...")
    print(f"Database: {DB_HOST}")
    print(f"SQS Queue: {SQS_QUEUE_URL}")
    app.run(host='0.0.0.0', port=8080, debug=False)
EOF

# Create systemd service for the app
cat <<EOF > /etc/systemd/system/appserver.service
[Unit]
Description=Flask App Server
After=network.target

[Service]
Type=simple
User=ec2-user
WorkingDirectory=/home/ec2-user
Environment=PATH=/usr/local/bin:/usr/bin:/bin
Environment=PYTHONPATH=/usr/local/lib/python3.9/site-packages
ExecStart=/usr/bin/python3 /home/ec2-user/app.py
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF

# Set proper permissions
chown ec2-user:ec2-user /home/ec2-user/app.py
chmod +x /home/ec2-user/app.py

# Enable and start the service
systemctl daemon-reload
systemctl enable appserver
systemctl start appserver

# Create a simple health check script
cat <<EOF > /home/ec2-user/health_check.sh
#!/bin/bash
curl -f http://localhost:8080/health > /dev/null 2>&1
if [ \$? -eq 0 ]; then
    echo "App is healthy"
    exit 0
else
    echo "App is unhealthy"
    exit 1
fi
EOF

chmod +x /home/ec2-user/health_check.sh
```



---

## Monitoring

### 23. Monitoring & Logging

1. **Enable CloudWatch Alarms**:
    - Web/App EC2 CPU, memory (via CloudWatch agent), disk usage.
    - RDS free storage, CPU, connections.
    - ALB 4xx/5xx error rates.
    - SQS queue length and Lambda errors.
    
2. **Centralized Logging**:
    - Send EC2 application logs to CloudWatch Logs.
    - Enable ALB access logs → S3 bucket.
    - Enable RDS Performance Insights (optional but useful).
    
3. **Dashboards**: Create a CloudWatch dashboard with graphs for:
    - Web/App latency        
    - RDS health
    - Queue depth
    - Global request counts
    


---


## Security

### 24. Security Hardening

- Enable **AWS WAF** on CloudFront to protect against DDoS, SQLi, XSS.
- Rotate IAM user access keys (use IAM roles instead).
- Encrypt:
    - EBS volumes (already covered by default KMS).
    - RDS at rest (done)
    - S3 buckets (if used later).
    
- Enable **GuardDuty + Security Hub** for continuous monitoring.    

---

# Test the Project




