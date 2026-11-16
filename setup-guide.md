# Complete Setup Guide: Step-by-Step Implementation

This guide provides an exhaustive walkthrough for configuring, attaching, and validating an AWS IAM Role for an Amazon EC2 instance to achieve secure, keyless read-only access to Amazon S3.

Follow each step in sequential order.

---

## Step 1: Sign in to AWS Management Console and Select Region

* **Navigation**: Open your browser and navigate to `https://console.aws.amazon.com/`.
* **Action**:
  1. Sign in using your IAM User credentials or AWS Single Sign-On (SSO) credentials.
  2. Locate the navigation bar at the top-right corner.
  3. Click the **Region Selector** dropdown and select:
     * **Region Name**: `Europe (Stockholm)`
     * **Region Identifier**: `eu-north-1`
* **Why this is necessary**: AWS resources such as EC2 instances and S3 buckets reside within specific geographic regions. Consistent regional alignment avoids cross-region latency, data transfer charges, and permission discovery mismatches.

---

## Step 2: Open the IAM Service

* **Navigation**: Click on the unified search bar at the very top of the console.
* **Action**: Type `IAM` into the search bar and click on the **IAM** (Identity and Access Management) service from the search results.
* **Why this is necessary**: IAM is the central identity control plane in AWS used to manage users, groups, roles, and fine-grained access policies.

---

## Step 3: Navigate to Roles

* **Navigation**: In the IAM left-hand navigation pane, find the **Access management** category.
* **Action**: Click on **Roles**.
* **Why this is necessary**: The Roles dashboard displays all identity roles currently provisioned in your AWS account and provides the entry point for creating new delegation identities.

---

## Step 4: Initiate IAM Role Creation

* **Action**: On the Roles dashboard page, locate and click the orange **[Create role]** button in the upper right.
* **Why this is necessary**: Starts the AWS role creation wizard which configures trust relationships, permissions, tags, and role naming.

---

## Step 5: Select Trusted Entity Type

* **Navigation**: On the **Select trusted entity** screen.
* **Action**: Under the section **Trusted entity type**, select the radio button for **AWS service**.
* **Why this is necessary**: An IAM role requires an explicit trust boundary. Selecting "AWS service" configures a trust policy that allows native AWS infrastructure services (like EC2, Lambda, or ECS) to assume the role via AWS STS.

---

## Step 6: Select EC2 as the Use Case

* **Navigation**: In the **Use case** section on the same screen.
* **Action**:
  1. Under **Common use cases**, click the radio button for **EC2**.
  2. Verify that the descriptive text reads: *Allows EC2 instances to call AWS services on your behalf.*
  3. Click the blue **[Next]** button at the bottom right.
* **Why this is necessary**: This automatically builds the trust relationship JSON document with `Principal: { "Service": "ec2.amazonaws.com" }` and action `sts:AssumeRole`.

---

## Step 7: Define Permissions Policy

You can attach the AWS-managed policy `AmazonS3ReadOnlyAccess` or craft a fine-grained inline policy scoped to your specific bucket. Scoping to the specific bucket represents cloud security best practice:

* **Action**:
  1. On the **Add permissions** page, click the **[Next]** button without selecting broad managed policies (we will add a precise inline policy to enforce true least privilege).
  2. *(Alternative option)*: If your curriculum permits the managed policy, search for `AmazonS3ReadOnlyAccess` in the filter box, select its checkbox, and click **[Next]**.
* **Why this is necessary**: Applying least privilege ensures the EC2 instance can only access the explicit bucket and objects intended for its operational scope.

---

## Step 8: Name and Review the IAM Role

* **Navigation**: On the **Name, review, and create** screen.
* **Action**:
  1. In the **Role name** input box, enter:
     ```text
     EC2-S3-ReadOnly-Role
     ```
  2. In the **Description** input box, enter:
     ```text
     Grants EC2 instances read-only access to specific S3 bucket without static credentials.
     ```
  3. Review the **Step 1: Select trusted entities** block:
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
* **Why this is necessary**: Role names must be distinct, descriptive, and adhere to AWS naming conventions.

---

## Step 9: Create the IAM Role & Attach Scoped Inline Policy

* **Action**:
  1. Click **[Create role]** at the bottom of the page.
  2. In the IAM Roles list, search for `EC2-S3-ReadOnly-Role` and click its name to open the role summary.
  3. Under the **Permissions** tab, click **Add permissions** &rarr; **Create inline policy**.
  4. Switch to the **JSON** tab in the policy editor.
  5. Delete any placeholder text and paste the following JSON:
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
  6. Click **[Next]**.
  7. In the **Policy name** field, enter:
     ```text
     EC2-S3-ReadOnly-RolePolicy
     ```
  8. Click **[Create policy]**.
* **Why this is necessary**: This binds `s3:ListBucket` (metadata listing) and `s3:GetObject` (file download) specifically to `iam-ec2-lab-20261115-4827` while keeping write (`s3:PutObject`) and global listing (`s3:ListAllMyBuckets`) strictly denied.

---

## Step 10: Open the EC2 Dashboard

* **Navigation**: Click the top search bar, type `EC2`, and select **EC2** from the search results.
* **Action**: Ensure the region remains `Europe (Stockholm) / eu-north-1`.
* **Why this is necessary**: EC2 management must take place in the region where your instance is provisioned.

---

## Step 11: Select the Target EC2 Instance

* **Navigation**: In the left-hand EC2 menu, click **Instances** &rarr; **Instances**.
* **Action**:
  1. Find your target instance named `IAM-S3-Lab-Instance`.
  2. Verify that **Instance state** is `Running`.
  3. Verify that **Status check** reports `2/2 checks passed` or `3/3 checks passed`.
