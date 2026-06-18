# The Smart Shop Platform (SSP)

Welcome to the Smart Shop Platform, a complete, cloud-native e-commerce application built with a modern microservices architecture. This project serves as a comprehensive demonstration of professional software engineering practices, including Infrastructure as Code (IaC), CI/CD, event-driven design, and modern backend/frontend development.

## High-Level Architecture

The platform is designed as a collection of independent, decoupled services that communicate via APIs and an event bus. This approach ensures scalability, resilience, and maintainability.

![Smart Shop Platform Architecture](https://raw.githubusercontent.com/Smart-Shop-Platform-Project/ssp-ui-service/main/smartshop_hla.svg)

### Core Components:

1.  **React UI (`ssp-ui-service`):** A modern Single-Page Application (SPA) built with React and TypeScript, providing the customer-facing storefront. It is deployed to an S3 bucket and served globally via **Amazon CloudFront**.

2.  **API Gateway (`ssp-api-gateway`):** The single entry point for all client requests. Built with FastAPI, it handles request routing, rate limiting, and automatic retries.

3.  **Backend Microservices (ECS Fargate):**
    *   **Auth Service:** Manages user authentication and JWT generation.
    *   **Product Service:** Manages the product catalog using DocumentDB (MongoDB).
    *   **Order Service:** The core transaction engine, orchestrating orders via Kafka.
    *   **Inventory Service:** An event-driven service that consumes Kafka events to manage stock.
    *   **Cart Service:** A high-performance service using Redis for cart management.
    *   **Search Service:** Provides product search capabilities using Amazon OpenSearch.

4.  **AI & Serverless Agents (AWS Lambda):**
    *   **Payment & Notification Services:** Handle interactions with external providers (Stripe, SES) in a serverless, cost-effective way.
    *   **AI Services:** Integrate with Amazon SageMaker and Bedrock for fraud detection and product recommendations.
    *   **Log Forwarder:** A central Lambda that subscribes to all application logs to detect errors and send alerts via SNS.

5.  **Event Bus (Self-Managed Kafka):**
    *   A single-node Kafka cluster running on an EC2 instance serves as the asynchronous backbone of the system, decoupling services like `Order` and `Inventory`.

## Repository Structure (Multi-Repo)

This project follows a **multi-repo** strategy, where each microservice lives in its own dedicated GitHub repository under the [Smart Shop Platform Project Organization](https://github.com/Smart-Shop-Platform-Project).

- **Application Repositories:** All `ssp-*` services.
- **Shared Infrastructure Modules:** The `terraform-infra-child` repository contains reusable Terraform modules and is maintained separately.

## Technology & Practices

- **Infrastructure as Code:** The entire AWS infrastructure is defined using **Terraform**. A `ssp-base-infra` project provisions the foundational network and databases, while each microservice manages its own application-level resources.
- **CI/CD:** Each service has its own **Jenkinsfile** defining a complete pipeline:
    1.  Automated build trigger on push.
    2.  Unit & Integration Testing.
    3.  SonarQube for code quality analysis.
    4.  Docker image build and push to **Amazon ECR**.
    5.  Parameterized deployment with a manual approval gate before applying Terraform changes.
- **Security:**
    - No hardcoded secrets; all credentials and API keys are fetched at runtime from **AWS SSM Parameter Store** or **Secrets Manager**.
    - Fine-grained IAM roles for each service, following the principle of least privilege.
    - Rate limiting and retry logic implemented at the API Gateway.

## Getting Started

To deploy this project, please refer to the detailed, step-by-step guide in the [ACTION_ITEMS.md](./ACTION_ITEMS.md) file.
