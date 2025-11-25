# ebs-unattached-automation
This Event Drive Architecture bypasses using AWS Config to detect unattached EBS Volumes that have not being attached to to an EC2 instance for 30 days and then it takes a snapshots of that unattached Volume and
save that for 90 days retention windows. that way Volume can be stored at the lowest costing snapshot pricing and if not snapshot is not restored after 90 days, the snapshote deleted.
Automating this using a Serverless environment give you the flexibility of retention of data at significant cost reduction. 

CloudFormation template + Serverless that snapshots and deletes unattached EBS volumes on a schedule, with tagged snapshots and automatic old-snapshot cleanup.

# EBS Unattached Volume Automation

This CloudFormation template deploys a single Lambda function and EventBridge rule that:

1. Finds **unattached (available) EBS volumes** in one or more AWS regions  
2. **Creates tagged snapshots** for each volume  
3. **Deletes the original volumes**  
4. **Deletes old snapshots** created by this automation once they fall outside your retention window  

It’s designed to be a simple, opinionated way to keep your account clean of forgotten, unattached EBS volumes while preserving snapshots for a configurable period.

---

## What this stack does

On each scheduled run, the Lambda:

1. **Discovers unattached EBS volumes**
   - Filters volumes with `status = available`
   - Optionally restricts to volumes with a specific tag key and/or value

2. **Creates snapshots**
   - Calls `CreateSnapshot` for each discovered volume
   - Tags each snapshot with `SnapshotTagKey` / `SnapshotTagValue`  
     - Default: `zotec:component = UnattachedVolume90daydelete`

3. **Deletes the original volumes**
   - After a snapshot is created, the original volume is deleted

4. **Cleans up old snapshots**
   - Lists your **own** snapshots (`OwnerIds=["self"]`) that have the same tag key/value
   - Deletes any snapshot older than `SnapshotRetentionDays`

All of this can be run in **“dry run” mode** so you can see exactly what it *would* do, without actually creating/deleting anything.

---

## CloudFormation parameters

The template exposes these key parameters:

- `SnapshotRetentionDays` (Number)  
  - **Default:** `90`  
  - Number of days to keep snapshots created by this automation before they are deleted.

- `ScheduleExpression` (String)  
  - **Default:** `cron(0 5 1 * ? *)` (05:00 UTC on the 1st of every month)  
  - Any valid EventBridge `cron()` or `rate()` expression.

- `ExecutionRegions` (CommaDelimitedList)  
  - **Default:** `us-east-2`  
  - Comma-separated AWS regions to operate in (e.g. `us-east-1,us-east-2`).  
  - If left empty at runtime, the Lambda falls back to its **current region** only.

- `VolumeTagFilterKey` (String)  
  - **Default:** `""` (empty)  
  - Optional. Only act on unattached volumes that have this tag key.  
  - Leave blank to act on **all** unattached volumes.

- `VolumeTagFilterValue` (String)  
  - **Default:** `""` (empty)  
  - Optional. If `VolumeTagFilterKey` is set, match this tag value.  
  - Leave blank to match any value for that key.

- `SnapshotTagKey` (String)  
  - **Default:** `zotec:component`  
  - Tag key applied to snapshots created by this Lambda.

- `SnapshotTagValue` (String)  
  - **Default:** `UnattachedVolume90daydelete`  
  - Tag value applied to snapshots created by this Lambda.  
  - Also used to find which snapshots are eligible for cleanup.

- `LogRetentionDays` (Number)  
  - **Default:** `30`  
  - CloudWatch Logs retention for the Lambda log group.

- `DryRun` (String: `"true"` / `"false"`)  
  - **Default:** `"false"`  
  - If `"true"`, **no snapshots or volumes are actually modified**.  
    The Lambda only logs what it *would* do.

---

## How it works (under the hood)

The Lambda code (Python 3.12) roughly does the following:

1. Reads configuration from environment variables set by the stack:
   - `RETENTION_DAYS`, `SNAPSHOT_TAG_KEY`, `SNAPSHOT_TAG_VALUE`
   - `EXECUTION_REGIONS`, `VOLUME_TAG_FILTER_KEY`, `VOLUME_TAG_FILTER_VALUE`
   - `DRY_RUN`

