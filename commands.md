# AWS CLI Commands Reference: Syntax, Execution & Expected Output

This document contains all AWS CLI and Linux commands used during the experiment, accompanied by detailed syntax breakdowns, placeholders, real-world expected outputs, and technical explanations.

---

## Command Quick Reference

| # | Command | Purpose | Expected Outcome |
| :--- | :--- | :--- | :--- |
| 1 | `aws --version` | Verify AWS CLI installation | Outputs CLI, Python, OS versions |
| 2 | `aws sts get-caller-identity` | Verify active caller identity | Assumed IAM Role ARN |
| 3 | `aws configure list` | Check credential provider | Shows credentials sourced from `iam-role` |
| 4 | `aws s3 ls` | Global bucket listing attempt | `AccessDenied` (`s3:ListAllMyBuckets`) |
| 5 | `aws s3 ls s3://<BUCKET_NAME>` | List target bucket contents | Success: displays bucket objects |
| 6 | `aws s3 cp s3://<BUCKET_NAME>/<FILE> .` | Download object from bucket | Success: file downloaded locally |
| 7 | `ls -lh <FILE>` | Verify downloaded file size/permissions | Displays local file metadata |
| 8 | `aws s3 cp <FILE> s3://<BUCKET_NAME>/` | Write / Upload attempt | `AccessDenied` (`s3:PutObject`) |

---

## Placeholders Guide

When running these commands in your own AWS account, substitute the following placeholders:

| Placeholder | Lab Value Used | Replacement Instruction |
| :--- | :--- | :--- |
| `<BUCKET_NAME>` | `iam-ec2-lab-20261115-4827` | Enter your unique target S3 bucket name |
| `<FILE_NAME>` | `AIDS-ASSIGNMENT-SOA.xlsx` | Enter the name of the file stored in your S3 bucket |
| `<ACCOUNT_ID>` | `123456789012` | Your 12-digit AWS Account ID |
| `<INSTANCE_ID>` | `i-0a1b2c3d4e5f67890` | Your EC2 instance ID |

---

## Detailed Command Walkthrough

### 1. Verify AWS CLI Installation

```bash
aws --version
```

* **Purpose**: Verifies that the AWS CLI is installed, located in the system `PATH`, and running an up-to-date version.
* **Execution Environment**: EC2 Instance Terminal (Amazon Linux 2023).
* **Expected Output**:
  ```text
  aws-cli/2.38.0 Python/3.11.11 Linux/6.1.115-126.197.amzn2023.x86_64 exe/x86_64.amzn.2023
  ```
* **Explanation**: Amazon Linux 2023 includes AWS CLI v2 by default. The version string indicates the CLI version, Python runtime, and host OS kernel architecture.

---

### 2. Verify Caller Identity & Attached IAM Role

```bash
aws sts get-caller-identity
```

* **Purpose**: Queries the AWS Security Token Service (STS) to determine which IAM principal credentials the AWS CLI is currently using.
* **Expected Output**:
  ```json
  {
      "UserId": "AROAQAAAAAAAEXAMPLE:i-0a1b2c3d4e5f67890",
      "Account": "123456789012",
      "Arn": "arn:aws:sts::123456789012:assumed-role/EC2-S3-ReadOnly-Role/i-0a1b2c3d4e5f67890"
  }
  ```
* **Explanation**:
  * `UserId`: Starts with `AROA...`, which is the unique prefix for an assumed IAM Role session, followed by the EC2 instance ID as the role session name.
  * `Arn`: Confirms the role is `EC2-S3-ReadOnly-Role` assumed by the EC2 instance `i-0a1b2c3d4e5f67890`.
  * **Critical Note**: Notice that no `aws configure` with static access keys was run. The credentials were obtained automatically via the instance metadata service!

---

### 3. Inspect Credential Provider Chain

```bash
aws configure list
```

* **Purpose**: Inspects where the AWS CLI is resolving its credentials.
* **Expected Output**:
  ```text
        Name                    Value             Type    Location
        ----                    -----             ----    --------
     profile                <not set>             None    None
  access_key     ****************WXYZ         iam-role    
  secret_key     ****************abcd         iam-role    
      region                eu-north-1             imds    [EC2 Metadata]
  ```
* **Explanation**:
  * The `Type` column explicitly states `iam-role`.
  * This confirms that no static credentials exist in `~/.aws/credentials`.

---

