# Experiment Details: Technical Architecture & Specification

## 1. Executive Summary

This experiment implements and demonstrates a least-privilege, keyless authentication architecture between an **Amazon Elastic Compute Cloud (EC2)** virtual server and an **Amazon Simple Storage Service (S3)** object storage bucket. 

By leveraging **AWS Identity and Access Management (IAM) Roles** and **EC2 Instance Profiles**, the virtual machine dynamically retrieves short-lived temporary security credentials via the **Instance Metadata Service (IMDS)**. This eliminates the critical security risk of hardcoding long-term AWS access keys (`AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY`) on virtual servers.

---

## 2. Resource Specifications

The table below outlines the exact infrastructure parameters configured during this experiment:

| Parameter | Configuration Value | Description |
| :--- | :--- | :--- |
| **AWS Region** | `eu-north-1` (Europe - Stockholm) | Geographic region hosting the EC2 instance and S3 bucket |
| **EC2 Instance Name** | `IAM-S3-Lab-Instance` | Target Linux host running Amazon Linux 2023 |
| **Instance Type** | `t3.micro` | 2 vCPUs, 1.0 GiB Memory (Free Tier eligible) |
| **IAM Role Name** | `EC2-S3-ReadOnly-Role` | Dedicated IAM entity assumed by EC2 |
| **IAM Policy Name** | `EC2-S3-ReadOnly-RolePolicy` | Inline IAM permission policy attached to the role |
| **Trusted Entity** | `ec2.amazonaws.com` | AWS EC2 service principal authorized to assume the role |
| **Target S3 Bucket** | `iam-ec2-lab-20261115-4827` | Private S3 bucket containing lab data |
| **Target S3 Object** | `AIDS-ASSIGNMENT-SOA.xlsx` | 21 KB Excel spreadsheet used for read validation |
| **AWS CLI Version** | AWS CLI v2 (`aws-cli/2.38.0`+) | Pre-installed CLI tool on Amazon Linux 2023 |

---

## 3. IAM Policies Definition

### 3.1 Trust Relationship Policy (AssumeRole Policy)
The Trust Policy defines *who* or *what* is authorized to assume the IAM Role. Here, the service principal `ec2.amazonaws.com` is granted permission to invoke `sts:AssumeRole`.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

### 3.2 Permissions Policy (`EC2-S3-ReadOnly-RolePolicy`)
The Permissions Policy dictates the exact AWS API actions the role is authorized to perform once assumed. To enforce the **Principle of Least Privilege**, this policy restricts access strictly to reading and listing the single assigned bucket.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowListSpecificBucket",
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::iam-ec2-lab-20261115-4827"
    },
    {
      "Sid": "AllowReadObjects",
      "Effect": "Allow",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::iam-ec2-lab-20261115-4827/*"
    }
  ]
}
```

---

## 4. Deep-Dive: Permission Mechanics & Resource ARN Structure

Understanding the distinction between bucket-level actions and object-level actions is critical when writing AWS IAM policies:

| IAM Action | Scope Level | Target Resource ARN Format | Purpose in Experiment |
| :--- | :--- | :--- | :--- |
| `s3:ListBucket` | **Bucket Level** | `arn:aws:s3:::<bucket-name>` | Allows listing object metadata (keys, size, timestamps) inside the bucket. Does not require `/*`. |
| `s3:GetObject` | **Object Level** | `arn:aws:s3:::<bucket-name>/*` | Allows downloading object payloads. Requires wildcard `/*` to address any object key inside the bucket. |
| `s3:PutObject` | **Object Level** | *(Omitted - Implicit Deny)* | Write/Upload action. Intentionally excluded to verify that write attempts return `AccessDenied`. |
| `s3:ListAllMyBuckets`| **Global Level**| *(Omitted - Implicit Deny)* | Enumerating all buckets in the account. Intentionally excluded to verify global isolation. |

---

## 5. How Keyless Authentication Works Under the Hood

### Step 1: Instance Profile & Association
When an IAM Role is attached to an EC2 instance via the AWS Console or API (`AssociateIamInstanceProfile`), AWS automatically creates and links an **IAM Instance Profile** container to the virtual machine.

### Step 2: Instance Metadata Service (IMDSv2)
The EC2 operating system queries the link-local IP address `169.254.169.254`. Under IMDSv2, the process:
1. Obtains a session token via `PUT /latest/api/token`.
2. Requests role credentials via `GET /latest/meta-data/iam/security-credentials/<Role-Name>`.

### Step 3: STS Credential Delivery
The AWS Security Token Service (STS) returns a JSON payload containing:
* `AccessKeyId` (starts with `ASIA...`, designating temporary credentials)
* `SecretAccessKey`
* `Token` (session token)
* `Expiration` (UTC timestamp)

### Step 4: Automatic SDK/CLI Integration
The AWS CLI running inside EC2 automatically resolves its credential provider chain:
1. Environment variables (`AWS_ACCESS_KEY_ID`) &rarr; None.
2. Credentials file (`~/.aws/credentials`) &rarr; None.
3. Config file (`~/.aws/config`) &rarr; None.
4. **Instance Metadata Service (IAM Role credentials)** &rarr; **Found and active!**

Credentials are automatically renewed prior to expiration without requiring server restarts or configuration updates.
