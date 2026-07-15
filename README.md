# Lab M2.06 - CloudWatch Monitoring and Alarms

**Repository:** [https://github.com/cloud-engineering-bootcamp/ce-lab-cloudwatch-monitoring-and-alarms](https://github.com/cloud-engineering-bootcamp/ce-lab-cloudwatch-monitoring-and-alarms)

**Activity Type:** Individual  
**Estimated Time:** 45-60 minutes

## Learning Objectives

- [ ] View and interpret CloudWatch metrics
- [ ] Create CloudWatch alarms bound to a specific instance
- [ ] Set up SNS email notifications
- [ ] Trigger an alarm with a real load test
- [ ] Explain how metric granularity determines alarm latency

## Prerequisites

- [ ] Completed Lab M2.01 (EC2 instance running, Amazon Linux 2023)
- [ ] AWS CLI configured on your laptop
- [ ] SSH access to your instance
- [ ] An email address you can check

---

## Introduction

In this lab you configure the standard AWS monitoring feedback loop: metric → alarm → SNS → email notification. You will also learn an important property of CloudWatch alarms: by default they respond on the order of several minutes, which is a consequence of metric granularity rather than a fault.

## Scenario

Your team runs a production EC2 instance that is not currently monitored. You have been asked to ensure that sustained high CPU utilization generates a notification, without requiring anyone to watch a dashboard.

---

## Before You Start: All Resources Must Be in the Same Region

CloudWatch metrics, CloudWatch alarms, and SNS topics are regional. An alarm in `us-east-1` cannot observe an instance in `us-east-2`, nor publish to a topic in `us-east-2`.

Set your region and confirm it:

```bash
aws configure get region
# If it is not the region your instance is in, set it:
export AWS_DEFAULT_REGION=us-east-2   # use your region
```

A region mismatch is a common cause of failure in this lab. If the alarm remains in `INSUFFICIENT_DATA`, verify this first.

---

## Your Task

**What you'll create:**
- An SNS topic with a confirmed email subscription
- A CPU utilization alarm bound to your instance
- A status check alarm
- A triggered alarm and a real email in your inbox

**Time limit:** 45-60 minutes

---

## Step-by-Step Instructions

### Step 1: Capture Your Instance ID

Retrieve the instance ID programmatically rather than copying it manually.

**From your laptop:**
```bash
# List running instances, showing each instance ID next to its Name tag
aws ec2 describe-instances \
  --filters "Name=instance-state-name,Values=running" \
  --query 'Reservations[].Instances[].[InstanceId,Tags[?Key==`Name`].Value|[0]]' \
  --output table
```

Then store it in a variable so later commands can reference it:
```bash
INSTANCE_ID=i-0abc123...    # paste yours
```

**Or from the instance itself** (Amazon Linux 2023 requires an IMDSv2 token):
```bash
# Step 1: request a short-lived session token (required by IMDSv2)
TOKEN=$(curl -sX PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 300")
# Step 2: use the token to read this instance's ID from the metadata service
curl -sH "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/instance-id
```

---

### Step 2: Create the SNS Topic and Subscribe

`create-topic` **returns the ARN** - capture it rather than typing it out.

```bash
TOPIC_ARN=$(aws sns create-topic --name ec2-alerts \
  --query TopicArn --output text)

echo "$TOPIC_ARN"
# arn:aws:sns:us-east-2:123456789012:ec2-alerts
```

The ARN format is:
```
arn:aws:sns:<region>:<account-id>:<topic-name>
```

> `create-topic` is idempotent: re-running it returns the existing ARN rather
> than producing an error, so it is safe to run again if the shell session is lost.

Subscribe your email:
```bash
aws sns subscribe --topic-arn "$TOPIC_ARN" \
  --protocol email \
  --notification-endpoint YOUR@EMAIL.com
```

### Confirm the Subscription

Check your inbox and click the confirmation link. Until the subscription is confirmed, its status is `PendingConfirmation` and no notifications will be delivered. This is a common cause of missing notifications in this lab.

Verify:
```bash
aws sns list-subscriptions-by-topic --topic-arn "$TOPIC_ARN" \
  --query 'Subscriptions[].[Protocol,Endpoint,SubscriptionArn]' --output table
```

**Expected outcome:** `SubscriptionArn` is a real ARN, **not** the string `PendingConfirmation`.

---

### 📸 Screenshot Required

**Filename**

```
screenshots/05-sns-subscription.png
```

**Capture**

Either the SNS console **Subscriptions** page showing your email subscription with status **Confirmed**, or the terminal output of `list-subscriptions-by-topic` where `SubscriptionArn` is a real ARN.

**Purpose**

Confirms the email endpoint is subscribed and confirmed, which is required for any notification to be delivered.

---

### Step 3: Understand Alarm Latency

Review this before creating the alarm, as it determines how long the alarm takes to respond.

EC2 basic monitoring publishes `CPUUtilization` once every 5 minutes, which is the finest resolution available to the alarm.

An alarm with `--period 300 --evaluation-periods 2` therefore requires two consecutive 5-minute datapoints above the threshold:

```
--period 300           = each datapoint covers 5 minutes
--evaluation-periods 2 = two consecutive datapoints are required
                       = 10+ minutes of sustained load before the alarm fires
```

A 60-second load burst is insufficient to trigger the alarm. Plan for the delay, or enable detailed monitoring (see Bonus Challenges).

---

### Step 4: Create the CPU Alarm

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name high-cpu-alarm \
  --alarm-description "Alert when CPU > 80%" \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --statistic Average \
  --period 300 \
  --evaluation-periods 2 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --dimensions Name=InstanceId,Value=$INSTANCE_ID \
  --alarm-actions "$TOPIC_ARN" \
  --ok-actions "$TOPIC_ARN"
```

What each argument controls:
- `--metric-name` / `--namespace` — which metric to watch (`CPUUtilization` in the `AWS/EC2` namespace)
- `--statistic Average` — aggregate each period by its average value
- `--period 300` — each datapoint spans 300 seconds (5 minutes)
- `--evaluation-periods 2` — require 2 consecutive breaching datapoints before firing
- `--threshold 80` / `--comparison-operator GreaterThanThreshold` — fire when the value exceeds 80
- `--dimensions` — bind the alarm to one specific instance
- `--alarm-actions` / `--ok-actions` — the SNS topic to notify when entering `ALARM` and when returning to `OK`

The `--dimensions` argument binds the alarm to a specific instance. `AWS/EC2` + `CPUUtilization` + `InstanceId=i-abc` is a distinct time series from the same metric on any other instance. Generating load on a different instance will not trigger this alarm.

> `--ok-actions` sends a notification on recovery, allowing the full
> `OK → ALARM → OK` cycle to be demonstrated.

> **Note:** `put-metric-alarm` replaces the alarm entirely rather than updating
> individual fields. Re-running it with the same `--alarm-name` overwrites the
> existing alarm, and any omitted flag is discarded. Omitting `--alarm-actions`
> produces a functioning alarm that sends no notifications. Re-supply every flag
> on each invocation.

Verify:
```bash
aws cloudwatch describe-alarms --alarm-names high-cpu-alarm \
  --query 'MetricAlarms[0].[StateValue,Dimensions,AlarmActions]'
```

**Expected outcome:** State is `OK` or `INSUFFICIENT_DATA` (normal for a brand-new alarm).

---

### 📸 Screenshot Required

**Filename**

```
screenshots/02-alarm-creation.png
```

**Capture**

The CloudWatch console **Alarms** page showing `high-cpu-alarm`, with its threshold (CPU > 80%), period, and bound instance dimension visible. The terminal output of `describe-alarms` is also acceptable.

**Purpose**

Documents that the alarm was created with the correct metric, threshold, and instance binding.

---

### Step 5: Create the Status Check Alarm

This one fires if AWS's own health checks on the instance fail (hardware or OS-level problems).

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name instance-status-check-alarm \
  --alarm-description "Alert when instance status check fails" \
  --metric-name StatusCheckFailed \
  --namespace AWS/EC2 \
  --statistic Maximum \
  --period 60 \
  --evaluation-periods 2 \
  --threshold 1 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --dimensions Name=InstanceId,Value=$INSTANCE_ID \
  --alarm-actions "$TOPIC_ARN"
```

This alarm uses `Maximum` rather than `Average`. A status check is a 0/1 signal, and any failure is significant, so the maximum is the appropriate statistic.

---

### Step 6: Stress Test the Instance

SSH to your instance, then install and run `stress`:

```bash
# Install the stress tool
sudo dnf install -y stress

# Load every vCPU for 15 minutes (long enough to cross two 5-minute periods)
stress --cpu 2 --timeout 900s
```

In a second terminal, confirm the CPU is saturated:
```bash
top      # press q to quit
```

### Wait 10-12 minutes

The delay is expected; see Step 3 for the explanation.

Monitor the alarm state without refreshing the console:
```bash
watch -n 30 'aws cloudwatch describe-alarms --alarm-names high-cpu-alarm \
  --query "MetricAlarms[0].StateValue" --output text'
```

**Expected outcome:** The state transitions from `OK` to `ALARM`, and a notification email is delivered.

### 📸 Screenshot Required

**Filename**

```
screenshots/01-cloudwatch-metrics-dashboard.png
```

**Capture**

The CloudWatch **Metrics** graph for `CPUUtilization` on your instance, showing utilization rising above the 80% threshold during the load test.

**Purpose**

Shows the underlying metric data that drives the alarm state change.

---

### 📸 Screenshot Required

**Filename**

```
screenshots/03-alarm-triggered.png
```

**Capture**

The CloudWatch console showing `high-cpu-alarm` in the **In alarm** (ALARM) state.

**Purpose**

Captures the alarm firing in response to sustained load - the core outcome of the lab.

---

### 📸 Screenshot Required

**Filename**

```
screenshots/06-email-notification.png
```

**Capture**

The notification email received in your inbox, showing the alarm name and the transition to the `ALARM` state.

**Purpose**

Confirms the metric → alarm → SNS → email notification path works end to end.

---

`stress` stops on its own when its `--timeout` elapses. To stop it early, press `Ctrl+C` in its terminal.

Within approximately 10 minutes the alarm returns to `OK` and a second email is delivered.

### 📸 Screenshot Required

**Filename**

```
screenshots/04-alarm-resolved.png
```

**Capture**

The CloudWatch console showing `high-cpu-alarm` back in the **OK** state after the load stopped.

**Purpose**

Demonstrates the full `OK → ALARM → OK` recovery cycle.

---

### Step 7: Record the Timeline

For your submission, capture the alarm's state history, which provides more precise evidence than a screenshot:

```bash
aws cloudwatch describe-alarm-history \
  --alarm-name high-cpu-alarm \
  --history-item-type StateUpdate \
  --query 'AlarmHistoryItems[].[Timestamp,HistorySummary]' \
  --output table
```

**Expected outcome:** A table showing the `OK → ALARM` and `ALARM → OK` transitions with timestamps.

---

## 📤 What to Submit

**Submission Type:** GitHub Repository

Create a **public** GitHub repository named `ce-lab-cloudwatch-monitoring` containing:

1. **CloudWatch Configuration:**
   - `alarm-config.json` (output of `describe-alarms`)
   - `sns-topic-details.txt` (topic ARN and subscription info)
   - `metrics-query.txt` (any custom metric queries you ran)

2. **Screenshots** (in `screenshots/` folder):
   - CloudWatch Metrics graph showing the CPU spike
   - Alarm configuration in console
   - Alarm in **ALARM** state
   - Alarm back in **OK** state
   - SNS subscription confirmed
   - The notification email you received

3. **Test Documentation:**
   - `stress-test-log.txt` (what you ran, and `top` output)
   - `alarm-timeline.txt` (output of `describe-alarm-history`)

4. **Lab Report** (`README.md`):
   - Step-by-step process
   - Why monitoring matters
   - **Why your alarm took ~10 minutes to fire** (metric granularity)
   - How SNS notifications work
   - What you observed during the stress test
   - Real-world use cases for CloudWatch alarms

**Structure:**
```
ce-lab-cloudwatch-monitoring/
├── README.md
├── configs/
│   ├── alarm-config.json
│   ├── sns-topic-details.txt
│   └── metrics-query.txt
├── test-results/
│   ├── stress-test-log.txt
│   └── alarm-timeline.txt
└── screenshots/
    ├── 01-cloudwatch-metrics-dashboard.png
    ├── 02-alarm-creation.png
    ├── 03-alarm-triggered.png
    ├── 04-alarm-resolved.png
    ├── 05-sns-subscription.png
    └── 06-email-notification.png
```

> ⚠️ **This repository is public.** Your SNS topic ARN contains your 12-digit AWS
> **account ID**. It is not a credential, but redact it as `123456789012` if you
> would rather not publish it.

---

## Bonus Challenges

- [ ] **Reduce alarm latency to approximately 2 minutes.** Enable detailed monitoring on the instance (1-minute metrics), then rebuild the alarm with `--period 60 --evaluation-periods 2`. Document the cost tradeoff.
- [ ] **Build a CloudWatch dashboard** with CPU, network, and disk widgets
- [ ] **Add a memory alarm.** Memory is not a default EC2 metric; the CloudWatch agent must be installed to publish it. Explain why AWS cannot observe memory usage by default.
- [ ] **Add an alarm action that stops the instance** instead of sending email, as a cost guardrail

---

## Common Issues & Solutions

### Issue 1: The alarm never fires

**Causes, in order of likelihood:**
1. **You did not wait long enough.** 10+ minutes minimum. See Step 3.
2. **Region mismatch** between instance, alarm, and topic.
3. **Wrong `InstanceId`** in `--dimensions` - you are stressing a different box.
4. The load is not actually pegging the CPU - check with `top`.

---

### Issue 2: The alarm fires but no email arrives

**Cause:** The subscription was never confirmed.

**Solution:**
```bash
aws sns list-subscriptions-by-topic --topic-arn "$TOPIC_ARN"
```
If `SubscriptionArn` says `PendingConfirmation`, check your inbox (and spam) and click the link.

Test the topic directly, bypassing the alarm entirely:
```bash
aws sns publish --topic-arn "$TOPIC_ARN" --message "test"
```
If this does not arrive, the problem is SNS, not CloudWatch.

---

### Issue 3: Alarm stuck in INSUFFICIENT_DATA

**Causes:**
1. The instance is stopped (no metrics being published)
2. Region mismatch
3. Wrong instance ID in the dimensions
4. The alarm is brand new - give it one full period

---

### Issue 4: The alarm fired but the metric graph looks flat

**Cause:** Basic monitoring only publishes a datapoint every 5 minutes, so a short spike can fall between samples.

**Solution:** Keep `stress` running for the full `--timeout` (at least 10-12 minutes), or enable detailed monitoring for 1-minute datapoints (see Bonus Challenges).

---

## Learning Reflections

Answer these in your README:

1. Why did the alarm take approximately 10 minutes to fire? Explain in terms of `period`, `evaluation-periods`, and metric granularity.
2. Why can CloudWatch observe CPU utilization but not memory usage by default? What does this indicate about where these metrics originate?
3. The threshold was set at 80%. What are the consequences of setting it at 50%, or at 95%? What is the operational value of an alarm that fires frequently?
4. What is the difference between `--alarm-actions` and `--ok-actions`, and why might both be useful?

---

## Cleanup

Alarms and SNS topics incur negligible cost, but remove them regardless:

```bash
aws cloudwatch delete-alarms --alarm-names high-cpu-alarm instance-status-check-alarm
aws sns delete-topic --topic-arn "$TOPIC_ARN"
```

- [ ] Load test stopped (`stress` timed out or `Ctrl+C`); do not leave the CPU under sustained load
- [ ] Instance stopped (later labs reuse it)

---

## Screenshot Checklist

Before submitting, verify that the following screenshots are present in the `screenshots/` folder:

- [ ] `screenshots/01-cloudwatch-metrics-dashboard.png`
- [ ] `screenshots/02-alarm-creation.png`
- [ ] `screenshots/03-alarm-triggered.png`
- [ ] `screenshots/04-alarm-resolved.png`
- [ ] `screenshots/05-sns-subscription.png`
- [ ] `screenshots/06-email-notification.png`

---

## Submission Checklist

- [ ] GitHub repository created and **public**
- [ ] SNS subscription **confirmed** (not PendingConfirmation)
- [ ] CPU alarm bound to the correct `InstanceId`
- [ ] Status check alarm created
- [ ] Alarm observed in **ALARM** state
- [ ] Notification email received and screenshotted
- [ ] Alarm returned to **OK**
- [ ] `alarm-timeline.txt` captured from `describe-alarm-history`
- [ ] Load test stopped
- [ ] Repository URL submitted

---

## Grading: 100 points

| Criteria | Points |
|----------|--------|
| **Alarm creation and configuration** (correct dimensions) | 30 |
| **SNS notifications working** (email received) | 25 |
| **Stress test and triggered alarm** (full OK → ALARM → OK) | 25 |
| **Documentation** (incl. explaining alarm latency) | 20 |
| **Total** | **100** |
