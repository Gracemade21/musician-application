# musician-app
NodeJS / React sample app for AWS CI/CD pipeline tutorial

https://www.youtube.com/watch?v=NwzJCSPSPZs


**Deploying a Node.js Musician App on AWS ECS Fargate with Load Balancer & IAM Setup**

## **Introduction**

This document provides a **step-by-step guide** on how to:

- **Containerize** a Node.js Musician App using Docker.
- **Push the Docker image** to Amazon Elastic Container Registry (ECR).
- **Set up networking** for public access using **two security groups**.
- **Deploy the app on AWS ECS (Fargate)** with **3 instances**.
- **Configure an Application Load Balancer (ALB)** for load balancing.

This guide assumes you have **AWS CLI installed** and configured with appropriate permissions.

---

## **Step 1: Create IAM Role for ECS Execution**

AWS ECS Fargate requires an IAM Role to pull images from **Amazon ECR** and access other AWS services.

### **1.1 Create a Trust Policy for ECS Execution Role**

Create a file named `ecs-execution-role-trust-policy.json` with the following content:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "ecs-tasks.amazonaws.com"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}
```

### **1.2 Create the IAM Role**

Run the following command to create the IAM role:

```cmd
aws iam create-role --role-name ecsTaskExecutionRole --assume-role-policy-document file://ecs-execution-role-trust-policy.json
```

✅ **Copy the IAM Role ARN (**``**)**

### **1.3 Attach Required Policies**

```cmd
aws iam attach-role-policy --role-name ecsTaskExecutionRole --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy
```

### **1.4 Add Permissions for Amazon ECR**

Create `ecs-ecr-policy.json` with the following content:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ecr:GetAuthorizationToken",
                "ecr:BatchCheckLayerAvailability",
                "ecr:GetDownloadUrlForLayer",
                "ecr:DescribeRepositories",
                "ecr:ListImages",
                "ecr:BatchGetImage"
            ],
            "Resource": "*"
        }
    ]
}
```

Apply this policy:

```cmd
aws iam put-role-policy --role-name ecsTaskExecutionRole --policy-name ECSTaskECRPolicy --policy-document file://ecs-ecr-policy.json
```

---

## **Step 2: Containerize the Node.js Musician App**

### **2.1 Create a **``

```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm install --production
COPY . .
EXPOSE 3001
CMD ["node", "app.js"]
```

### **2.2 Build and Run the Docker Image Locally**

```cmd
docker build -t musician-app .
docker run -p 3001:3001 musician-app
```

✅ Test at: `http://localhost:3001`

---

## **Step 3: Push Docker Image to Amazon ECR**

### **3.1 Authenticate Docker to AWS ECR**

```cmd
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin <AWS_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com
```

### **3.2 Create an ECR Repository**

```cmd
aws ecr create-repository --repository-name musician-app
```

### **3.3 Tag and Push the Image**

```cmd
docker tag musician-app:latest <AWS_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/musician-app:latest
docker push <AWS_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/musician-app:latest
```

---

## **Step 4: Set Up AWS Networking (VPC, Subnets, Security Groups)**

### **4.1 Create Security Groups**

```cmd
aws ec2 create-security-group --group-name sg-alb --description "Security group for ALB" --vpc-id <VPC_ID>
aws ec2 create-security-group --group-name sg-ecs --description "Security group for ECS tasks" --vpc-id <VPC_ID>
```

### **4.2 Open Ports in Security Groups**

```cmd
aws ec2 authorize-security-group-ingress --group-id sg-alb --protocol tcp --port 80 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id sg-ecs --protocol tcp --port 3001 --source-group sg-alb
```

---

## **Step 5: Create and Configure an Application Load Balancer**

### **5.1 Create ALB and Target Group**

```cmd
aws elbv2 create-load-balancer --name musician-lb --subnets <SUBNET_1> <SUBNET_2> --security-groups sg-alb --scheme internet-facing --type application
aws elbv2 create-target-group --name musician-target-group --protocol HTTP --port 3001 --vpc-id <VPC_ID> --target-type ip
```

### **5.2 Create a Listener for the ALB**

```cmd
aws elbv2 create-listener --load-balancer-arn <LB_ARN> --protocol HTTP --port 80 --default-actions Type=forward,TargetGroupArn=<TARGET_GROUP_ARN>
```

---

## **Step 6: Deploy ECS Service with Load Balancer**

```cmd
aws ecs create-service --cluster musician-cluster --service-name musician-service --task-definition musician-app --desired-count 3 --launch-type FARGATE --network-configuration "awsvpcConfiguration={subnets=[<SUBNET_1>,<SUBNET_2>],securityGroups=[sg-ecs],assignPublicIp=ENABLED}" --load-balancers "targetGroupArn=<TARGET_GROUP_ARN>,containerName=musician-app,containerPort=3001"
```

---

## **Step 7: Test the Application**

### **7.1 Get the ALB DNS Name**

```cmd
aws elbv2 describe-load-balancers --names musician-lb --query "LoadBalancers[0].DNSName" --output text
```

✅ Open in Browser:

```
http://musician-lb-1234567890.us-east-1.elb.amazonaws.com
```

🚀 Now your app is **live on AWS ECS Fargate with 3 instances!**

