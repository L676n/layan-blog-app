# MERN Stack Blog App Deployment on AWS

## Overview

This document outlines the steps to deploy a MERN (MongoDB, Express, React, Node.js) stack blog application on AWS using free-tier eligible services. The backend (Express) runs on EC2, the frontend (React) is hosted via S3 static website hosting, and MongoDB Atlas is used as the database. Media uploads are handled through a separate S3 bucket with appropriate IAM permissions.

## Technologies Used

*   **Frontend:** React
*   **Backend:** Express, Node.js
*   **Database:** MongoDB Atlas
*   **Hosting:** AWS (EC2, S3)
*   **Node Version Manager:** nvm

## Prerequisites

*   An AWS account
*   A MongoDB Atlas account
*   Basic knowledge of AWS services (EC2, S3, IAM)
*   Familiarity with Git, Node.js, and npm

## Deployment Steps

### Part 1: MongoDB Atlas Configuration (Optional)

1.  Create a MongoDB Atlas account and set up a free cluster.
2.  Allow the EC2 instance IP in the network access settings.
3.  Create a database user and obtain the connection string.
    *   **Note:** You can also use the existing MongoDB connection string in the backend `.env` file.

### Part 2: S3 Bucket for Frontend

1.  Create an S3 bucket named `yourname-blogapp-frontend` in the `eu-north-1` region.
2.  Disable "Block all public access" for the bucket.
3.  Enable static website hosting.
4.  Add a bucket policy for public access:

    ```json
    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Principal": "*",
                "Action": "s3:GetObject",
                "Resource": "arn:aws:s3:::yourname-blogapp-frontend/*"
            }
        ]
    }
    ```

### Part 3: S3 for Media Uploads

1.  Create a second S3 bucket: `yourname-blogapp-media`.
2.  Disable "Block all public access".
3.  Configure CORS for browser upload support:

    ```json
    [
        {
            "AllowedHeaders": ["\*"],
            "AllowedMethods": ["GET", "PUT", "POST", "DELETE", "HEAD"],
            "AllowedOrigins": ["\*"],
            "ExposeHeaders": ["ETag"]
        }
    ]
    ```

### Part 4: IAM User and Policy for S3 Media Bucket Access

1.  Go to IAM Console > Users > Add users.
2.  Username: `blog-app-user`.
3.  Permissions > Attach existing policies directly.
4.  Create a custom policy with the following JSON (replace `yourname-blogapp-media` with your actual bucket name):

    ```json
    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Action": [
                    "s3:PutObject",
                    "s3:GetObject",
                    "s3:DeleteObject",
                    "s3:ListBucket"
                ],
                "Resource": [
                    "arn:aws:s3:::yourname-blogapp-media",
                    "arn:aws:s3:::yourname-blogapp-media/*"
                ]
            }
        ]
    }
    ```

5.  Attach this policy to the user.
6.  **Important:** Create and save the Access Key ID and Secret Access Key for this user. **Do NOT share these credentials in your submission or public repository!**

### Part 5: EC2 Backend Setup

1.  Launch a `t3.micro` instance in the `eu-north-1` region using Ubuntu 22.04 LTS.
2.  Allow incoming traffic on necessary ports (SSH: 22, HTTP: 80, HTTPS: 443, Custom TCP: 5000).  Add this Inbound rule:

    | Type       | Protocol | Port Range | Source    | Description          |
    | ---------- | -------- | ---------- | --------- | -------------------- |
    | Custom TCP | TCP      | 5000       | 0.0.0.0/0 | Allow public traffic |

