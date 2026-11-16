# Screenshots Directory & Capture Guide

This directory houses the chronological visual evidence documenting the **EC2 + Amazon S3 Read-Only Access Using IAM Role** experiment.

---

## Screenshot Inventory & Checklist

Below is the complete inventory of the 12 experiment screenshots required for this repository:

| # | Filename | Description / Console View to Capture | Target Path | Status |
| :---: | :--- | :--- | :--- | :---: |
| 01 | `01-aws-console.png` | AWS Management Console home showing Region `Europe (Stockholm) eu-north-1` | `screenshots/01-aws-console.png` | Placeholder Ready |
| 02 | `02-create-iam-role.png` | IAM &rarr; Roles dashboard with the **[Create role]** button highlighted | `screenshots/02-create-iam-role.png` | Placeholder Ready |
| 03 | `03-trusted-entity.png` | Trusted entity wizard: **AWS service** selected & **EC2** use case checked | `screenshots/03-trusted-entity.png` | Placeholder Ready |
| 04 | `04-permissions-policy.png` | Permissions policy editor showing `EC2-S3-ReadOnly-RolePolicy` JSON block | `screenshots/04-permissions-policy.png` | Placeholder Ready |
| 05 | `05-iam-role-created.png` | IAM Role summary page showing `EC2-S3-ReadOnly-Role` with trust entity `ec2.amazonaws.com` | `screenshots/05-iam-role-created.png` | Placeholder Ready |
| 06 | `06-ec2-instance.png` | EC2 Instances dashboard showing `IAM-S3-Lab-Instance` in `Running` state | `screenshots/06-ec2-instance.png` | Placeholder Ready |
| 07 | `07-attach-iam-role.png` | EC2 **Modify IAM role** dialogue showing `EC2-S3-ReadOnly-Role` attached | `screenshots/07-attach-iam-role.png` | Placeholder Ready |
| 08 | `08-connect-to-ec2.png` | EC2 Instance Connect browser terminal open with Amazon Linux 2023 prompt | `screenshots/08-connect-to-ec2.png` | Placeholder Ready |
| 09 | `09-aws-cli-verification.png` | Terminal output of `aws --version` and `aws sts get-caller-identity` showing AssumedRole ARN | `screenshots/09-aws-cli-verification.png` | Placeholder Ready |
| 10 | `10-s3-bucket-access.png` | Terminal output of `aws s3 ls` (AccessDenied) and `aws s3 ls s3://iam-ec2-lab-20261115-4827` (Success) | `screenshots/10-s3-bucket-access.png` | Placeholder Ready |
| 11 | `11-read-only-permission-test.png` | Terminal output of `aws s3 cp ... .` (download successful) and `ls -l` verifying file size | `screenshots/11-read-only-permission-test.png` | Placeholder Ready |
| 12 | `12-final-output.png` | Terminal output of `aws s3 cp` upload attempt failing with `AccessDenied (PutObject)` | `screenshots/12-final-output.png` | Placeholder Ready |

---

## How to Replace Placeholders with Actual Captures

1. Capture your console or terminal window using Windows Snipping Tool (`Win + Shift + S`) or macOS Screenshot (`Cmd + Shift + 4`).
2. Save each screenshot directly into this `screenshots/` directory matching the exact filename above (e.g. `01-aws-console.png`).
3. Overwrite the corresponding placeholder image.
4. Git commit and push:
   ```bash
   git add screenshots/
   git commit -m "docs: add actual lab experiment screenshots"
   git push origin main
   ```
5. The `README.md` will immediately render your live captures with zero markdown modifications required!
