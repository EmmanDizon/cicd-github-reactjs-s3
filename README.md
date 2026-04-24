## CI/CD Setup Guide: React + GitHub Actions + AWS S3

This documentation teaches you how to automatically deploy your react app to s3 via github actions.

### Prerequisites

This step is for those who already have AWS organization (dev, staging and so on). If you don't have yet, set it up first. Check out my repo `aws-organization-instruction`

---

## 1. Access & Authentication (NO ROOT USAGE)

### 1.1 Enable IAM Identity Center (SSO)

From the AWS account you'll use as DEV:

1. Go to **IAM Identity Center**
2. Click **Enable**
3. Choose **AWS managed directory**
4. Wait until status is **Enabled**

### 1.2 Create an SSO User

1. Navigate to **IAM Identity Center → Users**
2. Click **Add user**
3. Fill in:
   - Username
   - Email
   - First name / Last name
4. Skip groups
5. Create user

### 1.3 Create Permission Set

1. Go to **IAM Identity Center → Permission sets**
2. Create permission set
3. Choose **Predefined**
4. Select: **AdministratorAccess**

### 1.4 Assign User to AWS Account

1. Navigate to **IAM Identity Center → AWS accounts**
2. Select your DEV account
3. Click **Assign users**
4. Choose your user
5. Assign permission set: **AdministratorAccess**

### 1.5 Login URL (IMPORTANT)

Your AWS login from now on is:

```
https://<directory-id>.awsapps.com/start
```

**Bookmark this.** You no longer use root or IAM users.

---

## 2. GitHub Actions → AWS (OIDC Setup)

**Goal:** Allow GitHub Actions to deploy to AWS without storing AWS credentials.

### 2.1 Create OIDC Provider (AWS side)

1. Go to **AWS Console → IAM → Identity providers**
2. Click **Add provider**
3. Configure:
   - **Type:** OpenID Connect
   - **Provider URL:** `https://token.actions.githubusercontent.com`
   - **Audience:** `sts.amazonaws.com`

### 2.2 Create IAM Role for GitHub Actions

1. Go to **AWS Console → IAM → Roles**
2. Click **Create role**
3. Configure:
   - **Trusted entity type:** Web identity
   - **Identity provider:** `token.actions.githubusercontent.com`
   - **Audience:** `sts.amazonaws.com`
   - **Role name:** `your-role-name`

### 2.3 Configure Trust Policy (CRITICAL)

Edit **Trust relationships** of role `your-role-name`.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::YOUR_ACCOUNT_ID:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:GITHUB_USERNAME/GITHUB_REPOSITORY:*"
        }
      }
    }
  ]
}
```

The `sub` must match your GitHub repo.

**Format:**

```
repo:<GITHUB_USERNAME>/<GITHUB_REPOSITORY>:*
```

**Example:**

```
repo:EmmanDizon/cicd-github-reactjs-s3:*
```

This allows GitHub Actions from that repo to assume the role.

> You can later restrict this to a specific branch (e.g. `develop`).

### 2.4 Attach Permissions Policy to Role

Go to **IAM → Roles → your-role-name → Permissions** tab.

Click **Add permissions → Create inline policy** → **JSON** tab.

Paste this IAM policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:PutObject", "s3:GetObject", "s3:DeleteObject", "s3:ListBucket"],
      "Resource": ["arn:aws:s3:::YOUR_BUCKET_NAME", "arn:aws:s3:::YOUR_BUCKET_NAME/*"]
    }
  ]
}
```

**Policy name:** `S3-InventoryAppFE-DeployAccess`

**Replace:** `YOUR_BUCKET_NAME` with your actual S3 bucket name.

**Purpose:** This gives the GitHub Actions role permission to upload, read, delete, and list files in your S3 bucket.

---

## 3. GitHub Repository Setup

### 3.1 Create Workflow Directory

In your root repository:

```
.github/
  └── workflows/
```

### 3.2 Create Workflow Files

Create workflow files such as:

- `ci-pr.yml` (for pull request checks)
- `deploy-dev.yml` (deploy on merge to develop)

#### **ci-pr.yml** - Pull Request Checks

Create `.github/workflows/ci-pr.yml`:

```yaml
# PURPOSE: This workflow runs AUTOMATICALLY when you create a Pull Request.
# It checks if your code is valid BEFORE merging into develop branch.
# This prevents broken code from being deployed.

name: CI - Pull Request

on:
  pull_request:
    branches:
      - develop

jobs:
  build-and-test:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Lint code
        run: npm run lint

      - name: Build application
        run: npm run build
```

#### **deploy-dev.yml** - Deploy to S3

Create `.github/workflows/deploy-dev.yml`:

```yaml
# PURPOSE: This workflow runs AUTOMATICALLY after you merge/push to develop branch.
# It builds your React app and deploys it to AWS S3.
# This is SEPARATE from ci-pr.yml (which only checks code quality).

name: Deploy to DEV

on:
  push:
    branches:
      - develop

permissions:
  id-token: write # Required for OIDC
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Build application
        run: npm run build

      - name: Configure AWS credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ vars.DEV_AWS_ROLE_ARN }}
          aws-region: ${{ vars.AWS_REGION }}

      - name: Deploy to S3
        run: |
          aws s3 sync dist/ s3://${{ vars.DEV_S3_BUCKET }} --delete

      - name: Deployment complete
        run: |
          echo "✅ Deployed to: http://${{ vars.DEV_S3_BUCKET }}.s3-website-${{ vars.AWS_REGION }}.amazonaws.com"
```

### 3.3 GitHub Actions Variables

Navigate to: **GitHub repo → Settings → Secrets and variables → Actions → Variables**

Add the following variables:

| Variable           | Value                                           |
| ------------------ | ----------------------------------------------- |
| `DEV_AWS_ROLE_ARN` | `arn:aws:iam::<ACCOUNT_ID>:role/your-role-name` |
| `AWS_REGION`       | `us-east-1`                                     |
| `DEV_S3_BUCKET`    | `react-deployment-bucket-test`                  |

**Note:** No AWS keys should exist in GitHub.

---

## 4. S3 Setup for React Static Hosting

### 4.1 Create S3 Bucket

- **Bucket name:** `react-deployment-bucket-test`
- **Region:** same as GitHub Actions region
- **Disable** Block all public access

### 4.2 Enable Static Website Hosting

1. Go to **S3 Bucket → Properties**
2. Enable **Static website hosting**
3. Configure:
   - **Index document:** `index.html`
   - **Error document:** `index.html` _(Required for React routing)_

### 4.3 Add Bucket Policy (Public Read)

Go to **S3 → Your bucket → Permissions → Bucket policy**.

Click **Edit** and paste this bucket policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadGetObject",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::YOUR_BUCKET_NAME/*"
    }
  ]
}
```

**Replace:** `YOUR_BUCKET_NAME` with your actual S3 bucket name.

**Purpose:** This makes your React app files publicly accessible so users can view your website.

### 4.4 Public Access URL

From Static website hosting, use the provided endpoint:

```
http://<bucket-name>.s3-website-<region>.amazonaws.com
```

This is your live React app.