### 4. Test Global S3 Listing (Least Privilege Check)

```bash
aws s3 ls
```

* **Purpose**: Attempts to enumerate all S3 buckets within the AWS Account.
* **Expected Output**:
  ```text
  An error occurred (AccessDenied) when calling the ListBuckets operation: Access Denied
  ```
* **Explanation**:
  * The attached inline policy `EC2-S3-ReadOnly-RolePolicy` intentionally omits the action `s3:ListAllMyBuckets`.
  * In AWS IAM, all actions default to an **implicit deny** unless explicitly permitted.
  * Receiving `AccessDenied` here is the **desired and correct** security behavior.

---

### 5. List Objects Within the Permitted S3 Bucket

```bash
aws s3 ls s3://iam-ec2-lab-20261115-4827
```

* **Purpose**: Lists objects and directory prefixes within the specifically authorized bucket.
* **Expected Output**:
  ```text
  2026-11-15 14:22:10      21191 AIDS-ASSIGNMENT-SOA.xlsx
  ```
* **Explanation**:
  * The policy explicitly grants `s3:ListBucket` on `arn:aws:s3:::iam-ec2-lab-20261115-4827`.
  * The request is authorized and returns object timestamp, file size (21,191 bytes), and object key.

---

### 6. Download (Read) Object from S3 to EC2

```bash
aws s3 cp s3://iam-ec2-lab-20261115-4827/AIDS-ASSIGNMENT-SOA.xlsx .
```

* **Purpose**: Copies the file from the remote S3 bucket to the local working directory (`.`) on the EC2 instance.
* **Expected Output**:
  ```text
  download: s3://iam-ec2-lab-20261115-4827/AIDS-ASSIGNMENT-SOA.xlsx to ./AIDS-ASSIGNMENT-SOA.xlsx
  ```
* **Explanation**:
  * The policy explicitly allows `s3:GetObject` on `arn:aws:s3:::iam-ec2-lab-20261115-4827/*`.
  * The file payload is transferred securely over TLS (HTTPS) to the EC2 storage volume.

---

### 7. Verify Local File Existence and Size

```bash
ls -lh AIDS-ASSIGNMENT-SOA.xlsx
```

* **Purpose**: Confirms that the file was written to disk and has an identical file size.
* **Expected Output**:
  ```text
  -rw-r--r--. 1 ec2-user ec2-user 21K Nov 15 14:22 AIDS-ASSIGNMENT-SOA.xlsx
  ```
* **Explanation**: Confirms successful end-to-end read access.

---

### 8. Test Write/Upload Operation (Negative Test)

```bash
aws s3 cp AIDS-ASSIGNMENT-SOA.xlsx s3://iam-ec2-lab-20261115-4827/upload-attempt.xlsx
```

* **Purpose**: Attempts to upload or overwrite an object in the S3 bucket to verify write denial.
* **Expected Output**:
  ```text
  upload failed: ./AIDS-ASSIGNMENT-SOA.xlsx to s3://iam-ec2-lab-20261115-4827/upload-attempt.xlsx An error occurred (AccessDenied) when calling the PutObject operation: Access Denied
  ```
* **Explanation**:
  * The IAM role does NOT have `s3:PutObject` permission.
  * AWS IAM denies the API call immediately, returning HTTP 403 Forbidden (`AccessDenied`).
  * This confirms that the role is strictly read-only and cannot tamper with bucket contents.

---

## Advanced: Inspecting Temporary Credentials via IMDSv2

To understand how EC2 retrieves credentials from the metadata service behind the scenes, run:

```bash
# 1. Obtain IMDSv2 Session Token (valid for 6 hours / 21600 seconds)
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")

# 2. Query attached IAM role name
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/iam/security-credentials/

# 3. View temporary credentials (do NOT share or commit this output!)
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/iam/security-credentials/EC2-S3-ReadOnly-Role
```

Sample IMDSv2 response:
```json
{
  "Code": "Success",
  "LastUpdated": "2026-11-15T14:20:00Z",
  "Type": "AWS-HMAC",
  "AccessKeyId": "ASIAEXAMPLETOKEN123",
  "SecretAccessKey": "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY",
  "Token": "IQoJb3JpZ2luX2VjEAUa...",
  "Expiration": "2026-11-15T20:55:00Z"
}
```

Notice the prefix `ASIA` indicating temporary, rotating credentials issued by STS.
