# Smart Shop Platform - End-to-End Deployment Guide

This document outlines the detailed, step-by-step phased approach to deploying the Smart Shop Platform from a fresh AWS account.

---

## Phase 1: Foundational AWS & CI/CD Setup (One-time)

1. **Provision EC2 for CI/CD Tools**:
    * Launch a single **t3.medium** or **t3.large** EC2 instance. (A `t2.micro` will not have enough memory to run all these tools).
    * Install **Jenkins**, **SonarQube**, and **Nexus** on this instance. (Using Docker Compose is the easiest way to manage all three).
    * Configure the AWS Security Group to allow inbound access to Jenkins (typically port 8080), SonarQube (9000), and Nexus (8081) from your IP address.

2. **Create S3 Bucket for Terraform State**:
    * Go to the AWS S3 console.
    * Create a new bucket with a globally unique name (e.g., `ssp-terraform-state-bucket-yourname-2024`).
    * **Crucial:** Enable **Bucket Versioning** to prevent accidental loss or corruption of your Terraform state.
    * *Action:* Once created, update the `backend "s3"` block in all `terraform/main.tf` files across the project to match your actual bucket name.

3. **Configure Jenkins**:
    * **Install Plugins**: Go to `Manage Jenkins` -> `Plugins`. Install:
        * `Pipeline`
        * `Docker Pipeline`
        * `Terraform`
        * `SonarQube Scanner`
    * **Configure Tools**: Go to `Manage Jenkins` -> `Global Tool Configuration`. Add installations for Docker and Terraform so Jenkins can execute those CLI commands.
    * **Configure SonarQube**: Go to `Manage Jenkins` -> `Configure System`. Add your SonarQube server details (Name it exactly `SonarQube-Server` to match the Jenkinsfiles).
    * **Create Credentials**: Go to `Manage Jenkins` -> `Credentials`. Add:
        * AWS Credentials (`AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY`) for your AWS account so Jenkins can push to ECR and run Terraform.
        * Your SonarQube authentication token (Give it the ID `SONAR_TOKEN`).

---

## Phase 2: Deploying the Base Infrastructure

1. **Deploy `ssp-base-infra` via Jenkins**:
    * In Jenkins, create a new **Multibranch Pipeline** job named `ssp-base-infra`.
    * Point it to your GitHub repository containing the base infra code.
    * Jenkins will scan the repo, find the `Jenkinsfile`, and run the `Terraform Plan` stage automatically.
    * Go into the Jenkins build and click **"Apply"** on the manual approval gate.
    * Terraform will now provision the VPC, subnets, NAT Gateway, RDS PostgreSQL, DocumentDB, ElastiCache, ECS Cluster, and the EC2 Kafka Broker.

2. **Understand Automatic vs. Manual SSM Parameters**:
    * **Automatic:** The Terraform code in `ssp-base-infra` is configured to automatically grab the generated database endpoints, URLs, and IPs and push them directly to AWS SSM Parameter Store as `String` types. You do not need to hunt for these!
    * **Manual (Secrets):** You *do* need to manually create the sensitive secrets in SSM, as these should never be committed to GitHub or hardcoded in Terraform state.

3. **Manually Create Secret SSM Parameters**:
    * Go to the AWS Systems Manager (SSM) console -> Parameter Store.
    * Create the following parameters as **`SecureString`**:
        * `/ssp/auth/jwt_secret` (Enter a strong, random string)
        * `/ssp/payment/stripe_secret_key` (From your Stripe Developer Dashboard)
        * `/ssp/payment/stripe_webhook_secret` (From your Stripe Developer Dashboard)
    * Create the following parameters as **`String`**:
        * `/ssp/notification/ses_sender_email` (Enter an email address you have verified in AWS SES)

---

## Phase 3: Deploying the Microservices

*Note: Order matters slightly for the first run. Deploy `ssp-log-forwarder` early so it can catch logs from others. Database schemas (Alembic) will automatically run on startup for Auth, Order, and Inventory services.*

1. **Create Jenkins Jobs for Services**:
    * For a service (e.g., `ssp-auth-service`), create a new **Multibranch Pipeline** job in Jenkins.
    * Point it to the respective GitHub repository.
    * **Important:** Configure the job as a **Parameterized Project**. Add a "Choice Parameter" named `TARGET_ENV` with choices like `dev`, `staging`, `prod`. (The Jenkinsfile uses this to select the Terraform workspace).

2. **Trigger the Build**:
    * Push a commit to your repository (or trigger it manually in Jenkins).
    * The pipeline will run Unit Tests, SonarQube analysis, build the Docker Image, and push it to AWS ECR.
    * The pipeline will pause at the "Deploy to Environment" stage, outputting the Terraform Plan.

3. **Review and Approve Deployment**:
    * Go into the paused Jenkins build.
    * Review the Terraform plan in the console output to ensure it looks correct.
    * Click **"Deploy"** (or "Proceed").
    * Jenkins will execute `terraform apply`, fetching the child modules from `terraform-infra-child` and deploying the new container image to ECS Fargate (or Lambda).

4. **Repeat**:
    * Repeat steps 1-3 for all microservices. 

---

## Phase 4: Post-Deployment & Enhancements (Optional/Resume Builders)

1. **Domain & SSL**:
    * Register a domain using AWS Route53.
    * Request an SSL Certificate via AWS Certificate Manager (ACM).
    * Update the `ssp-base-infra` ALB module to attach the ACM certificate for HTTPS support.

2. **Web Application Firewall (WAF)**:
    * Create a WebACL in AWS WAF.
    * Attach it to your Application Load Balancer to protect against SQLi, XSS, and bot traffic.

3. **React UI Service**:
    * Build out the React frontend in the `ssp-ui-service` folder.
    * Implement Role-Based Access Control (RBAC) in the UI, decoding the JWT to show Admin/Business user dashboards vs. standard user views.
    * Host the static build files in an S3 Bucket and serve them globally using an AWS CloudFront distribution.
