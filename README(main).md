# **Production Deployment Pipeline Documentation**  
*(Main Branch - Secure Workflow)*  

---

## **1. Overview**  
This document describes the **production-grade CI/CD pipeline** for deploying the application to AWS ECS. Unlike the development branch (`ccodingwizard03-patch-1`), this workflow follows a **secure, two-phase approach**:  

1. **Build & Scan Phase** (`build-push.yml`)  
   - Code is built, scanned for vulnerabilities, and pushed to **ECR**.  
   - Requires manual approval before deployment.  

2. **Deployment Phase** (`deploy-ecs.yml`)  
   - Only triggers after a successful build and manual approval.  
   - Deploys the approved image to **ECS Fargate** in production.  

🔹 **Key Security Improvements Over Development Branch:**  
✔ **Separate AWS IAM roles** (no shared access)  
✔ **Manual approval required** before ECS deployment  
✔ **Isolated infrastructure** (no shared ECR, ECS, RDS, or security groups)  
✔ **GitHub Secrets** used instead of hardcoded credentials  

---

## **2. Pipeline Architecture**  

### **Workflow 1: Build & Push to ECR (`build-push.yml`)**  
*(Trigger: Push to `main` branch)*  

| Step | Action | Purpose |
|------|--------|---------|
| 1. | **Code Checkout** | Pulls latest `main` branch code |
| 2. | **Security Scanning** | Runs SAST (Static Application Security Testing) |
| 3. | **Docker Build** | Creates a production-ready image |
| 4. | **Push to ECR** | Stores image in `hablist-production` (only keeps **1 latest image**) |  

**Approval Required:**  
✅ Before proceeding to deployment  

---

### **Workflow 2: Deploy to ECS (`deploy-ecs.yml`)**  
*(Trigger: Manual approval after `build-push.yml` succeeds)*  

| Step | Action | Purpose |
|------|--------|---------|
| 1. | **AWS Auth** | Uses GitHub Secrets (`secrets.AWS_ACCESS_KEY_ID`) |
| 2. | **Fetch ECR Image** | Pulls the approved image from `hablist-production` |
| 3. | **Update ECS Task Definition** | Modifies `hablist-task-production` |
| 4. | **Deploy to ECS** | Updates `hablist-task-production-service-6opjtvhv` |

---

## **3. Production Infrastructure (Currently Paused)**  

### **AWS Resources**  
| Resource | Name | Status |
|----------|------|--------|
| **ECR** | `hablist-production` | ✅ **Active** (1 image retained) |
| **ECS Cluster** | `prod-cluster-hablist` | ⏸️ **Stopped** (Cost-saving) |
| **ECS Service** | `hablist-task-production-service-6opjtvhv` | ⏸️ **Stopped** |
| **Task Definition** | `hablist-task-production` | ⏸️ **Stopped** |
| **RDS PostgreSQL** | `hablistdb-production` | ⏸️ **Stopped** (Will resume for prod) |

### **Security Isolation**  
🔒 **No shared resources with development branch:**  
- ❌ Different ECR repo (`hablist-production` vs `hablist`)  
- ❌ Different ECS cluster (`prod-cluster-hablist` vs `worthy-butterfly-8m6ir4`)  
- ❌ Different IAM roles (separate AWS credentials)  (`main-branch-deployer`vs `hablist-ecs-ecr-access`)
- ❌ Different RDS instance (`hablistdb-prod` vs `hablistdbdev`)  

---

## **4. How to Resume Production Deployment**  

### **Steps to Activate ECS & RDS**  
1. **Start RDS Instance:**  
   ```bash
   aws rds start-db-instance --db-instance-identifier hablistdb-prod --region us-east-1
   ```
2. **Start ECS Service:**  
   ```bash
   aws ecs update-service --cluster prod-cluster-hablist --service hablist-task-production-service-6opjtvhv --desired-count 1 --region us-east-1
   ```
3. **Trigger Deployment Manually:**  
   - Go to GitHub Actions → `deploy-ecs.yml` → **Run Workflow**  

---

## **5. Security Best Practices**  

### **GitHub Secrets (Under `main` Environment)**  
| Secret Name | Purpose |
|-------------|---------|
| `AWS_ACCESS_KEY_ID` | Grants deployment access to ECS/ECR |
| `AWS_SECRET_ACCESS_KEY` | Securely authenticates AWS API calls |



## **6. Troubleshooting & Rollback**  

### **If Deployment Fails:**  
1. **Check ECS Logs:**  
   ```bash
   aws logs tail /ecs/hablist-task-production --region us-east-1
   ```
2. **Revert ECS Task Definition:**  
   - Manually select an older revision in AWS Console.  

### **If RDS Issues Occur:**  
- Verify security group allows ECS traffic  
- Check CloudWatch for PostgreSQL errors  

---

## **7. Future Improvements**  
🔹 **Automated Rollback** (If health checks fail)  
🔹 **Blue/Green Deployments** (Zero-downtime updates)  
🔹 **Database Backups Before Deployment**  

---
