# WordPress Sites Deployment with Azure DevOps YAML Pipelines

This repository contains the YAML pipeline configuration for deploying WordPress sites using Azure DevOps. The deployment process has been migrated from classic pipelines to YAML for better version control and flexibility.

## Pipeline Overview
The pipeline is divided into three main stages:

1. Build Package: This stage builds the WordPress artifact and performs necessary checks.
2. Deploy to UAT: This stage deploys the built artifact to the UAT environment.
3. Deploy to Production: This stage deploys the built artifact to the production environment.

## Pipeline Configuration
### Trigger
The pipeline is triggered on changes to the master branch.

````

trigger:
- master

````
