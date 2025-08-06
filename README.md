# Hablist CI/CD Pipeline Documentation

This repository contains documentation for the Hablist application's deployment pipelines to AWS ECS. The system uses two distinct workflows for development and production environments.

## Pipeline Documentation

1. **[Simple Deployment (Development Branch)](README(simple%20deploy).md)**  
   - Automatic deployments from `ccodingwizard03-patch-1` branch
   - Direct deployment to ECS after build
   - Uses mutable ECR image tags

2. **[Production Deployment (Main Branch)](README(main).md)**  
   - Manual approval workflow
   - Security scanning before deployment
   - Isolated production infrastructure
   - Immutable ECR image tags

## Key Differences

| Feature          | Development Pipeline | Production Pipeline |
|------------------|----------------------|---------------------|
| Trigger          | Automatic on push    | Manual approval     |
| ECR Repository   | `hablist`            | `hablist-production`|
| Security Scans   | No                   | Yes                 |
| Infrastructure   | Shared resources     | Isolated resources  |

## Getting Started

To resume paused production resources (ECS/RDS), see the [Production Deployment documentation](README(main).md#how-to-resume-production-deployment).
