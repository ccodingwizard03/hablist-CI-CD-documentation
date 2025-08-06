# GitHub to AWS ECS Deployment Pipeline Documentation

## Overview
This document describes the CI/CD pipeline that automatically builds and deploys a containerized application from a GitHub repository to AWS ECS (Elastic Container Service) with Fargate launch type. The pipeline triggers on pushes to the `ccodingwizard03-patch-1` branch, builds a Docker image, pushes it to Amazon ECR (Elastic Container Registry), and updates the ECS service with the new image.
This pipeline was aimed for a simple deployment for the developemnt satge and is still the frame work to be usde on the other branches 

## Pipeline Components

### 1. Trigger Conditions
- **Trigger:** The pipeline runs on every push to the `ccodingwizard03-patch-1` branch
- **Environment Variables:**
  - AWS Region: `us-east-1`
  - ECR Repository: `hablist`
  - ECS Cluster: `worthy-butterfly-8m6ir4`
  - ECS Service: `hablist-task-def-service-ucun9rgl`
  - Container Name: `hablist-container`
  - Task Definition Family: `hablist-task-def`

### 2. Pipeline Stages

#### A. Build Stage
1. **Checkout Code:** Pulls the latest code from the repository
2. **Node.js Setup:** Configures Node.js version 20 with npm caching
3. **Dependency Installation:** Runs `npm ci` to install dependencies
4. **Application Build:** Executes `npm run build` to build the application
5. **AWS Configuration:** Sets up AWS credentials using provided access keys
6. **ECR Login:** Authenticates with Amazon ECR
7. **Docker Build:** Builds a Docker image tagged with the ECR repository name
8. **Push to ECR:** Pushes the built image to the ECR repository with the `latest` tag

#### B. Deploy Stage
1. **AWS Configuration:** Reconfigures AWS credentials
2. **ECR Login:** Reauthenticates with Amazon ECR
3. **jq Installation:** Installs jq for JSON processing
4. **Task Definition Download:** 
   - Retrieves the current task definition
   - Cleans metadata fields using jq
   - Saves as `task-definition.json`
5. **Task Definition Update:** 
   - Updates the container image reference in the task definition
   - Points to the newly pushed ECR image
6. **ECS Deployment:**
   - Deploys the updated task definition to the ECS service
   - Waits for service stability before completing

## Infrastructure Components

### 1. Container Registry (ECR)
- **Repository Name:** `hablist`
- **Image Tagging:** Uses mutable `latest` tag for simplicity in development
- **Access:** Accessed by ECS during deployment

### 2. Container Service (ECS)
- **Cluster:** `worthy-butterfly-8m6ir4`
- **Service:** `hablist-task-def-service-ucun9rgl`
- **Task Definition:** `hablist-task-def`
- **Container Name:** `hablist-container`

### 3. Database (RDS PostgreSQL)
- **Database Name:** `hablistdbdev`
- **Connection:** Accessed by the ECS task through the task definition configuration
- **Migration:** Data was migrated from Replit to this RDS instance

## Security Considerations


1. **ECR Access:**
   - Pipeline has permissions to push images to ECR
   - ECS has permissions to pull images from ECR

3. **Database Access:**
   - ECS tasks  have appropriate security groups to access RDS data base
   - Database credentials are managed through AWS Secrets Manager or environment variables check on Task definiion 
     

## Maintenance and Troubleshooting

### Common Issues
1. **Build Failures:**
   - Examine build logs for specific errors

2. **Deployment Failures:**
   - Verify ECR repository exists and is accessible
   - Check that the ECS service and cluster names are correct
   - Validate task definition has proper permissions and resource allocations

3. **Database Connection Issues:**
   - Verify RDS instance is running
   - Check security group rules allow traffic from ECS
   - Validate connection string and credentials in task definition

### Rollback Procedure
1. **Manual Rollback:**
   - Navigate to ECS console
   - Select the service
   - Click "Update" and select previous task definition revision
   - Deploy the older version

2. **Pipeline Rollback:**
   - Revert the code change in GitHub
   - Push to trigger a new deployment with previous working version

## Future Improvements

1. **Environment Separation:**
   - Implement separate dev/staging/prod environments
   - Use different branches or tags to trigger environment-specific deployments

2. **Immutable Tags:**
   - Consider using commit hashes or build numbers instead of `latest` tag
   - Improves traceability and rollback precision

3. **Infrastructure as Code:**
   - Use Terraform or AWS CDK to manage ECS/RDS infrastructure
   - Version control the infrastructure alongside application code

4. **Testing:**
   - Add unit/integration tests to run before deployment
   - Implement canary deployments or health checks

5. **Monitoring:**
   - Set up CloudWatch alarms for the ECS service
   - Implement logging for application and container metrics

## Appendix: Full Pipeline YAML

```yaml
name: ECS/Fargate Deployment Pipeline

on:
  push:
    branches: [ccodingwizard03-patch-1]

env:
  AWS_REGION: us-east-1
  ECR_REPOSITORY: hablist
  ECS_CLUSTER: worthy-butterfly-8m6ir4
  ECS_SERVICE: hablist-task-def-service-ucun9rgl
  CONTAINER_NAME: hablist-container
  TASK_DEFINITION_FAMILY: hablist-task-def
  AWS-ACCESS_KEY_ID: '#################'
  AWS-ACCESS_SECRET_ACCESS_KEY_ID: '##########################'

jobs:
  build:
    name: Build and Push Docker Image
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'

      - name: Install Dependencies
        run: npm ci

      - name: Build Application
        run: npm run build

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ env.AWS-ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ env.AWS-ACCESS_SECRET_ACCESS_KEY_ID }}
          aws-region: us-east-1

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build Docker image
        run: |
          docker build -t ${{ env.ECR_REPOSITORY }} .
          docker tag hablist:latest \
            ${{ steps.login-ecr.outputs.registry }}/hablist:latest

      - name: Push Docker image to ECR
        run: |
          docker push ${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:latest

  deploy:
    name: Deploy to ECS
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ env.AWS-ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ env.AWS-ACCESS_SECRET_ACCESS_KEY_ID }}
          aws-region: us-east-1

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Install jq
        run: sudo apt-get install -y jq

      - name: Download and clean task definition
        id: download-task-def
        run: |
          aws ecs describe-task-definition \
            --task-definition ${{ env.TASK_DEFINITION_FAMILY }} \
            --query taskDefinition \
            --region ${{ env.AWS_REGION }} \
            | jq 'del(.enableFaultInjection, .taskDefinitionArn, .revision, .status, .requiresAttributes, .compatibilities, .registeredAt, .registeredBy)' \
            > task-definition.json
          echo "::set-output name=task-definition::task-definition.json"

      - name: Update task definition
        id: update-task-def
        uses: aws-actions/amazon-ecs-render-task-definition@v1
        with:
          task-definition: ${{ steps.download-task-def.outputs.task-definition }}
          container-name: ${{ env.CONTAINER_NAME }}
          image: ${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:latest

      - name: Deploy to ECS Service
        uses: aws-actions/amazon-ecs-deploy-task-definition@v1
        with:
          task-definition: ${{ steps.update-task-def.outputs.task-definition }}
          service: ${{ env.ECS_SERVICE }}
          cluster: ${{ env.ECS_CLUSTER }}
          wait-for-service-stability: true
```
