# Smart Shop Platform - Infrastructure & Cost Estimation

This document outlines the AWS infrastructure components used by the Smart Shop Platform and provides an estimated monthly cost for a **development/personal project setup**. 

> **⚠️ IMPORTANT DISCLAIMER:** AWS pricing is dynamic and depends heavily on usage (data transfer, API requests, storage). The estimates below assume running the resources **24/7 for a full month (730 hours)** in the `us-east-1` (N. Virginia) region. If you are building this for your resume, **it is highly recommended to destroy or stop resources when you are not actively working on them** to save costs. Some resources might be covered if your account is within the 12-month AWS Free Tier.

---

## 1. Core Networking & Foundation

These are the foundational resources created by the `ssp-base-infra` Terraform scripts.

| Service | Resource Specification | Purpose | Est. Monthly Cost (24/7) |
| :--- | :--- | :--- | :--- |
| **VPC** | 1 VPC, 2 Public, 2 Private Subnets | Network isolation | Free |
| **NAT Gateway** | 1 NAT Gateway (AZ-1) | Outbound internet for private subnets | ~$32.00 (Base) + Data |
| **Application Load Balancer (ALB)** | 1 ALB | Routing internet traffic to ECS services | ~$16.50 (Base) |
| **SSM Parameter Store** | Standard parameters | Secret & config management | Free |
| **CloudWatch Logs** | Log Groups | Centralized logging for all apps | < $1.00 (Usage based) |

---

## 2. Databases & Storage (Stateful Resources)

These are provisioned by the base infrastructure and hold application state. We selected minimal instance sizes to keep costs low.

| Service | Resource Specification | Purpose | Est. Monthly Cost (24/7) |
| :--- | :--- | :--- | :--- |
| **Amazon RDS** | `db.t3.micro` (Single-AZ) | PostgreSQL for Auth, Order, Inventory | ~$13.00 + Storage |
| **Amazon DocumentDB** | `db.t3.medium` (1 instance) | MongoDB compatible for Product catalog | ~$53.00 + Storage |
| **Amazon ElastiCache** | `cache.t2.micro` (1 node) | Redis for Cart & Session state | ~$12.00 |
| **Amazon OpenSearch** | `t3.small.search` (1 node, 10GB) | Vector and Semantic Search engine | ~$26.00 + Storage |
| **Amazon S3** | Standard Storage | Terraform state, UI Hosting, Product images | < $1.00 (Usage based) |
| **Amazon ECR** | Container Registries | Storing Docker Images for services | < $1.00 (Usage based) |

---

## 3. Compute: Microservices & Event Bus

The actual application logic running on AWS.

| Service | Resource Specification | Purpose | Est. Monthly Cost (24/7) |
| :--- | :--- | :--- | :--- |
| **ECS Fargate** | 6 Services (0.25 vCPU, 0.5 GB RAM each) | Runs Auth, Product, Order, Inventory, Cart, API Gateway | ~$10.00 to $15.00 |
| **AWS Lambda** | 6 Functions (Python/Mangum) | Runs Payment, Notification, Log Forwarder, AI Agents | Free Tier / < $1.00 |
| **EC2 (Kafka)** | 1 x `t3.small` | Self-managed Kafka Event Bus (replaced MSK) | ~$15.00 + EBS Volume |
| **CloudFront** | Global CDN | Distributing the React UI service | Free Tier / < $1.00 |

---

## 4. CI/CD & AI Integrations

| Service | Resource Specification | Purpose | Est. Monthly Cost (24/7) |
| :--- | :--- | :--- | :--- |
| **EC2 (Jenkins/Sonar/Nexus)** | 1 x `t3.large` | Running the entire CI/CD pipeline | ~$60.00 + EBS Volume |
| **Amazon Bedrock** | Anthropic Claude v2 | Fraud Detection LLM inference | Pay-per-token (Pennies) |
| **Amazon SageMaker** | Real-time Endpoint (`ml.t2.medium`) | AI Recommender inference | ~$36.00 |
| **Amazon SES / SNS** | Email / SMS endpoints | Notification service outbound traffic | Free Tier / Pennies |

---

## 💰 Total Estimated Cost Summary

If you spin up this entire architecture and leave it running 24/7 for a month:

*   **Fixed/Provisioned Hourly Costs (EC2, RDS, DocDB, NAT, ALB, OpenSearch, SageMaker):** ~$260 - $300 / month
*   **Variable Costs (Fargate, Lambda, S3, Data Transfer):** ~$15 - $25 / month

### 🛑 Cost Saving Strategies for Personal Projects

To avoid paying $300/month for a resume project, you **must** implement these strategies:

1. **Stop EC2 Instances:** Shut down the Jenkins `t3.large` server and the Kafka `t3.small` server from the AWS Console when you aren't actively developing or deploying. You only pay for EBS storage when they are stopped.
2. **Delete Endpoints:** SageMaker endpoints and OpenSearch domains charge hourly as long as they exist. If you aren't testing the AI recommenders/search, delete them and recreate them via Terraform later.
3. **Scale Fargate to Zero:** You can go into the AWS ECS Console and set the "Desired Tasks" for your services to `0` when you are done working for the day.
4. **Remove NAT Gateway:** The NAT Gateway costs ~$32/mo just to exist. If your ECS tasks don't need to fetch things from the public internet *after* pulling their images from ECR, you can sometimes work around it, though private subnets generally require it for secure outbound access (like calling Stripe).
5. **`terraform destroy`:** The beauty of having everything in Terraform (`ssp-base-infra`) is that you can run `terraform destroy` when you are done for the week, and `terraform apply` to recreate the entire architecture on Monday morning in about 15 minutes.
