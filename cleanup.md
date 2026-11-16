# Resource Cleanup & Teardown Guide

To adhere to cloud engineering hygiene and avoid incurring unexpected AWS billing costs, follow this teardown guide to safely decommission the experiment resources.

> [!WARNING]
> Only delete resources created specifically for this lab experiment. **Do NOT delete** existing production or shared S3 buckets, VPCs, key pairs, or unrelated EC2 instances.

---

## Teardown Sequence Overview

```
1. Detach IAM Role from EC2 Instance
       ↓
2. Terminate or Stop the EC2 Instance
       ↓
3. Delete IAM Role & Policies
       ↓
4. Delete Lab Test Bucket (Optional / Lab-Specific)
       ↓
5. Verify Resource Termination
```

---

## Step 1: Detach the IAM Role from the EC2 Instance

Before terminating or re-purposing the EC2 instance, cleanly disassociate the IAM Instance Profile.

1. Open the **Amazon EC2 Console** (`eu-north-1`).
2. In the left navigation, click **Instances** &rarr; **Instances**.
3. Select the checkbox for `IAM-S3-Lab-Instance`.
4. Click **Actions** &rarr; **Security** &rarr; **Modify IAM role**.
5. In the **IAM role** dropdown, select **No IAM role**.
6. Click **[Update IAM role]**.
7. Confirm the notification: *Successfully detached IAM role from instance...*

---

## Step 2: Terminate or Stop the EC2 Instance

If the EC2 instance was launched solely for this experiment, terminate it to stop compute charges.

1. In the EC2 **Instances** list, select `IAM-S3-Lab-Instance`.
2. Click **Instance state** in the top action bar:
   * **Stop instance**: Pauses compute execution while preserving EBS root volume data (minimal EBS storage cost applies).
   * **Terminate instance**: Permanently deprovisions the virtual machine and deletes the associated root EBS volume.
3. Choose **Terminate instance**.
4. When prompted to confirm, click **[Terminate]**.
5. Refresh the dashboard until **Instance state** transitions to `Terminated` (gray status indicator).

---

## Step 3: Delete the IAM Role and Policies

1. Open the **AWS IAM Console** (`https://console.aws.amazon.com/iam/`).
2. In the left navigation pane, click **Roles**.
3. In the search box, type `EC2-S3-ReadOnly-Role`.
4. Select the checkbox next to `EC2-S3-ReadOnly-Role`.
5. *(Optional)* Click into the role, navigate to the **Permissions** tab, select `EC2-S3-ReadOnly-RolePolicy`, and click **Delete**.
6. Return to the Roles list, select `EC2-S3-ReadOnly-Role`, and click the **[Delete]** button at the top-right.
7. In the confirmation dialogue, enter the role name `EC2-S3-ReadOnly-Role` to confirm deletion.
8. Click **[Delete]**.

---

## Step 4: Clean Up Test Objects and Bucket (If Applicable)

> [!IMPORTANT]
> If `iam-ec2-lab-20261115-4827` was provided by your institution or shared lab environment, **DO NOT** delete it. Only perform this step if you created your own dedicated bucket for this experiment.

If using a self-created test bucket:
1. Open the **Amazon S3 Console**.
2. Select your test bucket (e.g., `aws-cli-s3-saketh-2026-12345`).
3. Click **[Empty]** to purge all object versions, type `permanently delete`, and click **[Empty]**.
4. After the bucket is empty, select the bucket name and click **[Delete]**.
5. Enter the exact bucket name to confirm and click **[Delete bucket]**.

---

## Step 5: Verification of Zero Lingering Charges

Confirm that all billable compute resources are terminated:

```bash
# Verify instance is in terminated state
aws ec2 describe-instances \
    --filters "Name=tag:Name,Values=IAM-S3-Lab-Instance" \
    --query "Reservations[*].Instances[*].[InstanceId,State.Name]" \
    --output table

# Verify IAM role no longer exists
aws iam get-role --role-name EC2-S3-ReadOnly-Role
```

* The EC2 command should show status `terminated`.
* The IAM command should return `NoSuchEntity` error, confirming clean removal.
