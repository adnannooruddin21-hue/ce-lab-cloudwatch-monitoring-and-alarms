# CloudWatch Monitoring and Alarms Lab - Solution

**Student Name:** [Your Name]  
**Date Completed:** [Date]

---

# Environment Details

| Item | Value |
|------|-------|
| Instance ID | [i-xxxxxxxxxxxxx] |
| Region | [us-east-2] |
| SNS Topic Name | [ec2-alerts] |
| SNS Topic ARN | [arn:aws:sns:region:123456789012:ec2-alerts] |
| Notification Email | [you@email.com] |
| CPU Alarm Name | [high-cpu-alarm] |
| Status Check Alarm | [instance-status-check-alarm] |

- [ ] All resources (instance, alarm, topic) are in the **same region**

---

# Step 1: Capture Your Instance ID

- [ ] Retrieved the instance ID with `describe-instances` (or IMDSv2 on the instance)
- [ ] Stored it in `INSTANCE_ID`

**My instance ID:** `_______________________`

---

# Step 2: Create the SNS Topic and Subscribe

## Screenshot 5 – SNS Subscription

```
screenshots/05-sns-subscription.png
```

![SNS Subscription](screenshots/05-sns-subscription.png)

---

- [ ] Topic `ec2-alerts` created, ARN captured in `TOPIC_ARN`
- [ ] Subscribed my email with `aws sns subscribe`
- [ ] Clicked the confirmation link in my inbox
- [ ] `list-subscriptions-by-topic` shows a real `SubscriptionArn` (**not** `PendingConfirmation`)

---

# Step 3: Understand Alarm Latency

- [ ] I understand basic monitoring publishes `CPUUtilization` every 5 minutes
- [ ] `--period 300 --evaluation-periods 2` = 10+ minutes of sustained load before firing

---

# Step 4: Create the CPU Alarm

## Screenshot 2 – Alarm Creation

```
screenshots/02-alarm-creation.png
```

![Alarm Creation](screenshots/02-alarm-creation.png)

---

- [ ] `high-cpu-alarm` created with `put-metric-alarm`
- [ ] Bound to my instance via `--dimensions Name=InstanceId,Value=$INSTANCE_ID`
- [ ] `--alarm-actions` and `--ok-actions` both point to my SNS topic
- [ ] `describe-alarms` shows state `OK` or `INSUFFICIENT_DATA`

---

# Step 5: Create the Status Check Alarm

- [ ] `instance-status-check-alarm` created for `StatusCheckFailed`
- [ ] Uses `--statistic Maximum` (a status check is a 0/1 signal)

---

# Step 6: Stress Test the Instance

## Screenshot 1 – CloudWatch Metrics (CPU spike)

```
screenshots/01-cloudwatch-metrics-dashboard.png
```

![CloudWatch Metrics](screenshots/01-cloudwatch-metrics-dashboard.png)

## Screenshot 3 – Alarm Triggered (ALARM state)

```
screenshots/03-alarm-triggered.png
```

![Alarm Triggered](screenshots/03-alarm-triggered.png)

## Screenshot 6 – Email Notification

```
screenshots/06-email-notification.png
```

![Email Notification](screenshots/06-email-notification.png)

## Screenshot 4 – Alarm Resolved (OK state)

```
screenshots/04-alarm-resolved.png
```

![Alarm Resolved](screenshots/04-alarm-resolved.png)

---

- [ ] Installed and ran `stress --cpu 2 --timeout 900s` on the instance
- [ ] Confirmed the CPU was saturated with `top`
- [ ] Waited 10–12 minutes
- [ ] Alarm transitioned `OK → ALARM`
- [ ] Received the notification email
- [ ] After load stopped, alarm returned to `OK` and a second email arrived

---

# Step 7: Record the Timeline

- [ ] Captured `describe-alarm-history` output to `alarm-timeline.txt`

### Alarm state transitions (from `describe-alarm-history`)

```text
[Paste the OK → ALARM and ALARM → OK rows with timestamps]
```

---

# Cleanup

- [ ] Load test stopped (`stress` timed out or `Ctrl+C`)
- [ ] Deleted the alarms (`delete-alarms`)
- [ ] Deleted the SNS topic (`delete-topic`)
- [ ] Instance **stopped** (not terminated — later labs reuse it)

---

# Submission Checklist

Repository name: `ce-lab-cloudwatch-monitoring` (**public**)

- [ ] `configs/` files committed (alarm config, SNS details, metrics query)
- [ ] `test-results/` files committed (stress-test log, alarm timeline)
- [ ] All 6 screenshots present
- [ ] `README.md` complete with the required explanations
- [ ] SNS subscription **confirmed** (not PendingConfirmation)
- [ ] CPU alarm bound to the correct `InstanceId`
- [ ] Status check alarm created
- [ ] Alarm observed in **ALARM** and back in **OK**
- [ ] Notification email received and screenshotted
- [ ] Account ID redacted (if I chose to)
- [ ] Repository URL submitted