* **Why this is necessary**: You cannot attach an instance profile reliably or connect via EC2 Instance Connect if the virtual machine is stopped or initializing.

---

## Step 12: Attach the IAM Role to the EC2 Instance

* **Action**:
  1. Click the checkbox next to `IAM-S3-Lab-Instance`.
  2. In the top-right action bar, click **Actions** &rarr; **Security** &rarr; **Modify IAM role**.
  3. In the **IAM role** dropdown menu, select `EC2-S3-ReadOnly-Role`.
  4. Click the blue **[Update IAM role]** button.
  5. Confirm the green banner: `Successfully attached EC2-S3-ReadOnly-Role to instance i-...`.
* **Why this is necessary**: This associates the IAM Instance Profile with the virtual machine hypervisor, making temporary STS security credentials available at the link-local metadata address `169.254.169.254`.

---

## Step 13: Connect to the EC2 Instance via EC2 Instance Connect

* **Action**:
  1. With `IAM-S3-Lab-Instance` selected, click the **[Connect]** button at the top of the dashboard.
  2. Select the **EC2 Instance Connect** tab.
  3. Leave the username as default (`ec2-user`).
  4. Ensure connection type is **Connect using EC2 Instance Connect**.
  5. Click the orange **[Connect]** button at the bottom-right corner.
  6. A new browser tab opens with the black terminal session showing:
     ```text
        ,     #_
        ~\_  ####_        Amazon Linux 2023
       ~~  \ _-###        AL2023 AMI
       ~~   \
     [ec2-user@ip-172-31-xx-xx ~]$
     ```
* **Why this is necessary**: EC2 Instance Connect provides a secure, identity-authenticated terminal connection directly in the browser without managing SSH private keys.

---

## Step 14: Verify AWS CLI Availability

* **Command**:
  ```bash
  aws --version
  ```
* **Expected Output**:
  ```text
  aws-cli/2.38.0 Python/3.11.11 Linux/6.1.115-126.197.amzn2023.x86_64 exe/x86_64.amzn.2023
  ```
* **Why this is necessary**: Confirms that AWS CLI v2 is present and capable of making API requests.

---

## Step 15: Verify the Attached IAM Role via AWS CLI

* **Command**:
  ```bash
  aws sts get-caller-identity
  ```
* **Expected Output**:
  ```json
  {
      "UserId": "AROAEXAMPLEEXAMPLE:i-0a1b2c3d4e5f67890",
      "Account": "123456789012",
      "Arn": "arn:aws:sts::123456789012:assumed-role/EC2-S3-ReadOnly-Role/i-0a1b2c3d4e5f67890"
  }
  ```
* **Why this is necessary**: Proves that the EC2 instance is actively assuming `EC2-S3-ReadOnly-Role` via AWS STS without any static credentials in `~/.aws/credentials`.

---

## Step 16: List All S3 Buckets (Global Enumeration Test)

* **Command**:
  ```bash
  aws s3 ls
  ```
* **Expected Output**:
  ```text
  An error occurred (AccessDenied) when calling the ListBuckets operation: Access Denied
  ```
* **Why this is necessary**: Demonstrates the Principle of Least Privilege. Because the policy only allows listing within `iam-ec2-lab-20261115-4827`, account-wide bucket enumeration is denied.

---

## Step 17: Test Read-Only S3 Access (Targeted Bucket & Download)

* **Command 1 (List permitted bucket)**:
  ```bash
  aws s3 ls s3://iam-ec2-lab-20261115-4827
  ```
* **Expected Output**:
  ```text
  2026-11-15 14:22:10      21191 AIDS-ASSIGNMENT-SOA.xlsx
  ```

* **Command 2 (Download object)**:
  ```bash
  aws s3 cp s3://iam-ec2-lab-20261115-4827/AIDS-ASSIGNMENT-SOA.xlsx .
  ```
* **Expected Output**:
  ```text
  download: s3://iam-ec2-lab-20261115-4827/AIDS-ASSIGNMENT-SOA.xlsx to ./AIDS-ASSIGNMENT-SOA.xlsx
  ```

* **Command 3 (Verify local copy)**:
  ```bash
  ls -lh AIDS-ASSIGNMENT-SOA.xlsx
  ```
* **Expected Output**:
  ```text
  -rw-r--r--. 1 ec2-user ec2-user 21K Nov 15 14:22 AIDS-ASSIGNMENT-SOA.xlsx
  ```
* **Why this is necessary**: Validates that `s3:ListBucket` and `s3:GetObject` are functioning properly through the IAM role.

---

## Step 18: Attempt Write Operation (Verify Access Denied)

* **Command**:
  ```bash
  aws s3 cp AIDS-ASSIGNMENT-SOA.xlsx s3://iam-ec2-lab-20261115-4827/upload-test.xlsx
  ```
* **Expected Output**:
  ```text
  upload failed: ./AIDS-ASSIGNMENT-SOA.xlsx to s3://iam-ec2-lab-20261115-4827/upload-test.xlsx An error occurred (AccessDenied) when calling the PutObject operation: Access Denied
  ```
* **Why this is necessary**: Proves that the IAM role is strictly read-only. AWS IAM defaults to an implicit deny for any action not explicitly granted in the policy.

---

## Step 19: Final Evaluation & Results Analysis

* **Outcome**:
  1. Identity assumed: `EC2-S3-ReadOnly-Role`.
  2. Credential delivery: Keyless via EC2 Instance Profile / IMDSv2.
  3. Read access: Allowed (`200 OK`).
  4. Write access: Denied (`403 AccessDenied`).
  5. Global enumeration: Denied (`403 AccessDenied`).
* **Conclusion**: The configuration achieves complete, defense-in-depth security compliance for cloud compute workloads interacting with Amazon S3.
