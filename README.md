**AWS Infrastructure Deployment with CI/CD**

**Description**

This project provisions AWS infrastructure using Terraform to support a containerized application, with a CI/CD pipeline managed through GitHub Actions workflows. 
The infrastructure includes an ECR to store Docker images and an ECS Fargate instance to run the application.

**Architecture Overview**

The project provisions the following AWS resources:

Elastic Container Registry (ECR): Stores Docker images for the application.

ECS Fargate Cluster: Runs the application using Docker images stored in ECR.


**Installation and Setup**

1. Clone the Repository:

    git clone https://github.com/prajaktm03/poc-5.git

    cd poc-5

2. Prerequisites:

    AWS Account with necessary permissions for ECS, ECR, and IAM.

    Terraform installed locally.

3. Configure AWS Credentials: Set up AWS credentials for CLI access:

   aws configure

4. Deploy the Infrastructure with Terraform:
   terraform init
   terraform apply

**CI/CD Pipeline**

The CI/CD pipeline automates the build, deployment, and updates of the application on AWS. 
GitHub Actions handles these tasks using two main jobs: build and deploy.
   

**Workflow Details**

CI/CD Pipeline YAML

name: CI/CD Pipeline

on:
  #workflow_dispatch:  # Allow manual trigger of the deployment workflow
  push:
    branches:
      - main
  

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1  # Replace with your AWS region

      - name: List files in the app directory
        run: ls -la ./App  # Adjust this path if needed

      - name: Install Node.js dependencies
        run: npm install
        working-directory: ./App  # Adjust this if package.json is in a different directory

      - name: Log in to Amazon ECR
        id: ecr-login
        uses: aws-actions/amazon-ecr-login@v1
        with:
          region: ${{ secrets.AWS_REGION }}  # Region should be passed here

      - name: Build, Tag, and Push Docker Image
        env:
          ECR_REPOSITORY: demo-app-ecr-repo  # Replace with your ECR repo name
          IMAGE_TAG: ${{ github.sha }}

        run: |
          # Build the Docker image
          docker build -t $ECR_REPOSITORY:$IMAGE_TAG ./App
          # Tag the image for ECR with the 'latest' tag
          docker tag $ECR_REPOSITORY:$IMAGE_TAG ${{ steps.ecr-login.outputs.registry }}/$ECR_REPOSITORY:latest
          # Tag the image for ECR with the SHA tag
          docker tag $ECR_REPOSITORY:$IMAGE_TAG ${{ steps.ecr-login.outputs.registry }}/$ECR_REPOSITORY:$IMAGE_TAG
          # Push the images to ECR
          docker push ${{ steps.ecr-login.outputs.registry }}/$ECR_REPOSITORY:latest
          docker push ${{ steps.ecr-login.outputs.registry }}/$ECR_REPOSITORY:$IMAGE_TAG

  deploy:
    runs-on: ubuntu-latest
    needs: build  # Ensure this job runs only after the build job

    steps:
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1

      - name: Get Latest Image URI
        id: get_image_uri
        run: |
          IMAGE_TAG=$(aws ecr describe-images --repository-name demo-app-ecr-repo \
                    --query 'sort_by(imageDetails, &imagePushedAt)[-1].imageTags[0]' --output text)
          if [ -z "$IMAGE_TAG" ]; then
            echo "Error: Unable to fetch the latest image tag."
            exit 1
          fi
          IMAGE_URI="${{ secrets.AWS_ACCOUNT_ID }}.dkr.ecr.us-east-1.amazonaws.com/demo-app-ecr-repo:$IMAGE_TAG"
          echo "LATEST_IMAGE_URI=$IMAGE_URI" >> $GITHUB_ENV

      - name: Describe Current ECS Task Definition
        id: describe_task
        run: |
          TASK_DEFINITION=$(aws ecs describe-task-definition --task-definition demo-app-task)
          if [ $? -ne 0 ]; then
            echo "Error: Unable to describe the task definition."
            exit 1
          fi
          echo "$TASK_DEFINITION" > current_task_definition.json

      - name: Create New Task Definition JSON
        id: create_task_def
        run: |
          # Create a new task definition based on the current one, ensuring to filter out unwanted parameters
          NEW_TASK_DEF=$(jq --arg IMAGE_URI "${{ env.LATEST_IMAGE_URI }}" \
            '.taskDefinition | 
            .containerDefinitions[0].image=$IMAGE_URI |
            del(.revision, .taskDefinitionArn, .status, .requiresAttributes, .compatibilities, .registeredAt, .registeredBy)' \
            current_task_definition.json)
          echo "$NEW_TASK_DEF" > new_task_definition.json

      - name: Register New ECS Task Definition
        id: register_task_def
        run: |
          TASK_DEFINITION_ARN=$(aws ecs register-task-definition --cli-input-json file://new_task_definition.json \
                                --query "taskDefinition.taskDefinitionArn" --output text)
          if [ $? -ne 0 ]; then
            echo "Error: Failed to register new task definition."
            exit 1
          fi
          echo "TASK_DEFINITION_ARN=$TASK_DEFINITION_ARN" >> $GITHUB_ENV

      - name: Deploy New Task Definition to ECS Service
        run: |
          aws ecs update-service \
            --cluster demo-app-cluster \
            --service cc-demo-app-service \
            --force-new-deployment \
            --task-definition ${{ env.TASK_DEFINITION_ARN }}
          if [ $? -ne 0 ]; then
            echo "Error: Failed to update ECS service."
            exit 1
          fi



**Workflow Steps**

1. Build Job:

Checkout code: Fetches the latest code from the repository.
Configure AWS Credentials: Sets up access to AWS using stored secrets.
List files and Install Dependencies: Lists directory contents and installs necessary Node.js dependencies.
Log in to ECR and Build Docker Image: Logs into Amazon ECR, builds the Docker image, and pushes it with a unique tag based on the commit SHA.

2. Deploy Job (runs only if the Build job completed successfully.):

Fetch Latest Image URI: Retrieves the latest image tag from ECR.
Describe and Create New Task Definition: Copies the existing ECS task definition and updates it with the latest image URI.
Register and Deploy New Task Definition: Registers the updated task definition and forces a new deployment to ECS.



**Environment Variables and Secrets**

Set these in the repository’s Secrets settings:

AWS_ACCESS_KEY_ID and AWS_SECRET_ACCESS_KEY

AWS_REGION

AWS_ACCOUNT_ID

**Usage**

After setting up, you can manually trigger the workflow from the Actions tab in GitHub. The build job will run first, followed by the deploy job, updating ECS with the latest Docker image.

**Configuration**

Adjust configurations in:

Terraform scripts: Update AWS region, ECR repository, and ECS details in main.tf.

GitHub Actions workflow: Customize environment variables or other parameters in ci.yml.
