# ECS Fargate Application with ALB-project

Infrastructure as Code project

## Overview

This template provisions the core AWS infrastructure required to run a scalable, Dockerized application on Amazon Elastic Container Service (ECS) using Fargate launch type. It's designed to be production-ready and easily integrated into a CI/CD pipeline.

Key components include:

*   **AWS ECS Fargate Cluster and Service:** Manages and runs your Docker containers without you needing to provision or manage servers.
*   **Application Load Balancer (ALB):** Distributes incoming HTTP/HTTPS traffic to your ECS tasks.
*   **VPC with Public and Private Subnets:** A well-architected network setup for security and scalability.
*   **Security Groups:** Configured to allow traffic flow between the ALB and your ECS tasks, and from the internet to the ALB.
*   **IAM Roles:** Dedicated roles for ECS task execution and task operations, following best practices.
*   **Sample Python Flask Application:** A basic Dockerized web application to help you quickly test the deployment.
*   **CI/CD Ready:** The infrastructure is set up to easily integrate with CI/CD services like AWS CodePipeline, GitHub Actions, or GitLab CI for automated deployments.

## Architecture

```
+----------------------------------------------------------------------------------+
| AWS Cloud                                                                        |
|                                                                                  |
|  +-----------------------------------------------------------------------------+ |
|  | VPC (10.0.0.0/16)                                                           | |
|  |                                                                             | |
|  |  +---------------------+        +-------------------------------------+   | |
|  |  | Public Subnets      |        | Private Subnets                     |   | |
|  |  |                     |        |                                     |   | |
|  |  |  Internet Gateway   |        |  NAT Gateway                        |   | |
|  |  |        |            |        |                                     |   | |
|  |  |        v            |        |        ^                            |   | |
|  |  |  +-----------+      |        |        |                            |   | |
|  |  |  | Application |      |        |  +-----------------------+        |   | |
|  |  |  | Load Balancer |<---+-------->|  ECS Fargate Cluster    |        |   | |
|  |  |  | (ALB)       |      |        |  +-----------------------+        | |
|  |  |  +-----------+      |        |    |                      |        | |
|  |  |        ^            |        |    v                      v        | |
|  |  |        |            |        |  +---------------+    +---------------+ |
|  |  |        | (HTTP/S)   |        |  | ECS Fargate   |    | ECS Fargate   | |
|  |  |        |            |        |  | Task 1        |    | Task 2        | |
|  |  |        |            |        |  | (Containerized|    | (Containerized| |
|  |  |        |            |        |  | App)          |    | App)          | |
|  |  |        |            |        |  +---------------+    +---------------+ |
|  |  |        |            |        |                                     |   | |
|  |  +---------------------+        +-------------------------------------+   | |
|  |                                                                             | |
|  +-----------------------------------------------------------------------------+ |
|                                                                                  |
|  +------------------------+                                                      |
|  | AWS Elastic Container  |                                                      |
|  | Registry (ECR)         |                                                      |
|  | (Stores Docker Images) |                                                      |
|  +------------------------+                                                      |
|                                                                                  |
+----------------------------------------------------------------------------------+

Traffic Flow:
1.  External users access the application via the ALB's DNS name.
2.  ALB distributes traffic to healthy ECS Fargate tasks running in private subnets.
3.  ECS tasks pull Docker images from ECR.
4.  NAT Gateway allows ECS tasks in private subnets to access external services (e.g., ECR, S3) if needed.

## Quick Start

Follow these steps to deploy your ECS Fargate application:

1.  **Configure AWS CLI:** Ensure you have the AWS CLI installed and configured with appropriate credentials.
2.  **Build and Push Docker Image (Optional for initial test):**
    *   Navigate to the `app/` directory.
    *   Build your Docker image: `docker build -t my-flask-app:latest .`
    *   Create an ECR repository and push your image. Replace `YOUR_AWS_ACCOUNT_ID` and `YOUR_REGION` with your details:
        ```bash
        aws ecr get-login-password --region YOUR_REGION | docker login --username AWS --password-stdin YOUR_AWS_ACCOUNT_ID.dkr.ecr.YOUR_REGION.amazonaws.com
        aws ecr create-repository --repository-name my-flask-app --region YOUR_REGION
        docker tag my-flask-app:latest YOUR_AWS_ACCOUNT_ID.dkr.ecr.YOUR_REGION.amazonaws.com/my-flask-app:latest
        docker push YOUR_AWS_ACCOUNT_ID.dkr.ecr.YOUR_REGION.amazonaws.com/my-flask-app:latest
        ```
3.  **Update Variables:**
    *   Edit `envs/dev/variables.tf` and update the `container_image` variable with the ECR image URI you just pushed (e.g., `"YOUR_AWS_ACCOUNT_ID.dkr.ecr.YOUR_REGION.amazonaws.com/my-flask-app:latest"`).
    *   Adjust `project_name`, `app_name`, `desired_count`, `fargate_cpu`, and `fargate_memory` as needed.
4.  **Initialize Terraform:**
    ```bash
    make init
    ```
5.  **Plan the Deployment:**
    ```bash
    make plan
    ```
6.  **Apply the Deployment:**
    ```bash
    make apply
    ```
7.  **Access Your Application:**
    Once applied, Terraform will output `load_balancer_dns_name`. Copy this DNS name and paste it into your web browser to access your deployed application.

## Project Structure

```
.
├── app/
│   ├── Dockerfile
│   ├── app.py
│   └── requirements.txt
├── envs/
│   └── dev/
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
├── infra/
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   ├── providers.tf
│   └── backend.tf
└── Makefile
└── README.md
```

## Environment Configuration

The `envs/dev/variables.tf` file defines the configurable parameters for your deployment:

| Variable        | Description                                            | Default          |
| :-------------- | :----------------------------------------------------- | :--------------- |
| `environment`   | Environment name (e.g., dev, staging, prod)            | `dev`            |
| `aws_region`    | AWS region to deploy resources                         | `us-east-1`      |
| `project_name`  | Project name used for resource naming                  | `my-ecs-app`     |
| `app_name`      | Name of the application                                | `my-flask-app`   |
| `container_image` | Docker image to deploy (e.g., 'your-ecr-repo/your-app:latest') | `python:3.9-slim-buster` (Placeholder) |
| `container_port` | Port the application listens on inside the container | `80`             |
| `desired_count` | Desired number of ECS tasks                            | `1`              |
| `fargate_cpu`   | Fargate task CPU units (e.g., 256, 512, 1024, 2048, 4096) | `256` (0.25 vCPU) |
| `fargate_memory` | Fargate task memory in MiB (e.g., 512, 1024, 2048, ...) | `512` (0.5 GB)    |
| `tags`          | Additional tags to apply to all resources              | `{}`             |

## Available Commands

Use the `Makefile` for convenient Terraform operations:

*   `make init`: Initializes Terraform and downloads necessary providers and modules.
*   `make plan`: Shows an execution plan of changes Terraform will make.
*   `make apply`: Applies the planned changes to create or update infrastructure.
*   `make destroy`: Destroys all managed infrastructure. **Use with caution!**

## CI/CD Integration (CodePipeline)

This template provides the foundational infrastructure for your application. To integrate with AWS CodePipeline:

1.  **Source Stage:** Configure CodePipeline to pull your application code (e.g., from GitHub, CodeCommit, S3).
2.  **Build Stage:** Use AWS CodeBuild to:
    *   Build your Docker image.
    *   Push the Docker image to AWS Elastic Container Registry (ECR).
    *   Generate a new `taskdef.json` file (or update the image in the existing one) with the latest image URI.
3.  **Deploy Stage:** Configure CodePipeline to deploy the updated ECS service:
    *   Use an `ECS` deploy action to update the service with the new `taskdef.json`.

You may need to create additional IAM roles and policies for CodePipeline and CodeBuild to interact with ECR and ECS.

## Support

For any questions or issues, please refer to the Senora.dev documentation or contact our support team.

*This template is maintained by Senora.dev.*

## Environment Variables

This project uses environment-specific variable files in the `envs/` directory.

### dev
Variables are stored in `envs/dev/terraform.tfvars`



## GitHub Actions CI/CD

This project includes automated Terraform validation via GitHub Actions.

### Required GitHub Secrets

Configure these in Settings > Secrets > Actions:

- `AWS_ACCESS_KEY_ID`: Your AWS Access Key ID
- `AWS_SECRET_ACCESS_KEY`: Your AWS Secret Access Key
- `TF_STATE_BUCKET`: S3 bucket name for Terraform state
- `TF_STATE_KEY`: Path to state file in S3 bucket

💡 **Tip**: Check your `backend.tf` file for the bucket and key values.


---
*Generated by [Senora](https://senora.dev)*
