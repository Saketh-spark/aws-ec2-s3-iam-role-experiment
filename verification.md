# Experiment Verification: Testing Matrix & Validation Checklist

This document details the complete validation strategy used to confirm that the IAM Role attached to the EC2 instance enforces least-privilege, read-only Amazon S3 access.

---

## 1. Verification Test Matrix

Every test in this matrix must be conducted in sequence to validate both positive authorizations (actions permitted) and negative boundaries (actions denied).

| Test ID | Test Category | Objective | Command Executed | Expected Status | Security Meaning |
| :---: | :--- | :--- | :--- | :---: | :--- |
| **TC-01** | **Identity** | Verify STS role assumption | `aws sts get-caller-identity` | **PASS** (Arn matches role) | Proves EC2 is using the IAM role, not user keys |
| **TC-02** | **Isolation** | Verify no static keys stored | `test -f ~/.aws/credentials && echo "Exists" \|\| echo "None"` | **PASS** (`None`) | Confirms zero local credential leakage risk |
| **TC-03** | **Boundary** | Test global bucket listing | `aws s3 ls` | **EXPECTED DENIAL** (`AccessDenied`) | Least privilege: cannot enumerate entire account |
| **TC-04** | **Read Access** | List objects in target bucket | `aws s3 ls s3://<BUCKET_NAME>` | **PASS** (`200 OK`, lists keys) | Validates `s3:ListBucket` permission |
| **TC-05** | **Read Access** | Download file from bucket | `aws s3 cp s3://<BUCKET_NAME>/<FILE> .` | **PASS** (Download complete) | Validates `s3:GetObject` permission |
| **TC-06** | **Integrity** | Verify downloaded file on disk | `ls -lh <FILE>` | **PASS** (File present & matching size) | Confirms payload received intact |
| **TC-07** | **Write Boundary**| Attempt to upload new object | `aws s3 cp <FILE> s3://<BUCKET_NAME>/upload.txt`| **EXPECTED DENIAL** (`AccessDenied`) | Enforces read-only constraint (`s3:PutObject` denied) |
| **TC-08** | **Delete Boundary**| Attempt to delete an object | `aws s3 rm s3://<BUCKET_NAME>/<FILE>` | **EXPECTED DENIAL** (`AccessDenied`) | Enforces non-destructive access (`s3:DeleteObject` denied) |

---

## 2. Step-by-Step Verification Protocol

### Verification Step 1: Confirm IAM Role Identity
Execute:
```bash
aws sts get-caller-identity
```
**Acceptance Criteria**:
1. `UserId` begins with `AROA...`
2. `Arn` ends with `assumed-role/EC2-S3-ReadOnly-Role/<Instance-ID>`
3. If this command outputs `Unable to locate credentials`, the IAM role has not been attached to the instance or the instance profile has not synced.

---

### Verification Step 2: Confirm Account-Wide S3 Isolation
Execute:
```bash
aws s3 ls
```
**Acceptance Criteria**:
```text
An error occurred (AccessDenied) when calling the ListBuckets operation: Access Denied
```
*Note*: This error is **mandatory** for lab compliance. If `aws s3 ls` returns a list of all buckets in your account, your policy is overly permissive (e.g., using `AmazonS3FullAccess` or broad wildcard policies).

---

### Verification Step 3: Confirm Scoped Bucket Reading
Execute:
```bash
aws s3 ls s3://iam-ec2-lab-20261115-4827
```
**Acceptance Criteria**:
The command succeeds and lists the lab spreadsheet:
```text
2026-11-15 14:22:10      21191 AIDS-ASSIGNMENT-SOA.xlsx
```

---

### Verification Step 4: Confirm Object Download
Execute:
```bash
aws s3 cp s3://iam-ec2-lab-20261115-4827/AIDS-ASSIGNMENT-SOA.xlsx .
ls -l AIDS-ASSIGNMENT-SOA.xlsx
```
**Acceptance Criteria**:
Output indicates `download:` followed by local filename, and `ls -l` shows `21191` bytes.

---

### Verification Step 5: Confirm Write Denial
Execute:
```bash
aws s3 cp AIDS-ASSIGNMENT-SOA.xlsx s3://iam-ec2-lab-20261115-4827/test-write.xlsx
```
**Acceptance Criteria**:
```text
upload failed: ./AIDS-ASSIGNMENT-SOA.xlsx to s3://iam-ec2-lab-20261115-4827/test-write.xlsx An error occurred (AccessDenied) when calling the PutObject operation: Access Denied
```

---

## 3. Comprehensive Verification Checklist

Mark each checkbox during or after lab evaluation:

- [x] **EC2 Instance State**: `IAM-S3-Lab-Instance` is in `Running` state.
- [x] **Status Checks**: All system and instance status checks report `Passed`.
- [x] **IAM Role Trust**: `EC2-S3-ReadOnly-Role` lists `ec2.amazonaws.com` as trusted service.
- [x] **Instance Profile**: Role is actively associated with `IAM-S3-Lab-Instance`.
- [x] **No Hardcoded Keys**: Directory `~/.aws/credentials` does not exist or contains no static keys.
- [x] **CLI Connectivity**: `aws --version` functions properly.
- [x] **STS Identity Confirmed**: `aws sts get-caller-identity` reflects assumed role ARN.
- [x] **Account Enumeration Blocked**: `aws s3 ls` returns `AccessDenied`.
- [x] **Target Bucket Enumeration Succeeded**: `aws s3 ls s3://...` lists bucket items.
- [x] **File Download Succeeded**: `aws s3 cp s3://... .` completes without error.
- [x] **File Integrity Verified**: Downloaded file size matches S3 metadata exactly.
- [x] **Write Operation Blocked**: `aws s3 cp` to S3 fails with `AccessDenied` on `PutObject`.

---

## 4. Troubleshooting & Diagnostic Guide

| Symptom | Probable Cause | Diagnostic Command | Remediation |
| :--- | :--- | :--- | :--- |
| `Unable to locate credentials` | Role not attached to EC2 or IMDS disabled | `curl -s http://169.254.169.254/latest/meta-data/iam/security-credentials/` | In EC2 Console: Actions &rarr; Security &rarr; Modify IAM role &rarr; Select `EC2-S3-ReadOnly-Role` |
| `AccessDenied` on `aws s3 ls s3://<bucket>` | Bucket ARN incorrect in policy | Review policy JSON in IAM Console | Ensure Resource ARN is `arn:aws:s3:::<exact-bucket-name>` (no trailing `/*`) |
| `AccessDenied` on download | Object ARN incorrect in policy | Review policy JSON in IAM Console | Ensure Resource ARN for `s3:GetObject` ends with `/*` |
| `Upload succeeded` (Security Failure!) | Policy contains `s3:PutObject` or `s3:*` | Review policy JSON in IAM Console | Remove `s3:PutObject` from the policy statement |