3.  SSH into the instance and run the User Data script:

    ```bash
    #!/bin/bash
    apt update -y
    apt install -y git curl unzip tar gcc g++ make unzip

    su - ubuntu << 'EOF'
    export NVM_DIR="$HOME/.nvm"
    curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash
    source "$NVM_DIR/nvm.sh"
    nvm install --lts
    nvm use --lts
    npm install -g pm2
    EOF

    # MongoDB Shell
    curl -L https://downloads.mongodb.com/compass/mongosh-2.1.1-linux-x64.tgz -o mongosh.tgz
    tar -xvzf mongosh.tgz
    mv mongosh-*/bin/mongosh /usr/local/bin/
    chmod +x /usr/local/bin/mongosh
    rm -rf mongosh*

    # AWS CLI
    curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
    unzip awscliv2.zip
    ./aws/install
    rm -rf aws awscliv2.zip
    ```

4.  Clone the MERN app:

    ```bash
    git clone https://github.com/cw-barry/blog-app-MERN.git /home/ubuntu/blog-app
    cd /home/ubuntu/blog-app
    ```

5.  Configure the backend:

    ```bash
    cd backend
    cat > .env << EOF
    PORT=5000
    HOST=0.0.0.0
    MODE=production
    MONGODB=mongodb+srv://test:qazqwe123@mongodb.txkjsso.mongodb.net/blog-app
    JWT_SECRET=$(openssl rand -hex 32)
    JWT_EXPIRE=30min
    JWT_REFRESH=$(openssl rand -hex 32)
    JWT_REFRESH_EXPIRE=3d
    AWS_ACCESS_KEY_ID=<access-key-from-step-6>
    AWS_SECRET_ACCESS_KEY=<secret-key-from-step-6>
    AWS_REGION=eu-north-1
    S3_BUCKET=<your-s3-media-bucket-name>
    MEDIA_BASE_URL=https://<your-s3-media-bucket-name>.eu-north-1.amazonaws.com
    DEFAULT_PAGINATION=20
    EOF
    ```

    **Important:** Replace placeholder data (especially AWS credentials and S3 bucket name) with your actual values. **Do NOT share your AWS\_ACCESS\_KEY\_ID and AWS\_SECRET\_ACCESS\_KEY in your submission or public repository!**

6.  Configure the frontend:

    ```bash
    cd ../frontend
    cat > .env << EOF
    VITE_BASE_URL=http://<your-ec2-dns>:5000/api
    VITE_MEDIA_BASE_URL=https://<your-s3-media-bucket-name>.eu-north-1.amazonaws.com
    EOF
    ```

    Replace `<your-ec2-dns>` with your EC2 instance's public DNS and `<your-s3-media-bucket-name>` with your S3 media bucket name.

7.  Configure AWS CLI:

    ```bash
    aws configure
    # Enter your AWS Access Key ID when prompted
    # Enter your AWS Secret Access Key when prompted
    # Set default region to eu-north-1
    # Set default output format to json
    ```

8.  Deploy the backend:

    ```bash
    cd /home/ubuntu/blog-app/backend
    npm install
    mkdir -p logs
    pm2 start index.js --name "blog-backend"
    pm2 startup
    sudo pm2 startup systemd -u ubuntu
    sudo env PATH=$PATH:/home/ubuntu/.nvm/versions/node/v22.15.0/bin /home/ubuntu/.nvm/versions/node/v22.15.0/lib/node_modules/pm2/bin/pm2 startup systemd -u ubuntu --hp /home/ubuntu
    pm2 save
    ```

9.  Build and deploy the frontend:

    ```bash
    cd /home/ubuntu/blog-app/frontend
    npm install -g pnpm@latest-10
    pnpm install
    pnpm run build
    aws s3 sync dist/ s3://<your-s3-frontend-bucket-name>/
    ```

    Replace `<your-s3-frontend-bucket-name>` with your S3 frontend bucket name.


## Deliverables

*   Screenshot of S3 public URL loading `index.html`
*   `curl -I` output showing `200 OK` for the S3 public URL
*   Screenshot of file upload to the media S3 bucket
*   Screenshot of the backend server running via pm2
*   Screenshot of the frontend S3 web page

## Cleanup

*   Stop the EC2 instance.
*   Remove IAM credentials.
*   Delete S3 buckets if no longer needed.
