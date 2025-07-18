# Deploy a Node.js App on ECS using GitHub Actions with OIDC role

The following lines will guide you through using the GitHub Actions pipeline to deploy a Node.js application on AWS. The pipeline uses AWS OIDC Roles for secure and passwordless authentication.

## Overview on OIDC

### Definition

**OIDC (OpenID Connect)** is an authentication protocol that verifies the identity of users when they sign in to access digital services. 

OIDC is used to securely connect AWS with external identity providers (like GitHub, Google, or your company’s SSO).

### Key components of OIDC
There are six primary components in OIDC:

1. **Authentication** – Confirms who the user is (verifies the user's identity).

2. **Client** – The app or website asking for user identity info.

3. **Relying Party (RP)** – The app that trusts an identity provider to authenticate users.

4. **Identity Token** – A secure message that contains user identity and authentication details.

5. **OpenID Provider** – A trusted service (like Google) that verifies users and issues identity tokens.

6. **User** – The person or service trying to access an app without creating a new account.


In the context of this project, we can see the various components on the following diagram: 

![image](https://github.com/user-attachments/assets/c4b59f69-7ac6-443d-9238-d922a99a0408)

### Basic OIDC Features in This Project

**1. Secure Identity Verification:** GitHub Actions proves its identity to AWS using OIDC — no AWS keys needed.

**2. No Secret Storage:** We don’t store AWS credentials in GitHub. OIDC gives GitHub a short-lived token at runtime.

**3. Short-Lived ID Tokens:** GitHub gets a temporary token that AWS trusts to allow actions like deploying to ECS.

**4. Trusted Login Flow:** AWS acts as the relying party and only allows access if the token comes from GitHub's trusted OIDC provider.

## Repository structure

This repository contains the Node.js application that will be deployed in AWS. Here is a brief description of the overall structure:
- **.github/workflows** directory: contains the Github Actions workflow that will be use to automate the provisioning of the various components of the infrastructure and the deployment the application.
  
- **backend:** Contains the code for the backend of the application with the corresponding Dockerfile

- **frontend:** Contains the code for the frontend of the app (Reactjs) with the corresponding Dockerfile

- **infra:** Contains the Terraform code to deploy the infrastructure that will host the application in AWS

- **ecs-deployment:** Contains the Terraform code used to deploy the ECS tasks definitions and services for the app


## Steps to deploy the app

### Prerequisites
Before starting this project, you need to have the following:

- Access to an AWS account with permissions to create IAM roles and identity providers.
- A GitHub repository where this role will be used. You will need to clone this repository and push its content to your own github repository
- Basic knowledge of GitHub Actions and AWS IAM.

### Step 1: Create the OIDC Identity Provider in AWS

1. Sign in to the AWS Management Console.
2. Navigate to **IAM** > **Identity Providers**.
3. Click **Add provider**.
4. Set the **Provider type** to `OIDC`.
5. Choose **OpenID Connect**.
6. For **Provider URL**, enter:
   ```
   https://token.actions.githubusercontent.com
   ```
7. For **Audience**, enter:
   ```
   sts.amazonaws.com
   ```
8. Click **Add provider** to save.

### Step 2: Create an AWS IAM role for Github Actions

Here we need to create an IAM Role for Github Actions, with appropriate permissions to manage ECR, ECS, and Terraform.

The role will trust GitHub’s OIDC provider to allow workflows from the repository to assume the role.

To achieve this:

1. In your AWS console, go to **IAM** > **Roles** click **Create role**.
2. Select **Web identity** as the trusted entity type.
3. Choose the OIDC provider you created: `token.actions.githubusercontent.com`.
4. Set the **Audience** to `sts.amazonaws.com`.
5. Replace:
   * `<OWNER>` with your GitHub username or organization: (for us we use `utrains`)
   * `<REPO>` with your repository name: Deploy-a-NodeJS-Application-on-AWS-Using-GithubActions
   * `<BRANCH>` with the branch name: main (put your branch name)

Click **Next** to proceed.

6. Attach IAM Permissions

Attach the following AWS managed policies to the role:
* `AmazonEC2ContainerRegistryFullAccess`
* `AmazonEC2ContainerRegistryPowerUser`
* `AmazonEC2FullAccess`
* `AmazonECS_FullAccess`
* `IAMFullAccess`

Then click **Next**.


7. Name the Role

* **Role name**: `github-actions-oidc-role`
* Add tags as needed (optional)
* Click **Create role**

8. Save the Role ARN

After creation, copy the **Role ARN** from the role summary page. It will look like this:
```
arn:aws:iam::123456789012:role/github-actions-oidc-role
```

### Step 3: Configure GitHub Actions to use the IAM role

#### 1. Configure the AWS role ARN as secret in the Github repository

- In your GitHub repository, go to **Settings** > **Secrets and variables** > **Actions**.
- Click on Add a new secret:

   * Name: `AWS_ROLE_ARN`
   * Value: Paste the Role ARN you copied earlier.

#### 2. Set the secret (AWS_ROLE_ARN) in your GitHub Actions workflow like this:

```yaml
name: Deploy to AWS

on:
  push:
    branches:
      - main

permissions:
  id-token: write
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: us-east-1

      - name: Display caller identity
        run: aws sts get-caller-identity
```

### Step 4: Configure Manual Approval in GitHub Actions 

In our Github workflow we have final steps that should be executed only when we want to destroy the infrastructure.

In some CI/CD scenarios when dealing with destructive infrastructure operations like tearing down environments, it's important to require **manual approval** before execution. 

GitHub Actions provides this functionality via **Environments** with required reviewers.

Here is how to implement manual approval step-by-step:

#### 1. Create an Environment with Required Reviewers

* In your GitHub repository, go to **Settings > Environments**.
* Create a new environment named `destroy-approval`.
* Under **Deployment protection rules**, add required reviewers (your GitHub username or a team).
* Save the environment.

> This ensures any job using this environment will pause for approval before execution.

#### 2. Specify the environment created in the manual approval jobs of the workflow

In our workflow, we have 2 jobs that destroy the ressources in AWS. 
- The job `destroy-ecs`: which destroys all the resources created by the job `Apply-the-terraform-code-to-Launch-the-frontend-and-the-backend-app`
- The job `destroy-infra`: which destroys the infra created by the job `Create-the-infrastructure-in-AWS`

We created an intermediate job before each of these jobs to act as manual checkpoints. These jobs should be tied to the environment we created for Github actions to pause the workflow until a reviewer clicks on the **Approve deployment** button.

Set the Environment in the jobs as follows (do the same for the second intermediate job)

```yaml
wait-for-ecs-destroy-approval:
  runs-on: ubuntu-latest
  needs: Apply-the-terraform-code-to-Launch-the-frontend-and-the-backend-app
  environment:
    name: destroy-approval  # Triggers manual approval
  steps:
    - name: Wait for Approval
      run: echo "Waiting for manual approval to destroy resources"
```

This job will not run until a GitHub reviewer approves it in the Actions tab.

**Note: The destroy jobs depend on these intermediate jobs.**

##### Destroy Job

```yaml
destroy-ecs:
  runs-on: ubuntu-latest
  needs: wait-for-ecs-destroy-approval
  permissions:
    id-token: write
    contents: read

  steps:
    - name: Checkout Code
      uses: actions/checkout@v3

    - name: Setup Terraform
      uses: hashicorp/setup-terraform@v2
      with:
        terraform_version: 1.5.0

    - name: Install AWS CLI
      uses: unfor19/install-aws-cli-action@v1
      with:
        version: 2
        verbose: false
        arch: amd64

    - name: Configure AWS Credentials Using OIDC
      uses: aws-actions/configure-aws-credentials@v4
      with:
        role-to-assume: ${{ env.OIDC_ROLE_ARN }}
        aws-region: ${{ env.AWS_REGION }}

    - name: Download Artifact
      uses: actions/download-artifact@v4
      with:
        name: ecs-terraform-state-file

    - name: Use the terraform-state artifact
      run: |
        terraform init -input=false
        terraform refresh
        terraform destroy -auto-approve
```

## Step 5: Pipeline Environment Variables

Hardcoding sensitives values in the pipeline is not a good practice so we will need to set environment variables in the workflow through GitHub repository settings instead.
In this workflow, the following environment variables are set globally:

  * `AWS_REGION`
  * `FRONTEND_REPO`
  * `BACKEND_REPO`
  * `AWS_ROLE_ARN`
  * `TAG`
        
 - Replace the AWS Account ID and role ARN with your own values.

### 1. Go to Your Repository Settings

* Open your repository on GitHub.
* Click the **Settings** tab at the top right.

### 2. Define the Variables

* In the left sidebar, click on:

  * **Secrets and variables > Actions** (recommended for GitHub Actions secrets and variables)
  * OR just **Secrets** if your interface doesn’t show the new Variables section yet.
* Click on **New repository variable** for non-sensitive info like region or repo URLs.
  * **Name**: e.g., `AWS_REGION`
  * **Value**: e.g., `us-east-2`
* Click **Add secret** or **Add variable**.

Repeat for: the Others variables 

*Note:* We will leave `TAG` it in the your workflow but it can also be put  here.

### Step 6: Execute the pipeline and test the application

#### 1. Execute the pipeline

For the pipeline to start execution, you just need to commit and push modifications. You can also trigger the workflow manually.

![image](https://github.com/user-attachments/assets/8713b72c-cb36-4109-8ccc-576305b0fce1)

The workflow will pause at the `wait-for-ecs-destroy-approval` job.

When it's done you must have something like this :

![image](https://github.com/user-attachments/assets/8bf28e6f-42df-42ea-b78f-83fb73b0d258)


#### 2. Test the application

To verify that the app is working properly, you will need to enter the URL of your frontend in the browser.

The URL will be something like: http://node-frontend-lb-XXXXXXXX.us-east-2.elb.amazonaws.com:3000/

you can see it at bthe end of the step *Use the ecs-apply-plan.tfplan artifact* in the job *Apply-the-terraform-code-to-Launch-the-frontend-and-the-backend-app* 

![image](https://github.com/user-attachments/assets/3c4d47b7-2f4f-40a2-8c39-7af19a15bd87)


The expected result should be like the one bellow and when you refresh you will see that the value has changed:

![image](https://github.com/user-attachments/assets/83eda7e9-8d76-4b11-815f-eaf79e74d868)


## Clean up

When done testing the app, we will destroy the whole infrastructure to avoid recurrent charges in AWS.

1. In your github repo, go to **Actions > Workflow Run > Approval Job**.
2. Click on **Review deployments**.
2. Click on **Approve and deploy**.

Once approved, the `destroy-ecs` job will run.


![image](https://github.com/user-attachments/assets/7d8d7233-d997-431f-8ec7-2dbfda20f6b1)



## Quick note on Terraform state file transfer from one job to another (Optional)
Before defining the destruction job, we need to understand this basics:
- When Terraform is used to automate the provisioning of ressources, a terraform state file is generated.
- The Terraform state file keeps a record of everything it created, so Terraform uses it to know what to delete when destroying the infrastructure.
- Thus, we need to find a way to transfer that file from one job (creation job) to the other (destruction job).

**Note: In the context of this project, we did not use a remote backend to store the state files.**

In our Github Action workflow, we had two jobs that generated a terraform state file.  
- The `Create-the-infrastructure-in-AWS` job to create the infrastructure
- The `Apply-the-terraform-code-to-Launch-the-frontend-and-the-backend-app` job to launch the app on the infrastructure.

The 2 terraform state files were then saved as artifact (verify the steps of the jobs) in order to transfer them to the destroy jobs.

**Note: Go through the workflow to better understand**
