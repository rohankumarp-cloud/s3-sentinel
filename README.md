# S3-Sentinel: Automated Serverless-Style Data Pipeline on AWS
An automated serverless-style cloud data pipeline using AWS EC2, S3, IAM, and Python.
📖 Overview
S3-Sentinel is an automated cloud data pipeline built on Amazon Web Services (AWS). It uses an EC2 worker instance running a Python automation script to continuously monitor a Raw S3 bucket, process incoming files, and securely transfer the transformed output to a Processed S3 bucket.

This project demonstrates core cloud architecture, security best practices (IAM roles over hardcoded credentials), Python programming with boto3, and Linux system automation via cron.

🛠️ Tech Stack & Services Used
Cloud Provider: Amazon Web Services (AWS)

Compute: Amazon EC2 (Ubuntu 24.04 LTS)

Storage: Amazon S3 (Raw Bucket & Processed Bucket)

Security & Access Management: AWS IAM (Custom Least-Privilege Policies & Roles)

Language & Libraries: Python 3.12, boto3 (AWS SDK for Python)

Automation: Linux cron scheduler, Virtual Environments (venv)

🔒 Security Architecture (IAM Best Practices)
To avoid security risks like hardcoding AWS access keys inside the source code, S3-Sentinel leverages IAM Roles for EC2.

A custom IAM policy (S3-Sentinel-Worker-Policy) was created and attached to the EC2 instance profile, granting granular, least-privilege permissions:

Raw Bucket: s3:ListBucket, s3:GetObject, s3:DeleteObject

Processed Bucket: s3:ListBucket, s3:PutObject

⚙️ Step-by-Step Implementation Guide
1. S3 Bucket Configuration
Two buckets were provisioned via the AWS S3 Console:

Raw Data Bucket: s3-sentinel-raw (Ingestion zone for incoming unstructured files)

Processed Data Bucket: s3-sentinel-processed (Destination zone for transformed data)

2. EC2 Worker & Python Setup
Launched an Ubuntu 24.04 LTS EC2 instance and connected securely via SSH / PowerShell.

Created a dedicated Python virtual environment:

Bash
python3 -m venv venv
source venv/bin/activate
Installed the official AWS SDK for Python (boto3):

Bash
pip install boto3
3. Core Processing Pipeline Script (pipeline.py)
Developed a Python script that polls the raw bucket, applies a text transformation (converting content to uppercase and appending a pipeline header), uploads the result to the processed bucket, and deletes the original file to prevent duplicate processing.

Python
import os
import boto3

# Initialize S3 client (automatically assumes EC2 Instance IAM Role)
s3 = boto3.client('s3')

RAW_BUCKET = 's3-sentinel-raw'
PROCESSED_BUCKET = 's3-sentinel-processed'

def process_files():
    print("Checking raw bucket for new files...")
    
    response = s3.list_objects_v2(Bucket=RAW_BUCKET)
    
    if 'Contents' not in response:
        print("No files found to process.")
        return

    for obj in response['Contents']:
        file_key = obj['Key']
        print(f"Processing file: {file_key}")
        
        # Download file from Raw bucket to local temp storage
        download_path = f"/tmp/{file_key}"
        s3.download_file(RAW_BUCKET, file_key, download_path)
        
        # Transform data (convert content to uppercase with a header)
        processed_path = f"/tmp/processed_{file_key}"
        with open(download_path, 'r', encoding='utf-8', errors='ignore') as f:
            content = f.read()
            
        processed_content = f"--- PROCESSED BY S3-SENTINEL ---\n\n" + content.upper()
        
        with open(processed_path, 'w', encoding='utf-8') as f:
            f.write(processed_content)
            
        # Upload transformed file to Processed bucket
        processed_key = f"processed-{file_key}"
        s3.upload_file(processed_path, PROCESSED_BUCKET, processed_key)
        print(f"Successfully uploaded processed file to: {processed_key}")
        
        # Clean up original file from raw bucket
        s3.delete_object(Bucket=RAW_BUCKET, Key=file_key)
        print(f"Deleted original file {file_key} from raw bucket.")

if __name__ == "__main__":
    process_files()
    
4. Production Automation (Cron)
To make the architecture fully autonomous, the Linux cron scheduler was configured to execute the script in the background every 5 minutes:

Installed and started the cron service on Ubuntu:

Bash
sudo apt update -y && sudo apt install cron -y
sudo systemctl enable cron && sudo systemctl start cron
Added the cron job rule (crontab -e):

Bash
*/5 * * * * /home/ubuntu/venv/bin/python /home/ubuntu/pipeline.py >> /home/ubuntu/pipeline.log 2>&1

🚀 How to Test the Pipeline
Upload a sample text file (e.g., sample.txt containing lowercase text) into your Raw S3 Bucket via the AWS Console.

Wait up to 5 minutes for the cron job to trigger automatically (or run manually via python pipeline.py for instant verification).

Check your Processed S3 Bucket to verify that processed-sample.txt appears with the applied data transformations!

Monitor execution logs anytime using:

Bash
cat /home/ubuntu/pipeline.log

💡 Key Takeaways & Skills Demonstrated
Cloud Architecture Design: Setting up decoupled storage zones and worker compute resources.

Cloud Security Implementation: Utilizing IAM instance profiles instead of hardcoded credentials.

Scripting & SDK Integration: Leveraging Python and boto3 to programmatically interact with cloud object storage.

System Automation: Managing background tasks and log rotation using Linux system tools (cron).
