# EC2 + Amazon S3 Read-Only Access Using IAM Role

[![AWS](https://img.shields.io/badge/AWS-Cloud-orange?logo=amazon-aws&logoColor=white)](https://aws.amazon.com/)
[![IAM](https://img.shields.io/badge/IAM-Role--Based%20Security-red)](https://aws.amazon.com/iam/)
[![EC2](https://img.shields.io/badge/Amazon-EC2-orange)](https://aws.amazon.com/ec2/)
[![S3](https://img.shields.io/badge/Amazon-S3-green)](https://aws.amazon.com/s3/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

A complete, production-grade implementation guide and documentation repository demonstrating how an Amazon EC2 Linux instance securely accesses Amazon Simple Storage Service (S3) buckets using AWS Identity and Access Management (IAM) Roles—without storing static access keys on the instance.

---

## Table of Contents

- [Experiment Title](#experiment-title)
- [Introduction](#introduction)
- [Aim & Objectives](#aim--objectives)
- [AWS Services Used](#aws-services-used)
- [Architecture Overview](#architecture-overview)
- [Prerequisites](#prerequisites)
- [Step-by-Step Implementation](#step-by-step-implementation)
- [AWS CLI Commands Reference](#aws-cli-commands-reference)
- [Verification & Expected Outputs](#verification--expected-outputs)
- [Results](#results)
- [Conclusion](#conclusion)
- [Important Security Considerations](#important-security-considerations)
- [Cleanup](#cleanup)
- [Repository Structure](#repository-structure)
- [License](#license)

---

## Experiment Title

**EC2 + Amazon S3 Read-Only Access Using IAM Role**

---

## Introduction

In modern cloud computing, applications hosted on virtual compute instances (such as Amazon EC2) frequently need to access downstream services like object storage (Amazon S3). Historically, developers stored static, long-lived AWS IAM User credentials (`AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY`) in configuration files, environment variables, or local profile paths on the virtual machine.

This legacy practice introduces severe security vulnerabilities:
1. **Credential Exposure**: Hardcoded keys can be accidentally committed to public Git repositories or leaked via logs.
2. **Maintenance Overhead**: Rotating keys across distributed servers without application downtime is complex and error-prone.
3. **Privilege Creep**: Static keys often accumulate excessive privileges over time.

This experiment implements the AWS-recommended best practice: assigning an **IAM Role** to an EC2 instance via an **Instance Profile**. The instance retrieves short-lived, automatically rotated credentials via the **Instance Metadata Service (IMDSv2)**, completely eliminating static credentials from the host machine.

---

## Aim & Objectives

### Aim
To configure, attach, and validate an AWS IAM Role with a fine-grained read-only policy for an Amazon EC2 instance, enabling secure read and download access to a designated Amazon S3 bucket while enforcing strict write and account-level enumeration denials.

### Objectives
1. **Create an IAM Role** specifying `ec2.amazonaws.com` as the Trusted Entity.
2. **Craft a Least-Privilege IAM Policy** permitting `s3:ListBucket` and `s3:GetObject` strictly for a designated target bucket.
3. **Attach the IAM Role** to an active Amazon EC2 instance via an Instance Profile.
4. **Connect to the EC2 Terminal** using browser-based EC2 Instance Connect.
5. **Verify AWS CLI & STS Identity** to prove keyless role assumption (`aws sts get-caller-identity`).
6. **Validate Positive Read Access**: Successfully list bucket contents and download a target object.
7. **Validate Negative Security Boundaries**: Verify that account-wide bucket listing (`aws s3 ls`) and object uploads (`aws s3 cp` to S3) fail with `AccessDenied`.
8. **Document and Teardown**: Provide end-to-end evidence, architecture diagrams, and teardown procedures.

---

## AWS Services Used

| Service | Badge / Icon | Role in Experiment |
| :--- | :---: | :--- |
| **AWS IAM** | ![IAM](https://img.shields.io/badge/IAM-Identity-red) | Defines the `EC2-S3-ReadOnly-Role` identity, trust relationships, and permissions policies. |
| **Amazon EC2** | ![EC2](https://img.shields.io/badge/EC2-Compute-orange) | Hosts the `IAM-S3-Lab-Instance` virtual machine running Amazon Linux 2023. |
| **Amazon S3** | ![S3](https://img.shields.io/badge/S3-Storage-green) | Stores the target lab bucket `iam-ec2-lab-20261115-4827` and lab files. |
| **AWS STS** | ![STS](https://img.shields.io/badge/STS-Token_Service-blue) | Security Token Service; issues short-lived, rotating credentials for role assumption. |
| **EC2 Instance Connect** | ![EIC](https://img.shields.io/badge/EIC-Terminal-lightgrey) | Provides secure, identity-authenticated browser SSH access without local key pairs. |
| **AWS IMDSv2** | ![IMDS](https://img.shields.io/badge/IMDS-v2-purple) | Instance Metadata Service at `169.254.169.254` that delivers temporary STS credentials. |

---

## Architecture Overview

```text
+-------------------------------------------------------------------------------+
|                                  AWS Cloud                                    |
|                                                                               |
|  +--------------------+        Assumes         +---------------------------+  |
|  |    EC2 Instance    | ---------------------> |         IAM Role          |  |
|  | IAM-S3-Lab-Instance|                        |   EC2-S3-ReadOnly-Role    |  |
|  | (Amazon Linux 2023)| <--------------------- |                           |  |
|  +--------------------+   Temporary STS Token  +---------------------------+  |
|           |                                                  |                |
|           |                                                  | Attached       |
|           |                                                  v                |
|           |                                    +---------------------------+  |
|           |                                    |     Permission Policy     |  |
|           |                                    | EC2-S3-ReadOnly-RolePolicy|  |
|           |                                    +---------------------------+  |
|           |                                                  |                |
|           | Reads (s3:ListBucket, s3:GetObject)              | Grants Scoped  |
|           v                                                  v Read-Only      |
|  +-------------------------------------------------------------------------+  |
|  |                       Amazon S3 Storage Bucket                          |  |
|  |                     iam-ec2-lab-20261115-4827                           |  |
|  |               [ Object: AIDS-ASSIGNMENT-SOA.xlsx ]                      |  |
|  +-------------------------------------------------------------------------+  |
|                                                                               |
|  [ Write Attempt (s3:PutObject)             ==> 403 AccessDenied ]            |
|  [ Global List Attempt (s3:ListAllMyBuckets)==> 403 AccessDenied ]            |
+-------------------------------------------------------------------------------+
```

---

## Prerequisites

Before executing the experiment, ensure you have:
* An active **AWS Account** with permissions to manage IAM and EC2.
* Selected region: **`eu-north-1` (Europe - Stockholm)**.
* An active EC2 instance (`IAM-S3-Lab-Instance`) running Amazon Linux 2023.
* A target S3 bucket (`iam-ec2-lab-20261115-4827`) containing sample test data (`AIDS-ASSIGNMENT-SOA.xlsx`).
* Network connectivity permitting outbound HTTPS (port 443) and EC2 Instance Connect inbound access.

For full setup details, refer to [prerequisites.md](prerequisites.md).

---

## Step-by-Step Implementation

For an exhaustive, click-by-click breakdown with explanations for every decision, see [setup-guide.md](setup-guide.md). Below is the 19-step execution summary:

### Phase 1: IAM Role & Policy Provisioning
1. **Sign in to AWS Console**: Authenticate and set region to `Europe (Stockholm) / eu-north-1`.
2. **Open IAM**: Search and select **IAM** in the top navigation.
3. **Navigate to Roles**: Select **Roles** from the left-hand navigation pane.
4. **Create Role**: Click the orange **[Create role]** button.
5. **Select Trusted Entity**: Choose **AWS service** as the trusted entity type.
6. **Select EC2 Use Case**: Under Common use cases, choose **EC2** and click **Next**.
7. **Policy Configuration**: Proceed to the review page to attach a fine-grained inline policy.
8. **Role Naming**: Name the role `EC2-S3-ReadOnly-Role` and provide a descriptive summary.
9. **Finalize Role Creation**: Create the role, then attach the scoped inline policy `EC2-S3-ReadOnly-RolePolicy`:
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

### Phase 2: EC2 Association & Connection
10. **Open EC2 Dashboard**: Navigate to EC2 in `eu-north-1`.
11. **Select Instance**: Locate `IAM-S3-Lab-Instance` and verify its state is `Running`.
12. **Attach IAM Role**: Select instance &rarr; **Actions** &rarr; **Security** &rarr; **Modify IAM role** &rarr; Select `EC2-S3-ReadOnly-Role` &rarr; Click **[Update IAM role]**.
13. **Connect to Instance**: Click **[Connect]** &rarr; Choose **EC2 Instance Connect** &rarr; Launch browser terminal shell.

### Phase 3: Terminal Testing & Verification
14. **Verify AWS CLI**: Run `aws --version` to verify CLI availability.
15. **Verify STS Identity**: Run `aws sts get-caller-identity` to confirm the instance assumes `EC2-S3-ReadOnly-Role`.
16. **Global S3 Listing Test**: Run `aws s3 ls` &rarr; Returns `AccessDenied` (expected boundary test).
17. **Targeted Read Test**: Run `aws s3 ls s3://iam-ec2-lab-20261115-4827` and download the object via `aws s3 cp`.
18. **Write Operation Test**: Run `aws s3 cp AIDS-ASSIGNMENT-SOA.xlsx s3://...` &rarr; Returns `AccessDenied` (expected write restriction).
19. **Final Result Analysis**: Evaluate test results against cloud security standards.

---

## AWS CLI Commands Reference

All commands executed in the EC2 Instance Connect terminal:

```bash
# 1. Check AWS CLI Version
aws --version

# 2. Verify Caller Identity and Assumed Role
aws sts get-caller-identity

# 3. Test Global Bucket Enumeration (Expected: AccessDenied)
aws s3 ls

# 4. List Objects in Permitted Bucket (Expected: Success)
aws s3 ls s3://iam-ec2-lab-20261115-4827

# 5. Download Object from S3 to EC2 (Expected: Success)
aws s3 cp s3://iam-ec2-lab-20261115-4827/AIDS-ASSIGNMENT-SOA.xlsx .

# 6. Verify Local File Existence and Size
ls -lh AIDS-ASSIGNMENT-SOA.xlsx

# 7. Attempt Object Upload (Expected: AccessDenied)
aws s3 cp AIDS-ASSIGNMENT-SOA.xlsx s3://iam-ec2-lab-20261115-4827/
```

For full option descriptions and placeholder definitions, see [commands.md](commands.md).

---

## Verification & Expected Outputs

| Step / Test | Command Executed | Expected Output / Response | Status |
| :--- | :--- | :--- | :---: |
| **CLI Availability** | `aws --version` | `aws-cli/2.38.0 Python/3.11.11 Linux/...` | **PASS** |
| **Caller Identity** | `aws sts get-caller-identity` | `Arn: ...:assumed-role/EC2-S3-ReadOnly-Role/i-...` | **PASS** |
| **Global Bucket Listing** | `aws s3 ls` | `An error occurred (AccessDenied) ... ListBuckets` | **PASS** *(Denied by Design)* |
| **Permitted Bucket List** | `aws s3 ls s3://iam-ec2-lab...` | `2026-11-15 14:22:10 21191 AIDS-ASSIGNMENT-SOA.xlsx` | **PASS** |
| **Object Download** | `aws s3 cp s3://... .` | `download: s3://... to ./AIDS-ASSIGNMENT-SOA.xlsx` | **PASS** |
| **File Verification** | `ls -lh AIDS-ASSIGNMENT-SOA.xlsx`| `-rw-r--r--. 1 ec2-user ec2-user 21K ...` | **PASS** |
| **Write Restriction** | `aws s3 cp ... s3://...` | `upload failed: ... (AccessDenied) PutObject` | **PASS** *(Denied by Design)* |

For detailed test criteria and troubleshooting instructions, see [verification.md](verification.md).

---

## Results

1. **Authentication Without Static Credentials**: The EC2 instance successfully authenticated against AWS S3 endpoints without storing `AWS_ACCESS_KEY_ID` or `AWS_SECRET_ACCESS_KEY` in environment variables or configuration files.
2. **Authorized Read Operations**: Listing objects in `iam-ec2-lab-20261115-4827` and downloading the `AIDS-ASSIGNMENT-SOA.xlsx` object completed successfully with standard HTTP 200 responses.
3. **Enforced Access Boundaries**: 
   - Global bucket enumeration (`aws s3 ls`) was denied (`s3:ListAllMyBuckets` not authorized).
   - Write/upload operations (`aws s3 cp` to S3) were denied (`s3:PutObject` not authorized).

---

## Conclusion

This experiment proves that **AWS IAM Roles for Amazon EC2** provide an airtight, production-ready security mechanism for cloud workloads. By decoupling permissions from machine identities and leveraging automated STS token rotation via IMDSv2, organizations eliminate credential exfiltration vulnerabilities while adhering rigorously to the **Principle of Least Privilege**.

---

## Important Security Considerations

### 1. IAM Roles vs. IAM Users
* **IAM Users**: Intended for human operators requiring console or long-term programmatic access. Storing IAM user access keys on compute instances is considered an anti-pattern.
* **IAM Roles**: Intended for applications, compute resources (EC2, ECS, Lambda), or federated entities. They rely exclusively on short-lived, rotatable security tokens issued by AWS STS.

### 2. Trusted Entity & Trust Policy
* A **Trust Policy** defines *who can assume the role*. For EC2 instances, the trust policy explicitly declares the service principal `ec2.amazonaws.com`:
  ```json
  "Principal": { "Service": "ec2.amazonaws.com" }
  ```
* Without this trust policy, the EC2 hypervisor cannot request credentials on behalf of the virtual machine.

### 3. Permission Policies vs. Trust Policies
* **Trust Policy**: Evaluated when the instance requests credentials (`sts:AssumeRole`).
* **Permission Policy**: Evaluated when the application attempts actions on target resources (e.g., `s3:GetObject`).

### 4. Principle of Least Privilege (PoLP)
* The role grants access **only** to the specific bucket `arn:aws:s3:::iam-ec2-lab-20261115-4827` and **only** for read actions (`s3:ListBucket`, `s3:GetObject`).
* Broad wildcards (`s3:*`) and administrative policies (`AmazonS3FullAccess`) must never be used in production workloads when read-only access is sufficient.

### 5. Instance Metadata Service Version 2 (IMDSv2)
* Always enforce **IMDSv2** (`HttpTokens=required`) on EC2 instances. IMDSv2 utilizes session-oriented requests with PUT tokens, effectively blocking Server-Side Request Forgery (SSRF) vulnerabilities that could otherwise leak role credentials.

### 6. Risks of Hardcoded Credentials
| Threat Vector | Hardcoded IAM User Key | IAM Role via Instance Profile |
| :--- | :--- | :--- |
| **Source Code Leak** | High risk (keys exposed permanently) | Zero risk (no keys stored in code) |
| **Credential Rotation**| Manual, error-prone, causes downtime | Automatic (hourly rotation via STS) |
| **Audit & Attribution**| Hard to isolate across multiple servers | CloudTrail logs individual instance ID |
| **Server Compromise** | Attacker retains keys indefinitely | Attacker access expires with session |

---

## Cleanup

To avoid unwanted cloud charges, refer to [cleanup.md](cleanup.md) for step-by-step instructions on:
1. Detaching `EC2-S3-ReadOnly-Role` from `IAM-S3-Lab-Instance`.
2. Terminating the EC2 instance.
3. Deleting the IAM Role and inline policies.

---

## Repository Structure

```text
aws-ec2-s3-iam-role-experiment/
│
├── README.md                   # Primary experiment documentation & guide
├── experiment-details.md       # Technical specifications, JSON policies & IMDS deep-dive
├── prerequisites.md            # Account, regional, networking & AMI prerequisites
├── setup-guide.md              # Exhaustive 19-step console and configuration guide
├── commands.md                 # Complete AWS CLI command reference & expected outputs
├── verification.md             # Test matrix, validation protocol & troubleshooting
├── cleanup.md                  # Safe decommissioning & teardown procedures
├── .gitignore                  # Git ignore rules for cloud credentials & keys
└── LICENSE                     # MIT License
```

---

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.
