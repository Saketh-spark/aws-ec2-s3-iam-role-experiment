# Prerequisites: EC2 + Amazon S3 Read-Only Access Using IAM Role

Before executing this experiment, ensure all architectural, account, and networking prerequisites are met. Following these requirements guarantees that access tests behave predictably and security boundaries are preserved.

---

## 1. AWS Account & Administrative Permissions

To perform this experiment, you must have access to an AWS Account with permissions to manage IAM roles, policies, Amazon EC2 instances, and Amazon S3 buckets.

Required IAM permissions include:
* `iam:CreateRole`, `iam:PutRolePolicy`, `iam:AttachRolePolicy`, `iam:PassRole`, `iam:ListRoles`
* `ec2:DescribeInstances`, `ec2:AssociateIamInstanceProfile`, `ec2:ReplaceIamInstanceProfileAssociation`
* `ec2-instance-connect:SendSSHPublicKey`
* `s3:CreateBucket`, `s3:ListBucket`, `s3:GetObject`, `s3:PutObject` (for pre-lab bucket preparation)

> **Important**: Never perform experiments using the AWS root account. Use an IAM administrator identity or a dedicated lab sandbox account.

---

## 2. AWS Region Selection

Choose a specific AWS Region and maintain it consistently across all created resources (EC2 instance, IAM role, and S3 bucket).

* **Default Lab Region**: `eu-north-1` (Europe - Stockholm)
* **Alternative Standard Regions**: `us-east-1` (N. Virginia), `us-west-2` (Oregon), `ap-south-1` (Mumbai)

> **Note**: While IAM is a global service (`iam.amazonaws.com`), EC2 instances and S3 buckets are region-specific. Ensure your EC2 instance and S3 bucket reside in the same region to minimize data transfer latency and avoid cross-region data transfer fees.

---

## 3. Pre-Configured S3 Bucket & Test Data

The experiment verifies read-only access to a specific S3 bucket. Before starting, an Amazon S3 bucket must exist with at least one file uploaded.

| Parameter | Lab Value (Example) | Custom Value (Your Account) |
| :--- | :--- | :--- |
| **Bucket Name** | `iam-ec2-lab-20261115-4827` | `<your-unique-bucket-name>` |
| **Sample Object** | `AIDS-ASSIGNMENT-SOA.xlsx` | `test-read-file.txt` or `sample.xlsx` |
| **Bucket Region** | `eu-north-1` | Matches EC2 Instance Region |
| **Bucket Privacy** | Block all public access: **Enabled** | Block all public access: **Enabled** |

### Creating the Bucket via AWS CLI (Optional Pre-step)
```bash
# If using eu-north-1
aws s3api create-bucket \
    --bucket <YOUR-UNIQUE-BUCKET-NAME> \
    --region eu-north-1 \
    --create-bucket-configuration LocationConstraint=eu-north-1

# Upload a test file
aws s3 cp sample.xlsx s3://<YOUR-UNIQUE-BUCKET-NAME>/
```

---

## 4. Amazon EC2 Target Instance

An EC2 instance must be launched and healthy.

| Specification | Recommendation |
| :--- | :--- |
| **Name Tag** | `IAM-S3-Lab-Instance` |
| **Operating System / AMI** | Amazon Linux 2023 AMI (x86_64 or ARM64) |
| **Instance Type** | `t3.micro` or `t2.micro` (AWS Free Tier eligible) |
| **Subnet** | Public subnet with **Auto-assign public IPv4** enabled |
| **Instance State** | `Running` with `2/2` or `3/3` Status Checks passed |

---

## 5. Security Group & Network Requirements

The EC2 instance requires specific inbound and outbound traffic rules:

### Outbound Rules (Egress)
* **Protocol**: HTTPS (`TCP 443`)
* **Destination**: `0.0.0.0/0` (Allows the instance to communicate securely with AWS STS and Amazon S3 endpoints).

### Inbound Rules (Ingress)
* **Protocol**: SSH (`TCP 22`)
* **Source**:
  * Option A: Your local IP address (`<YOUR_IP>/32`) if connecting via OpenSSH.
  * Option B: The AWS EC2 Instance Connect CIDR prefix for your region (e.g., `13.48.4.200/30` for `eu-north-1`).

---

## 6. EC2 Instance Connect Prerequisites

This experiment uses **EC2 Instance Connect** directly from the AWS Console for terminal access. No local SSH client or private key (`.pem`) pair is required on your machine.
* Modern web browser (Chrome, Chromium, Firefox, Edge).
* Browser pop-up blocker disabled for `*.console.aws.amazon.com`.

---

## 7. AWS CLI v2 Availability

Amazon Linux 2023 AMIs come pre-installed with AWS CLI v2. If you are using an alternative Linux distribution (such as Ubuntu or Debian), install the AWS CLI using:

```bash
# Verify AWS CLI installation
aws --version

# If not installed on Ubuntu / Debian:
sudo apt-get update && sudo apt-get install -y unzip curl
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

---

## Summary Checklist

- [ ] Target AWS Region selected (`eu-north-1`).
- [ ] Active EC2 instance (`IAM-S3-Lab-Instance`) running Amazon Linux 2023.
- [ ] Accessible S3 bucket (`iam-ec2-lab-20261115-4827`) containing at least one test object.
- [ ] EC2 security group permits outbound HTTPS (port 443).
- [ ] EC2 Instance Connect opens without network blockades.
- [ ] No static AWS access keys (`~/.aws/credentials`) configured inside the EC2 instance.