2. For each region:
   - Creates an `ec2` client with `boto3.client("ec2", region_name=region)`
   - Paginates through `DescribeVolumes` with filters:
     - `status = available` (unattached)
     - Optional tag filters based on `VOLUME_TAG_FILTER_KEY` / `VALUE`
   - For each volume:
     - Creates a snapshot (unless `DRY_RUN` is true)
     - Tags the snapshot with `SNAPSHOT_TAG_KEY = SNAPSHOT_TAG_VALUE`
     - Deletes the original volume (unless `DRY_RUN` is true)

3. Snapshot cleanup:
   - Computes a cutoff timestamp `now - RETENTION_DAYS`
   - Paginates `DescribeSnapshots` with:
     - `OwnerIds = ["self"]`
     - Filter on `tag:SNAPSHOT_TAG_KEY = SNAPSHOT_TAG_VALUE`
   - Deletes any snapshot older than the cutoff (unless `DRY_RUN` is true)

4. Logs all actions (and intended actions in dry run) to CloudWatch Logs.

---

## IAM permissions

The stack creates an IAM role for the Lambda with:
- `AWSLambdaBasicExecutionRole` managed policy (logging)
- Inline policy granting:
  - `ec2:DescribeVolumes`, `ec2:DescribeSnapshots`
  - `ec2:CreateSnapshot`, `ec2:DeleteSnapshot`, `ec2:CreateTags`, `ec2:DeleteVolume`
  - Minimal KMS permissions for encrypted volumes:
    - `kms:CreateGrant`, `kms:DescribeKey`, `kms:Encrypt`, `kms:Decrypt`, `kms:ReEncrypt*`,
      `kms:GenerateDataKeyWithoutPlainText` with condition `kms:GrantIsForAWSResource = true`

You shouldn’t need to add extra permissions for basic usage, but always review policies for your own security and compliance requirements.

---

## Safety notes & recommended usage

This stack **permanently deletes EBS volumes** and eventually **deletes old snapshots**. A few things to keep in mind:

1. **Strongly recommended: start with `DryRun = "true"`**
   - Deploy the stack with `DryRun` enabled.
   - Trigger the Lambda (either by waiting for the schedule or via a manual test invoke).
   - Review CloudWatch Logs to confirm which volumes and snapshots it *would* touch.
   - Once you’re confident in the behavior, update the stack to set `DryRun = "false"`.

2. **Use tag filters to limit scope**
   - Set `VolumeTagFilterKey` (and optionally `VolumeTagFilterValue`) so the automation only touches volumes *you’ve explicitly marked* for cleanup.
   - For example:
     - `VolumeTagFilterKey = CleanupPolicy`
     - `VolumeTagFilterValue = Unattached`

3. **Retention is only for snapshots created by this Lambda**
   - The cleanup logic only deletes snapshots with:
     - `SNAPSHOT_TAG_KEY = SNAPSHOT_TAG_VALUE`
   - Existing snapshots with different tags or no tags are **not touched**.

4. **Not multi-account**
   - This stack operates only **within the AWS account** where it is deployed.
   - It can operate across multiple regions in that account via `ExecutionRegions`.

5. **Schedule with care**
   - Default runs **monthly**.
   - You can reduce or increase frequency (e.g., `rate(7 days)`),
     but consider retention and how often unattached volumes are expected.

---

## Deploying the stack

High-level deployment steps:

1. Download `ebs-unattached-automation.yaml`.
2. In the AWS console, go to **CloudFormation → Create stack → With new resources (standard)**.
3. Upload the template file.
4. Choose stack name, adjust parameters:
   - Set `ExecutionRegions`, `SnapshotRetentionDays`, tag filters, and `DryRun`.
5. Review and create the stack.
6. After deployment, verify:
   - Lambda function name from the **Outputs** (`LambdaFunctionName`)
   - EventBridge rule ARN from **Outputs** (`EventRuleArn`)
   - CloudWatch Logs group: `/aws/lambda/<lambda-name>`

---

## When you should (and shouldn’t) use this

**A good fit when:**
- You routinely end up with unattached EBS volumes after instance termination / experiments.
- You want a simple, opinionated “set it and forget it” cleanup mechanism.
- You are comfortable with snapshot-based backup and a time-based retention policy.

**Not a good fit when:**
- You need per-volume custom retention logic.
- You want interactive approvals per deletion.
- You require integration with external CMDB / ticketing / approval workflows (this template is intentionally minimal).

---

## Outputs

The template exports two outputs:

- `LambdaFunctionName` – name of the automation Lambda function
- `EventRuleArn` – ARN of the EventBridge rule that triggers it

You can use these for monitoring, additional automation, or cross-stack references.
