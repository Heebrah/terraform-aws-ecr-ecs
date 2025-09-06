
# 📘 Project Notes – ECS, ECR, Docker & Terraform

## 🐳 Docker

* Docker is used to **containerize** the React application.
* The React (Vite) app is built inside a Docker image, then served with **Nginx**.
* This ensures the app runs consistently in any environment (local, staging, production).
* Key Steps:

  1. `docker build -t react-app .`
  2. `docker run -p 3000:80 react-app`
* Advantage → Portability, consistency, and easy deployment.

---

## 🏷️ Amazon ECR (Elastic Container Registry)

* ECR is a **private Docker image repository** on AWS.
* The built Docker image is tagged and pushed to ECR.
* ECS (Elastic Container Service) pulls the image from ECR at runtime.
* Key Steps:

  1. Create repository in ECR.
  2. Authenticate Docker with ECR (`aws ecr get-login-password`).
  3. Push the image (`docker push <ECR_URL>`).
* Advantage → Secure image storage integrated with AWS services.

---

## ⚡ Amazon ECS (Elastic Container Service)

* ECS is a fully managed container orchestration service.
* **Fargate launch type** is used here (serverless, no EC2 management).
* The ECS setup includes:

  * **Cluster** → logical grouping of tasks.
  * **Task Definition** → blueprint (image, CPU, memory, ports).
  * **Service** → ensures the task runs, scales, and integrates with ALB.
* The app runs in containers managed by ECS, connected to a **public ALB**.
* Advantage → No servers to manage, auto scaling, secure, integrates with VPC, ALB, IAM.

---

## ⚙️ Terraform

* Terraform provides **Infrastructure as Code (IaC)**.
* AWS resources (ECR, ECS, ALB, IAM roles, Security Groups) are defined in `.tf` files.
* Benefits:

  * Version control for infra
  * Reproducibility
  * Automated provisioning and teardown
* Key Steps:

  1. `terraform init` → Initialize modules/providers.
  2. `terraform plan` → Preview infra changes.
  3. `terraform apply` → Deploy infra.
  4. `terraform destroy` → Clean up resources.

---

## 🏗️ How They Work Together

1. **Docker** builds the React app into a container.
2. **ECR** stores the Docker image securely.
3. **Terraform** provisions infrastructure (ECS, ALB, IAM, SG, etc.).
4. **ECS** pulls the image from ECR and runs it on **Fargate**.
5. **ALB** exposes the app publicly via DNS.

---

## 🔑 Key Takeaways

* Docker standardizes the app environment.
* ECR stores and manages images securely in AWS.
* ECS (with Fargate) runs containers without server management.
* Terraform automates and codifies the entire infrastructure.

---

👉 This note shows **why each tool was chosen** and **how they work together**.

Do you want me to also create a **simple architecture diagram** (flow: Docker → ECR → ECS → ALB → Browser) and embed it in your README?




## Project Tittle
 **step-by-step implementation** using your React project:

---

## 🔹 Step 1: Dockerize Your React App

Inside your React app root directory:

**Dockerfile**

```dockerfile
# Use official Node.js image as base
FROM node:18-alpine AS build

# Set working directory
WORKDIR /app

# Copy package.json and package-lock.json
COPY package*.json ./

# Install dependencies
RUN npm install

# Copy all source code
COPY . .

# Build the React app for production
RUN npm run build

# Stage 2: Serve with Nginx
FROM nginx:alpine

# Copy built app to Nginx
COPY --from=build /app/build /usr/share/nginx/html

# Expose port
EXPOSE 80

# Run Nginx
CMD ["nginx", "-g", "daemon off;"]
```
![caption](/img/2.docker-file.jpg)
👉 Test locally:

```bash
docker build -t my-react-app .
docker run -p 3000:80 my-react-app
```
![caption](/img/3.build-docker.jpg)
![caption](/img/4.run-docker.jpg)

Visit `http://localhost:3000`.
![caption](/img/5.get-display-react.jpg)

---

## 🔹 Step 2: Terraform Module for Amazon ECR

Create folder `modules/ecr`.

**modules/ecr/main.tf**

```hcl
resource "aws_ecr_repository" "this" {
  name = "react-webapp"
}

output "repository_url" {
  value = aws_ecr_repository.this.repository_url
}
```
![caption](/img/7.main-tf-content.jpg)

---

## 🔹 Step 3: Terraform Module for ECS

Create folder `modules/ecs`.

**modules/ecs/main.tf**

```hcl
resource "aws_ecs_cluster" "this" {
  name = "react-app-cluster"
}

resource "aws_ecs_task_definition" "this" {
  family                   = "react-app-task"
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  cpu                      = "256"
  memory                   = "512"

  container_definitions = jsonencode([
    {
      name  = "react-app"
      image = var.image_url
      essential = true
      portMappings = [
        {
          containerPort = 80
          hostPort      = 80
        }
      ]
    }
  ])
}

resource "aws_ecs_service" "this" {
  name            = "react-app-service"
  cluster         = aws_ecs_cluster.this.id
  task_definition = aws_ecs_task_definition.this.arn
  desired_count   = 1
  launch_type     = "FARGATE"

  network_configuration {
    subnets         = var.subnets
    security_groups = [var.security_group]
    assign_public_ip = true
  }
}
```
where vpc, subnets, security_groups are added from our aws profile
![caption](/img/9.main-tf-ecs.jpg)

**modules/ecs/variables.tf**

```hcl
variable "image_url" {}
variable "subnets" { type = list(string) }
variable "security_group" { type = string }
```

---

## 🔹 Step 4: Main Terraform Config

In project root:

**main.tf**

```hcl
provider "aws" {
  region = "us-east-1"
}

module "ecr" {
  source = "./modules/ecr"
}

module "ecs" {
  source          = "./modules/ecs"
  image_url       = "${module.ecr.repository_url}:latest"
  subnets         = ["subnet-xxxxxx"]     # replace with your subnets
  security_group  = "sg-xxxxxx"           # replace with your security group
}
```
![caption](/img/9.main-tf-ecs.jpg)

---

## 🔹 Step 5: Deployment Steps

1. **Initialize Terraform**

   ```bash
   terraform init
   terraform apply -auto-approve
   ```
   ![caption](/img/10.terraform-init.jpg)
 

2. 🔹 **Push your Docker image to ECR**

i. **Create an ECR repo** (once):

   ```sh
   aws ecr create-repository --repository-name react-app
   ```
  ![caption](/img/25.aws-ecr-connect.jpg)
ii. **Authenticate Docker with ECR**
   ```bash
   aws ecr get-login-password --region us-east-1 \
   | docker login --username AWS --password-stdin <your_account_id>.dkr.ecr.us-east-1.amazonaws.com
   ```
   iii. **Tag your local image for ECR**:

   ```sh
   docker tag react-app:latest <your-account-id>.dkr.ecr.us-east-1.amazonaws.com/react-app:latest
   ```

iv. **Push the image**:

   ```sh
   docker push <your-account-id>.dkr.ecr.us-east-1.amazonaws.com/react-app:latest
   ```
 
  


3. **Re-run Terraform apply** (to update ECS service with latest image).

---
```
terraform apply
```

After that, `terraform apply` will create/update your ECS service → ALB → and your app will show up at the ALB DNS name. 🚀

### confirm this from Aws console
- From aws console check for ECS dashboard
![caption](/img/20.ecs-dashboard.jpg)
- Go to cluster there you can find the created react project used. This shows our ECS file is successfully created on AWS
![caption](/img/21.cluster.jpg)



