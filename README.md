# AWS Secrets Access Monitoring System

This project monitors AWS Secrets Manager access activity and sends email alerts whenever a secret is accessed.  
The full chain is **Secrets Manager → CloudTrail → CloudWatch Logs → Metric Filter → Alarm → SNS Email Notification**.

---

## Overview

- Create and store a secret in AWS Secrets Manager.
- Enable CloudTrail to log all secret access events.
- Stream CloudTrail logs to CloudWatch Logs and build a metric filter.
- Create a CloudWatch Alarm and SNS topic for email alerts.
- (Secret Mission) Configure direct CloudTrail → SNS notifications and compare approaches.

![Image](http://learn.nextwork.org/overjoyed_silver_glamorous_cape_gooseberry/uploads/aws-security-monitoring_reghtjy)

**Key Services Used:**

- AWS CloudTrail
- AWS CloudWatch Logs + Metrics + Alarms
- AWS SNS
- AWS Secrets Manager

---

## What This Project Covers

Hands-on security lab to monitor and alert on access to sensitive secrets in AWS.

- **Stage 1 – Secret & logging**: Create a secret in Secrets Manager, enable CloudTrail, and verify that `GetSecretValue` events are logged.
- **Stage 2 – Monitoring & alerts**: Send CloudTrail logs to CloudWatch Logs, build a metric filter and alarm, wire it to SNS, then test and troubleshoot notifications.

---

## Why Monitor Secret Access

Every access to a secret (API keys, passwords, confidential data) is a potential security event.

This project shows how to:

- Track _who_ accessed a secret, _when_, and _from where_ using CloudTrail.
- Turn raw logs into actionable alerts via CloudWatch metrics, alarms, and SNS emails.

---

## 1. Project Setup & Secret Creation

**Goal**: Create a secret that you will monitor.

- Log in as IAM Admin, go to **Secrets Manager** → _Store a new secret_.
- Secret type: **Other type of secret**.
- Key: `The Secret is` → Value: any demo secret string.
- Secret name: `TopSecretInfo`, description like `Secret created for monitoring project`.
- Leave default KMS encryption and rotation disabled, then click **Store**.

**Result**: A managed, encrypted secret ready to be monitored.

![Image](http://learn.nextwork.org/overjoyed_silver_glamorous_cape_gooseberry/uploads/aws-security-monitoring_o5p6q7r8)

---

## 2. Enable CloudTrail & Generate Events

**Goal**: Record and confirm secret access events.

### Create CloudTrail trail

- Go to **CloudTrail** → _Trails_ → **Create trail**.
- Trail name: `secrets-manager-trail`.
- Storage location: **Create new S3 bucket**, e.g. `nextwork-secrets-manager-trail-yourinitials`.
- Uncheck **Log file SSE-KMS encryption** to avoid extra KMS costs.
- Log events:
  - Event type: **Management events**.
  - API activity: **Read** and **Write**.
  - Exclude AWS KMS events and RDS Data API events.
- Create trail.

### Generate secret access events

- In **Secrets Manager**, open `TopSecretInfo` → **Retrieve secret value** (console).
- Open **CloudShell** and run:
  ```
  aws secretsmanager get-secret-value --secret-id "TopSecretInfo" --region your-region-code
  ```

If this sends the email, SNS works fine; the problem lies with metrics or alarm configuration.

![Image](http://learn.nextwork.org/overjoyed_silver_glamorous_cape_gooseberry/uploads/aws-security-monitoring_s8t9u0v1)

## 3. Send Logs to CloudWatch & Create Metric

### Goal

Turn **CloudTrail logs** into a **CloudWatch metric** to track when secrets are accessed.

### Steps

1. **Enable CloudWatch Logs on the Trail**

   - Go to **CloudTrail → Trails**.
   - Open **`secrets-manager-trail`**.
   - Scroll to **CloudWatch Logs → Edit**.
   - Enable CloudWatch Logs.
   - **Log group:** `my-secretsmanager-loggroup`
   - **IAM Role:** Create new → `CloudTrailRoleForCloudWatchLogs_secrets-manager-trail`
   - Save changes.

2. **Verify Logs in CloudWatch**

   - Go to **CloudWatch → Logs → Log groups**.
   - Open **`my-secretsmanager-loggroup`**.
   - Open any log stream and verify CloudTrail events are visible.

3. **Create Metric Filter**
   - In the log group, click **Actions → Create metric filter**.
   - **Filter pattern:** `GetSecretValue`
   - Continue → **Filter name:** `GetSecretValue`
   - **Metric namespace:** `SecurityMetrics`
   - **Metric name:** `Secret is accessed`
   - **Metric value:** `1`
   - **Default value:** `0`
   - Review and create the metric filter.

**Result:** Each `GetSecretValue` event increments a custom metric named **“Secret is accessed”** in the **SecurityMetrics** namespace.

![Image](http://learn.nextwork.org/overjoyed_silver_glamorous_cape_gooseberry/uploads/aws-security-monitoring_a9b0c1d2)

---

## 4. Create CloudWatch Alarm & SNS Notifications

### Goal

Send an email alert whenever a secret is accessed.

### Steps

1. **Create Alarm**

   - In **CloudWatch → Alarms**, create alarm from the new metric:
     - **Namespace:** `SecurityMetrics`
     - **Metric name:** `Secret is accessed`
     - **Statistic:** `Average`
     - **Period:** 5 minutes
   - **Conditions:**
     - Threshold type: `Static`
     - Whenever metric is **≥ 1**

2. **Create SNS Topic & Subscribe Email**

   - In the alarm wizard, under **Notification**:
     - Create new topic → Name: `SecurityAlarms`
     - **Email endpoint:** your email address.
     - Create and proceed.
   - Alarm name: `Secret is accessed`
   - Description: `This alarm triggers when a secret in Secrets Manager is accessed.`
   - Review and create the alarm.

3. **Confirm Subscription**
   - Check your email for **AWS Notification - Subscription Confirmation**.
   - Click **Confirm subscription**.
   - Verify “Subscription confirmed!” page.

**Result:** Alarm is connected to the SNS topic, which is subscribed to your email address.

![Image](http://learn.nextwork.org/overjoyed_silver_glamorous_cape_gooseberry/uploads/aws-security-monitoring_fsdghstt)

---

## 5. Test & Troubleshoot the Monitoring System

### Goal

Verify that the email alert triggers when the secret is accessed.

### Steps

1. **Trigger via Secret Access**

   - Go to **Secrets Manager** and retrieve your secret (`TopSecretInfo`).
   - Wait **2–5 minutes** and check for an email notification.

2. **If No Email: Test Manually**
   ```bash
   aws cloudwatch set-alarm-state \
       --alarm-name "Secret is accessed" \
       --state-value ALARM \
       --state-reason "Manually triggered for testing"
   ```

If this sends the email, SNS works fine; the problem lies with metrics or alarm configuration.

3. **Fix Alarm Configuration**
   - Open **CloudWatch** → **Alarms** → Edit Secret is accessed.
   - Change Statistic from `Average` → `Sum`.
   - Optional: Change **Period** from 5 minutes → 1 minute.
   - Ensure threshold: Static, ≥ 1.
   - Save and update the alarm.

Retrieve the secret again and verify the alarm triggers.

**Result:** Fully functional monitoring pipeline:
Secrets Manager → CloudTrail → CloudWatch Logs → Metric Filter → Alarm → SNS email.

![Image](http://learn.nextwork.org/overjoyed_silver_glamorous_cape_gooseberry/uploads/aws-security-monitoring_ageraergearge)

---

## Direct CloudTrail → SNS

### Goal

Compare CloudWatch-based alerts vs direct CloudTrail SNS notifications.

### Steps

- In CloudTrail → Trails → secrets-manager-trail → Edit, enable SNS notification delivery.
- Choose Use existing SNS topic → SecurityAlarms.
- Save changes.
- Retrieve the secret again and check your inbox.
  **Note: You’ll receive frequent CloudTrail SNS emails for every log file delivery — not just secret access events.**
- Disable SNS notification delivery after testing to stop the flood of messages.

![Image](http://learn.nextwork.org/overjoyed_silver_glamorous_cape_gooseberry/uploads/aws-security-monitoring_d7e8f9g0)

**Takeaway:**

- CloudWatch Alarm path: Low-noise, targeted alerts (best for humans).
- Direct CloudTrail SNS: High-volume, detailed notifications (best for automated processing or machine integration).

---

## 6. Cleanup

### To avoid unnecessary AWS charges, delete all created resources:

**AWS Secrets Manager**

- Delete secret: TopSecretInfo

**AWS CloudTrail**

- Delete trail: secrets-manager-trail

**Amazon S3**

- Delete S3 bucket used for CloudTrail logs:
- my-secrets-manager-trail

**Amazon CloudWatch**

- Delete alarm: Secret is accessed
- Delete log group: my-secretsmanager-loggroup

**Amazon SNS**

- Delete SNS topic: SecurityAlarms
- Delete the associated email subscription.

**Result: All project resources are fully cleaned up.**
