# Module 07: Cloud Fundamentals

> Part of the [DevOps Career Course](./README.md) by UncleJS

---

## Table of Contents

- [Overview](#overview)
- [Learning Objectives](#learning-objectives)
- [Beginner: Cloud Computing Concepts](#beginner-cloud-computing-concepts)
- [Beginner: Cloud Service Models](#beginner-cloud-service-models)
- [Beginner: The Big Three — AWS, Azure, GCP](#beginner-the-big-three--aws-azure-gcp)
- [Intermediate: Compute Services](#intermediate-compute-services)
- [Intermediate: Storage Services](#intermediate-storage-services)
- [Intermediate: Networking in the Cloud](#intermediate-networking-in-the-cloud)
- [Intermediate: Identity & Access Management (IAM)](#intermediate-identity--access-management-iam)
- [Intermediate: Databases in the Cloud](#intermediate-databases-in-the-cloud)
- [Intermediate: Cloud CLI Tools](#intermediate-cloud-cli-tools)
- [Intermediate: Cloud Cost Management](#intermediate-cloud-cost-management)
- [Advanced: Serverless & Functions as a Service](#advanced-serverless--functions-as-a-service)
- [Advanced: Auto Scaling & High Availability Groups](#advanced-auto-scaling--high-availability-groups)
- [Advanced: Advanced VPC Networking](#advanced-advanced-vpc-networking)
- [Advanced: Container Registries](#advanced-container-registries)
- [Tools & Commands Reference](#tools--commands-reference)
- [Hands-On Labs](#hands-on-labs)
- [Further Reading](#further-reading)

---

## Overview

Cloud computing is the backbone of modern DevOps. Instead of buying and managing physical servers, you provision infrastructure on-demand from a provider — pay for what you use, scale in minutes, and access global infrastructure from an API.

This module is cloud-agnostic: we cover concepts that apply across all providers, then show the equivalent service in AWS, Azure, and GCP side-by-side.

[↑ Back to TOC](#table-of-contents)

---

## Learning Objectives

By the end of this module you will be able to:

- Explain the core cloud service and deployment models
- Navigate the three major cloud providers and their equivalent services
- Launch and manage virtual machines on any cloud provider
- Create and manage object storage buckets
- Configure cloud networking (VPC, subnets, security groups)
- Manage identities and permissions with IAM
- Use cloud-managed databases
- Use the CLI tools for AWS, Azure, and GCP
- Estimate and optimize cloud costs
- Deploy serverless functions with AWS Lambda, Azure Functions, and GCP Cloud Functions
- Configure Auto Scaling Groups and managed instance groups for elastic compute
- Design advanced VPC topologies with NAT gateways, VPC peering, and private endpoints
- Set up and use cloud container registries (ECR, ACR, Artifact Registry)

[↑ Back to TOC](#table-of-contents)

---

## Beginner: Cloud Computing Concepts

### Why Cloud?

| Traditional (On-Premises) | Cloud |
|---|---|
| Buy hardware upfront | Pay as you go |
| Months to provision | Minutes to provision |
| Fixed capacity | Infinite scale on demand |
| You maintain hardware | Provider maintains hardware |
| Single region | Global availability |

### Deployment Models

| Model | Description | Who Manages Hardware |
|---|---|---|
| **Public Cloud** | Shared infrastructure by provider (AWS, Azure, GCP) | Provider |
| **Private Cloud** | Dedicated infrastructure for one org | You (or co-lo) |
| **Hybrid Cloud** | Mix of public and private | Both |
| **Multi-Cloud** | Use multiple public cloud providers | Multiple providers |

[↑ Back to TOC](#table-of-contents)

---

## Beginner: Cloud Service Models

| Model | You Manage | Provider Manages | Examples |
|---|---|---|---|
| **IaaS** (Infrastructure as a Service) | OS, apps, data | Hardware, virtualization, network | EC2, Azure VMs, GCE |
| **PaaS** (Platform as a Service) | Apps, data | OS, runtime, middleware, hardware | AWS Elastic Beanstalk, Azure App Service, GCP App Engine |
| **SaaS** (Software as a Service) | Your data only | Everything | Gmail, Salesforce, GitHub |
| **FaaS** (Functions as a Service) | Code only | Everything else | AWS Lambda, Azure Functions, GCP Cloud Functions |
| **CaaS** (Containers as a Service) | Containers | Orchestration, infrastructure | EKS, AKS, GKE |

[↑ Back to TOC](#table-of-contents)

---

## Beginner: The Big Three — AWS, Azure, GCP

### Market Position (2025)

| Provider | Market Share | Strengths |
|---|---|---|
| **AWS** (Amazon Web Services) | ~31% | Broadest service catalog, most job postings |
| **Microsoft Azure** | ~25% | Strong enterprise/Microsoft integration |
| **Google Cloud (GCP)** | ~12% | Kubernetes (invented it), data/ML, pricing |

### Equivalent Services Cross-Reference

| Category | AWS | Azure | GCP |
|---|---|---|---|
| **Virtual Machines** | EC2 | Virtual Machines | Compute Engine |
| **Kubernetes** | EKS | AKS | GKE |
| **Serverless Functions** | Lambda | Azure Functions | Cloud Functions |
| **Object Storage** | S3 | Blob Storage | Cloud Storage |
| **Block Storage** | EBS | Managed Disks | Persistent Disk |
| **File Storage** | EFS | Azure Files | Filestore |
| **Managed PostgreSQL** | RDS (PostgreSQL) | Azure Database for PostgreSQL | Cloud SQL |
| **NoSQL Database** | DynamoDB | Cosmos DB | Firestore / Bigtable |
| **VPC Networking** | VPC | Virtual Network (VNet) | VPC |
| **Load Balancer** | ELB/ALB/NLB | Azure Load Balancer / App Gateway | Cloud Load Balancing |
| **DNS** | Route 53 | Azure DNS | Cloud DNS |
| **CDN** | CloudFront | Azure CDN | Cloud CDN |
| **IAM** | IAM | Azure AD / Entra ID | Cloud IAM |
| **Secrets** | Secrets Manager | Key Vault | Secret Manager |
| **Container Registry** | ECR | Azure Container Registry | Artifact Registry |
| **CI/CD** | CodePipeline | Azure DevOps | Cloud Build |
| **Monitoring** | CloudWatch | Azure Monitor | Cloud Monitoring |
| **Logging** | CloudWatch Logs | Log Analytics | Cloud Logging |

[↑ Back to TOC](#table-of-contents)

---

## Intermediate: Compute Services

### AWS EC2

```bash
# Launch an instance via AWS CLI
aws ec2 run-instances \
  --image-id ami-0c55b159cbfafe1f0 \    # Replace with current AMI for your region — see: aws ec2 describe-images
  --instance-type t3.micro \
  --key-name my-keypair \
  --security-group-ids sg-12345678 \
  --subnet-id subnet-12345678 \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=webserver}]'

# List running instances
aws ec2 describe-instances --filters "Name=instance-state-name,Values=running" \
  --query 'Reservations[].Instances[].[InstanceId,PublicIpAddress,Tags[?Key==`Name`].Value|[0]]' \
  --output table

# Stop / start / terminate
aws ec2 stop-instances --instance-ids i-1234567890abcdef0
aws ec2 start-instances --instance-ids i-1234567890abcdef0
aws ec2 terminate-instances --instance-ids i-1234567890abcdef0
```

### Azure VMs

```bash
# Create a resource group
az group create --name myRG --location eastus

# Create a VM
az vm create \
  --resource-group myRG \
  --name myVM \
  --image Ubuntu2404    # Ubuntu 24.04 LTS \
  --admin-username azureuser \
  --generate-ssh-keys \
  --size Standard_B1s

# List VMs
az vm list --output table

# Open a port
az vm open-port --port 80 --resource-group myRG --name myVM

# Stop / deallocate / delete
az vm deallocate --resource-group myRG --name myVM
az vm delete --resource-group myRG --name myVM
```

### GCP Compute Engine

```bash
# Create a VM
gcloud compute instances create webserver \
  --machine-type=e2-micro \
  --image-family=ubuntu-2404-lts \    # Ubuntu 24.04 LTS
  --image-project=ubuntu-os-cloud \
  --zone=us-central1-a \
  --tags=http-server

# List instances
gcloud compute instances list

# SSH into instance
gcloud compute ssh webserver --zone=us-central1-a

# Stop / delete
gcloud compute instances stop webserver --zone=us-central1-a
gcloud compute instances delete webserver --zone=us-central1-a
```

[↑ Back to TOC](#table-of-contents)

---

## Intermediate: Storage Services

### Object Storage (S3 / Blob / GCS)

Object storage is for unstructured data: backups, logs, static files, build artifacts.

```bash
# AWS S3
aws s3 mb s3://my-unique-bucket-name                    # Create bucket
aws s3 ls                                                # List buckets
aws s3 ls s3://my-bucket/                               # List objects
aws s3 cp file.txt s3://my-bucket/                      # Upload
aws s3 cp s3://my-bucket/file.txt ./                    # Download
aws s3 sync ./local-folder s3://my-bucket/folder/      # Sync directory
aws s3 rm s3://my-bucket/file.txt                       # Delete object
aws s3 rb s3://my-bucket --force                        # Delete bucket

# Azure Blob Storage
az storage account create --name mystorageaccount --resource-group myRG --location eastus --sku Standard_LRS
az storage container create --name mycontainer --account-name mystorageaccount
az storage blob upload --file ./file.txt --container-name mycontainer --name file.txt --account-name mystorageaccount
az storage blob list --container-name mycontainer --account-name mystorageaccount --output table

# GCP Cloud Storage
gsutil mb gs://my-unique-bucket-name
gsutil ls
gsutil cp file.txt gs://my-bucket/
gsutil cp gs://my-bucket/file.txt ./
gsutil rsync -r ./local gs://my-bucket/
gsutil rm gs://my-bucket/file.txt
```

[↑ Back to TOC](#table-of-contents)

---

## Intermediate: Networking in the Cloud

### VPC Concepts

A VPC (Virtual Private Cloud) is your isolated private network in the cloud.

```
VPC: 10.0.0.0/16
├── Public Subnet:  10.0.1.0/24  (has route to Internet Gateway)
│   └── Web servers, load balancers
├── Private Subnet: 10.0.2.0/24  (no direct internet access)
│   └── App servers, databases
└── Database Subnet: 10.0.3.0/24 (private, restricted)
    └── RDS, ElastiCache
```

```bash
# AWS VPC
aws ec2 create-vpc --cidr-block 10.0.0.0/16
aws ec2 create-subnet --vpc-id vpc-12345 --cidr-block 10.0.1.0/24 --availability-zone us-east-1a

# Security Group (firewall for EC2 instances)
aws ec2 create-security-group --group-name web-sg --description "Web SG" --vpc-id vpc-12345
aws ec2 authorize-security-group-ingress --group-id sg-12345 --protocol tcp --port 80 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id sg-12345 --protocol tcp --port 22 --cidr your.ip/32

# GCP VPC
gcloud compute networks create my-vpc --subnet-mode=custom
gcloud compute networks subnets create my-subnet \
  --network=my-vpc \
  --range=10.0.1.0/24 \
  --region=us-central1

# GCP Firewall rule
gcloud compute firewall-rules create allow-http \
  --network=my-vpc \
  --allow=tcp:80 \
  --source-ranges=0.0.0.0/0
```

[↑ Back to TOC](#table-of-contents)

---

## Intermediate: Identity & Access Management (IAM)

IAM controls **who** can do **what** on **which resources**.

### Core IAM Concepts

| Concept | AWS | Azure | GCP |
|---|---|---|---|
| **User** | IAM User | Microsoft Entra ID User | Google Account |
| **Group** | IAM Group | Microsoft Entra ID Group | Google Group |
| **Role** | IAM Role | Azure Role | IAM Role |
| **Policy/Permission** | IAM Policy | Role Definition | IAM Binding |
| **Service Identity** | IAM Role (for EC2) | Managed Identity | Service Account |

### AWS IAM Example

```bash
# Create an IAM user
aws iam create-user --user-name devops-user

# Attach a policy
aws iam attach-user-policy \
  --user-name devops-user \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

# Create an IAM role for EC2 (service identity)
aws iam create-role \
  --role-name ec2-s3-role \
  --assume-role-policy-document file://trust-policy.json

# List users
aws iam list-users
```

### IAM Best Practices

1. **Principle of Least Privilege** — grant only the permissions needed
2. **Never use root account** for day-to-day tasks
3. **Use roles, not users** for applications and services
4. **Enable MFA** for all human users
5. **Rotate credentials regularly** — access keys, passwords
6. **Use service accounts/managed identities** instead of hardcoded credentials

[↑ Back to TOC](#table-of-contents)

---

## Intermediate: Databases in the Cloud

Cloud-managed databases eliminate the overhead of installation, patching, backups, and replication.

```bash
# AWS RDS (PostgreSQL)
aws rds create-db-instance \
  --db-instance-identifier mydb \
  --db-instance-class db.t3.micro \
  --engine postgres \
  --engine-version 16.1 \
  --master-username admin \
  --master-user-password "${DB_PASSWORD}" \    # Never hardcode — use a secret or env var
  --allocated-storage 20 \
  --vpc-security-group-ids sg-12345 \
  --no-publicly-accessible

# Azure Database for PostgreSQL
az postgres flexible-server create \
  --resource-group myRG \
  --name mypostgres \
  --admin-user admin \
  --admin-password "${DB_PASSWORD}" \    # Never hardcode — use a secret or env var
  --sku-name Standard_B1ms \
  --tier Burstable

# GCP Cloud SQL
gcloud sql instances create mydb \
  --database-version=POSTGRES_16 \
  --tier=db-f1-micro \
  --region=us-central1

gcloud sql databases create appdb --instance=mydb
gcloud sql users create admin --instance=mydb --password="${DB_PASSWORD}"   # Never hardcode — use a secret or env var
```

[↑ Back to TOC](#table-of-contents)

---

## Intermediate: Cloud CLI Tools

### AWS CLI Setup

```bash
# Install
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip && sudo ./aws/install

# Configure
aws configure
# Prompts for: Access Key ID, Secret Access Key, Region, Output format

# Test
aws sts get-caller-identity

# Use named profiles
aws configure --profile production
aws s3 ls --profile production
export AWS_PROFILE=production
```

### Azure CLI Setup

```bash
# Install
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash

# Login
az login                            # Opens browser
az login --service-principal ...   # For automation

# Set subscription
az account list --output table
az account set --subscription "My Subscription"

# Test
az account show
```

### gcloud CLI Setup

```bash
# Install
curl https://sdk.cloud.google.com | bash
exec -l $SHELL

# Initialize
gcloud init

# Login
gcloud auth login
gcloud auth application-default login   # For SDKs/tools

# Set project
gcloud config set project my-project-id

# Test
gcloud config list
```

[↑ Back to TOC](#table-of-contents)

---

## Intermediate: Cloud Cost Management

### Key Principles

1. **Right-sizing** — use the smallest instance type that meets your needs
2. **Auto-scaling** — scale in when load drops, scale out when load rises
3. **Reserved/Committed use** — commit to 1 or 3 years for 30–70% discounts
4. **Spot/Preemptible/Spot VMs** — up to 90% off for interruptible workloads
5. **Delete unused resources** — stale snapshots, old load balancers, unattached volumes
6. **Storage tiering** — move infrequently accessed data to cheaper storage classes

### Cost Tools

| Tool | Provider | Purpose |
|---|---|---|
| AWS Cost Explorer | AWS | Visualize and analyze spending |
| AWS Budgets | AWS | Set alerts when costs exceed thresholds |
| Azure Cost Management | Azure | Cost analysis and budgets |
| GCP Billing Reports | GCP | Usage and cost reports |
| Infracost | Any (Terraform) | Estimate cost of IaC changes in PRs |

```bash
# AWS — get current month costs
aws ce get-cost-and-usage \
  --time-period Start=$(date -d "first day of month" +%Y-%m-%d),End=$(date +%Y-%m-%d) \
  --granularity MONTHLY \
  --metrics UnblendedCost

# GCP — check billing
gcloud billing accounts list
```

[↑ Back to TOC](#table-of-contents)

---

## Advanced: Serverless & Functions as a Service

Serverless (FaaS) lets you run code without provisioning or managing servers. You pay only for execution time — billed in milliseconds.

### AWS Lambda

```bash
# Create a simple Lambda function (Python)
cat > handler.py << 'EOF'
import json

def lambda_handler(event, context):
    name = event.get("name", "World")
    return {
        "statusCode": 200,
        "body": json.dumps({"message": f"Hello, {name}!"})
    }
EOF

# Package it
zip function.zip handler.py

# Create the Lambda function
aws lambda create-function \
  --function-name hello-world \
  --runtime python3.12 \
  --role arn:aws:iam::123456789012:role/lambda-execution-role \
  --handler handler.lambda_handler \
  --zip-file fileb://function.zip

# Invoke it
aws lambda invoke \
  --function-name hello-world \
  --payload '{"name": "DevOps"}' \
  --cli-binary-format raw-in-base64-out \
  response.json
cat response.json

# Update function code
zip function.zip handler.py
aws lambda update-function-code \
  --function-name hello-world \
  --zip-file fileb://function.zip

# Add an HTTP trigger (API Gateway)
aws lambda add-permission \
  --function-name hello-world \
  --statement-id apigateway-invoke \
  --action lambda:InvokeFunction \
  --principal apigateway.amazonaws.com
```

### Azure Functions

```bash
# Create a Function App
az functionapp create \
  --resource-group myRG \
  --consumption-plan-location eastus \
  --runtime python \
  --runtime-version 3.11 \
  --functions-version 4 \
  --name my-function-app \
  --storage-account mystorageaccount

# Deploy from local project
func azure functionapp publish my-function-app
```

### GCP Cloud Functions

```bash
# Deploy a Cloud Function
cat > main.py << 'EOF'
import functions_framework

@functions_framework.http
def hello(request):
    name = request.args.get("name", "World")
    return f"Hello, {name}!"
EOF

gcloud functions deploy hello \
  --runtime python312 \
  --trigger-http \
  --allow-unauthenticated \
  --region us-central1

# Call it
gcloud functions call hello --data '{"name": "DevOps"}' --region us-central1
```

### When to use serverless

| Use case | Good fit? |
|---|---|
| Webhooks / event processing | ✅ Excellent |
| Scheduled jobs / cron tasks | ✅ Good |
| API backends with variable traffic | ✅ Good |
| Long-running batch jobs (> 15 min) | ❌ Poor — use containers |
| Stateful workloads | ❌ Poor — functions are stateless |
| Low-latency requirements (cold start) | ⚠️ Provisioned concurrency helps |

[↑ Back to TOC](#table-of-contents)

---

## Advanced: Auto Scaling & High Availability Groups

Auto Scaling automatically adjusts compute capacity based on demand — scaling out under load and scaling in when idle.

### AWS Auto Scaling Groups (ASG)

```bash
# Create a launch template
aws ec2 create-launch-template \
  --launch-template-name web-lt \
  --launch-template-data '{
    "ImageId": "ami-0c55b159cbfafe1f0",
    "InstanceType": "t3.micro",
    "SecurityGroupIds": ["sg-12345678"],
    "UserData": "IyEvYmluL2Jhc2gKYXB0LWdldCB1cGRhdGUKYXB0LWdldCBpbnN0YWxsIC15IG5naW54"
  }'

# Create an Auto Scaling Group
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name web-asg \
  --launch-template LaunchTemplateName=web-lt,Version='$Latest' \
  --min-size 2 \
  --max-size 10 \
  --desired-capacity 2 \
  --vpc-zone-identifier "subnet-abc123,subnet-def456" \
  --health-check-type ELB \
  --health-check-grace-period 300

# Attach to a load balancer target group
aws autoscaling attach-load-balancer-target-groups \
  --auto-scaling-group-name web-asg \
  --target-group-arns arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/my-tg/abc123

# Create a scaling policy (target tracking — CPU at 60%)
aws autoscaling put-scaling-policy \
  --auto-scaling-group-name web-asg \
  --policy-name cpu-target-tracking \
  --policy-type TargetTrackingScaling \
  --target-tracking-configuration '{
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ASGAverageCPUUtilization"
    },
    "TargetValue": 60.0
  }'

# Manually scale
aws autoscaling set-desired-capacity \
  --auto-scaling-group-name web-asg \
  --desired-capacity 5
```

### GCP Managed Instance Groups (MIG)

```bash
# Create an instance template
gcloud compute instance-templates create web-template \
  --machine-type=e2-micro \
  --image-family=ubuntu-2404-lts \
  --image-project=ubuntu-os-cloud \
  --tags=http-server \
  --metadata=startup-script='#!/bin/bash
    apt-get update && apt-get install -y nginx'

# Create a regional managed instance group with autoscaling
gcloud compute instance-groups managed create web-mig \
  --base-instance-name=web \
  --template=web-template \
  --size=2 \
  --region=us-central1

# Configure autoscaling
gcloud compute instance-groups managed set-autoscaling web-mig \
  --region=us-central1 \
  --max-num-replicas=10 \
  --min-num-replicas=2 \
  --target-cpu-utilization=0.6 \
  --cool-down-period=90
```

[↑ Back to TOC](#table-of-contents)

---

## Advanced: Advanced VPC Networking

### NAT Gateway — outbound internet for private subnets

Private subnets (where your app servers and databases live) have no direct internet access. A **NAT Gateway** in the public subnet allows outbound traffic while blocking inbound connections.

```bash
# AWS — create a NAT Gateway
aws ec2 allocate-address --domain vpc   # Get an Elastic IP
aws ec2 create-nat-gateway \
  --subnet-id subnet-public-12345 \
  --allocation-id eipalloc-12345678

# Add a route in the private subnet's route table
aws ec2 create-route \
  --route-table-id rtb-private-12345 \
  --destination-cidr-block 0.0.0.0/0 \
  --nat-gateway-id nat-12345678abcdef0
```

```bash
# GCP — Cloud NAT
gcloud compute routers create my-router \
  --network=my-vpc \
  --region=us-central1

gcloud compute routers nats create my-nat \
  --router=my-router \
  --region=us-central1 \
  --auto-allocate-nat-external-ips \
  --nat-all-subnet-ip-ranges
```

### VPC Peering — connect two VPCs

VPC peering allows private IP communication between two VPCs — within the same account or across accounts.

```bash
# AWS — create a peering connection
aws ec2 create-vpc-peering-connection \
  --vpc-id vpc-aaaa1111 \
  --peer-vpc-id vpc-bbbb2222

# Accept the peering request (from the peer account/region if cross-account)
aws ec2 accept-vpc-peering-connection \
  --vpc-peering-connection-id pcx-12345678

# Add routes in both VPCs
aws ec2 create-route \
  --route-table-id rtb-aaaa1111 \
  --destination-cidr-block 10.1.0.0/16 \
  --vpc-peering-connection-id pcx-12345678
```

### Private Endpoints — access AWS services without internet

Without private endpoints, traffic to S3 or DynamoDB from your VPC traverses the public internet. **VPC Endpoints** (AWS) / **Private Service Connect** (GCP) keep traffic on the AWS backbone.

```bash
# AWS — Gateway endpoint for S3 (free)
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-12345 \
  --service-name com.amazonaws.us-east-1.s3 \
  --route-table-ids rtb-12345

# AWS — Interface endpoint for Secrets Manager (charged per hour)
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-12345 \
  --service-name com.amazonaws.us-east-1.secretsmanager \
  --vpc-endpoint-type Interface \
  --subnet-ids subnet-private-12345 \
  --security-group-ids sg-12345
```

### Full production VPC topology

```
VPC: 10.0.0.0/16
│
├── AZ us-east-1a
│   ├── Public subnet 10.0.1.0/24
│   │   ├── Load Balancer (ALB)
│   │   └── NAT Gateway + Elastic IP
│   ├── Private subnet 10.0.11.0/24 (app servers)
│   └── Database subnet 10.0.21.0/24 (RDS, ElastiCache)
│
└── AZ us-east-1b
    ├── Public subnet 10.0.2.0/24
    │   ├── Load Balancer (ALB)
    │   └── NAT Gateway + Elastic IP
    ├── Private subnet 10.0.12.0/24 (app servers)
    └── Database subnet 10.0.22.0/24 (RDS Multi-AZ standby)

Route tables:
  Public subnets → Internet Gateway (0.0.0.0/0)
  Private subnets → NAT Gateway in same AZ (0.0.0.0/0)
  Database subnets → No internet route (isolated)

VPC Endpoints:
  S3 Gateway Endpoint → on private route tables
  Secrets Manager Interface Endpoint → in private subnets
```

[↑ Back to TOC](#table-of-contents)

---

## Advanced: Container Registries

Container registries store and distribute Docker/OCI images. Cloud providers offer managed registries tightly integrated with their compute services.

```bash
# AWS ECR — Elastic Container Registry
# Create a repository
aws ecr create-repository --repository-name myapp --region us-east-1

# Authenticate Docker to ECR
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  123456789012.dkr.ecr.us-east-1.amazonaws.com

# Tag and push an image
docker tag myapp:latest 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:latest
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:latest

# Enable image scanning on push
aws ecr put-image-scanning-configuration \
  --repository-name myapp \
  --image-scanning-configuration scanOnPush=true

# Azure Container Registry (ACR)
az acr create --resource-group myRG --name myregistry --sku Basic
az acr login --name myregistry
docker tag myapp:latest myregistry.azurecr.io/myapp:latest
docker push myregistry.azurecr.io/myapp:latest

# GCP Artifact Registry
gcloud artifacts repositories create myrepo \
  --repository-format=docker \
  --location=us-central1

gcloud auth configure-docker us-central1-docker.pkg.dev

docker tag myapp:latest us-central1-docker.pkg.dev/my-project/myrepo/myapp:latest
docker push us-central1-docker.pkg.dev/my-project/myrepo/myapp:latest
```

[↑ Back to TOC](#table-of-contents)

---

## Tools & Commands Reference

| Tool | Command Example | Purpose |
|---|---|---|
| AWS CLI | `aws ec2 describe-instances` | Manage AWS resources |
| Azure CLI | `az vm list` | Manage Azure resources |
| gcloud | `gcloud compute instances list` | Manage GCP resources |
| `aws s3` | `aws s3 cp file.txt s3://bucket/` | S3 file operations |
| `gsutil` | `gsutil cp file.txt gs://bucket/` | GCS file operations |
| `aws configure` | — | Set up AWS credentials |
| `az login` | — | Authenticate to Azure |
| `gcloud init` | — | Initialize GCP CLI |
| `aws lambda invoke` | `aws lambda invoke --function-name fn out.json` | Invoke a Lambda function |
| `aws autoscaling` | `aws autoscaling describe-auto-scaling-groups` | Manage ASGs |
| `aws ecr get-login-password` | — | Authenticate Docker to ECR |
| `az acr login` | `az acr login --name myregistry` | Authenticate Docker to ACR |
| `aws ec2 create-nat-gateway` | — | Create a NAT Gateway |
| `aws ec2 create-vpc-peering-connection` | — | Peer two VPCs |

[↑ Back to TOC](#table-of-contents)

---

## Hands-On Labs

### Lab 7.1 — First Cloud VM (pick any provider)

1. Create a free-tier account on AWS, Azure, or GCP
2. Launch a VM using the CLI (not the web console)
3. SSH into the VM
4. Install nginx: `sudo apt install nginx`
5. Open port 80 via a security group / firewall rule
6. Access the default nginx page from your browser
7. Stop and delete the VM (to avoid charges)

### Lab 7.2 — Object Storage

1. Create a storage bucket (S3 / Blob / GCS)
2. Upload a file via CLI
3. List the bucket contents
4. Set the file as publicly readable and access it via URL
5. Remove the public access and delete the bucket

### Lab 7.3 — IAM Roles

1. Create an IAM user or service account
2. Attach a read-only policy for object storage
3. Configure the CLI with those credentials
4. Verify you can list bucket contents but cannot create a bucket

### Lab 7.4 — Cloud Database

1. Launch a managed PostgreSQL instance (RDS / Azure / Cloud SQL)
2. Connect to it from your local machine using `psql`
3. Create a database and a table
4. Verify backup retention settings

### Lab 7.5 — Serverless Function

1. Write a simple HTTP function in Python or Node.js
2. Deploy it using AWS Lambda (or Azure Functions / GCP Cloud Functions)
3. Invoke it via the CLI and confirm the response
4. Check the invocation logs in CloudWatch / Azure Monitor / Cloud Logging
5. Delete the function and any associated resources

### Lab 7.6 — Container Registry

1. Build a simple Docker image locally: `docker build -t myapp:v1 .`
2. Create a private repository on ECR, ACR, or Artifact Registry
3. Authenticate your local Docker daemon to the registry
4. Tag and push the image to the registry
5. Pull the image from the registry on a VM to verify it works
6. Enable vulnerability scanning and review the scan results

[↑ Back to TOC](#table-of-contents)

---

## Further Reading

- [AWS Documentation](https://docs.aws.amazon.com/)
- [Azure Documentation](https://learn.microsoft.com/en-us/azure/)
- [Google Cloud Documentation](https://cloud.google.com/docs)
- [AWS Free Tier](https://aws.amazon.com/free/)
- [AWS Lambda Developer Guide](https://docs.aws.amazon.com/lambda/latest/dg/)
- [AWS Auto Scaling User Guide](https://docs.aws.amazon.com/autoscaling/ec2/userguide/)
- [AWS VPC User Guide](https://docs.aws.amazon.com/vpc/latest/userguide/)
- [A Cloud Guru / Linux Academy](https://acloudguru.com/)
- [Glossary: Cloud Provider](./glossary.md#c), [IAM](./glossary.md#i), [VPC](./glossary.md#v), [IaaS](./glossary.md#i)
- **Certifications**: AWS Cloud Practitioner, AWS DevOps Engineer, Azure Administrator AZ-104

[↑ Back to TOC](#table-of-contents)

---

*© 2026 UncleJS — Licensed under [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/)*
