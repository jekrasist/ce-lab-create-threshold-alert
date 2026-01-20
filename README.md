# Lab M6.02 - Create Basic Threshold Alert

**Repository:** [https://github.com/cloud-engineering-bootcamp/ce-lab-create-threshold-alert](https://github.com/cloud-engineering-bootcamp/ce-lab-create-threshold-alert)

**Activity Type:** Individual  
**Estimated Time:** 30-45 minutes

## Learning Objectives

- [ ] Create CloudWatch alarms with appropriate thresholds
- [ ] Configure SNS notifications for alerts
- [ ] Test alert triggering
- [ ] Understand alarm states (OK, ALARM, INSUFFICIENT_DATA)
- [ ] Implement tiered alerting (warning vs critical)

## Your Task

Set up trigger-based alerts using CloudWatch Alarms that notify you via email when resources exceed thresholds.

**Success Criteria:**
- CloudWatch alarm created
- SNS topic configured with email subscription
- Alert triggers when threshold exceeded
- Email notification received

## Quick Start

```bash
# 1. Create SNS topic
aws sns create-topic --name CloudWatchAlerts

# 2. Subscribe your email
aws sns subscribe \
  --topic-arn arn:aws:sns:REGION:ACCOUNT-ID:CloudWatchAlerts \
  --protocol email \
  --notification-endpoint your-email@example.com

# 3. Confirm subscription (check your email)

# 4. Create CloudWatch alarm
aws cloudwatch put-metric-alarm \
  --alarm-name HighCPUUtilization \
  --alarm-description "Alert when CPU exceeds 80%" \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --statistic Average \
  --period 300 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 \
  --alarm-actions arn:aws:sns:REGION:ACCOUNT-ID:CloudWatchAlerts
```

## 📤 What to Submit

**Submission Type:** GitHub Repository

Create a **public** GitHub repository named `ce-lab-threshold-alerts` containing:

### Required Files

**1. README.md**
- Overview of alerting strategy
- Threshold justifications
- Testing methodology
- Screenshots of alerts

**2. Alert Configurations** (`config/`)
- `sns-topic-config.txt` - SNS topic details
- `alarm-configs.txt` - All alarm configurations
- `thresholds.md` - Threshold decisions and rationale

**3. Testing Documentation** (`tests/`)
- `test-plan.md` - How you tested each alarm
- `test-results.md` - Results of testing
- Screenshots of triggered alarms

**4. Screenshots** (`screenshots/`)
- SNS topic created
- Email subscription confirmed
- Alarm in OK state
- Alarm in ALARM state
- Email notification received

### Repository Structure
```
ce-lab-threshold-alerts/
├── README.md
├── config/
│   ├── sns-topic-config.txt
│   ├── alarm-configs.txt
│   └── thresholds.md
├── tests/
│   ├── test-plan.md
│   └── test-results.md
└── screenshots/
    ├── 01-sns-topic.png
    ├── 02-email-confirmation.png
    ├── 03-alarm-ok-state.png
    ├── 04-alarm-alarm-state.png
    └── 05-email-notification.png
```

## Grading: 100 points

- SNS topic and subscription: 20pts
- CloudWatch alarms created: 35pts
- Testing and triggering: 25pts
- Documentation and rationale: 20pts

## Detailed Instructions

### Part 1: Create SNS Topic and Subscribe (10 min)

**Create SNS Topic:**
```bash
# Create topic
TOPIC_ARN=$(aws sns create-topic \
  --name CloudWatchAlerts \
  --tags Key=Environment,Value=Production \
  --query 'TopicArn' \
  --output text)

echo "Topic ARN: $TOPIC_ARN"

# Save topic ARN for later use
echo $TOPIC_ARN > sns-topic-arn.txt
```

**Subscribe Email:**
```bash
# Subscribe your email
aws sns subscribe \
  --topic-arn $TOPIC_ARN \
  --protocol email \
  --notification-endpoint your-email@example.com

# You'll receive a confirmation email - click the link!
```

**Verify Subscription:**
```bash
# List subscriptions
aws sns list-subscriptions-by-topic --topic-arn $TOPIC_ARN

# Should show Status: Confirmed after you click email link
```

**Test SNS Topic:**
```bash
# Send test message
aws sns publish \
  --topic-arn $TOPIC_ARN \
  --subject "Test Alert" \
  --message "This is a test alert from CloudWatch"

# Check your email!
```

### Part 2: Create EC2 CPU Alarm (10 min)

**High CPU Alarm:**
```bash
# Get your EC2 instance ID
INSTANCE_ID=$(aws ec2 describe-instances \
  --filters "Name=instance-state-name,Values=running" \
  --query 'Reservations[0].Instances[0].InstanceId' \
  --output text)

# Create alarm
aws cloudwatch put-metric-alarm \
  --alarm-name HighCPUUtilization \
  --alarm-description "Alert when CPU exceeds 80% for 10 minutes" \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --statistic Average \
  --period 300 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 \
  --dimensions Name=InstanceId,Value=$INSTANCE_ID \
  --alarm-actions $TOPIC_ARN \
  --ok-actions $TOPIC_ARN \
  --treat-missing-data notBreaching
```

**Explanation:**
- **Period: 300 seconds** (5 minutes)
- **Evaluation periods: 2** (10 minutes total)
- **Threshold: 80%**
- **Comparison: GreaterThan**

**Verify Alarm:**
```bash
# Describe alarm
aws cloudwatch describe-alarms --alarm-names HighCPUUtilization

# Check alarm state
aws cloudwatch describe-alarms \
  --alarm-names HighCPUUtilization \
  --query 'MetricAlarms[0].StateValue'
```

### Part 3: Create Additional Alarms (10 min)

**Memory Alarm (Custom Metric):**
```bash
aws cloudwatch put-metric-alarm \
  --alarm-name HighMemoryUtilization \
  --alarm-description "Alert when memory exceeds 85%" \
  --metric-name MemoryUtilization \
  --namespace CWAgent \
  --statistic Average \
  --period 300 \
  --threshold 85 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 \
  --dimensions Name=InstanceId,Value=$INSTANCE_ID \
  --alarm-actions $TOPIC_ARN
```

**Disk Space Alarm:**
```bash
aws cloudwatch put-metric-alarm \
  --alarm-name LowDiskSpace \
  --alarm-description "Alert when disk usage exceeds 80%" \
  --metric-name disk_used_percent \
  --namespace CWAgent \
  --statistic Average \
  --period 300 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 1 \
  --dimensions Name=InstanceId,Value=$INSTANCE_ID Name=path,Value=/ \
  --alarm-actions $TOPIC_ARN
```

**Application Error Rate Alarm (from Logs):**
```bash
# First, create metric filter from logs
aws logs put-metric-filter \
  --log-group-name /aws/application/api \
  --filter-name ErrorCount \
  --filter-pattern '[timestamp, request_id, level = "ERROR", ...]' \
  --metric-transformations \
    metricName=ErrorCount,metricNamespace=Application,metricValue=1

# Then create alarm
aws cloudwatch put-metric-alarm \
  --alarm-name HighErrorRate \
  --alarm-description "Alert when error rate exceeds 10 per 5 minutes" \
  --metric-name ErrorCount \
  --namespace Application \
  --statistic Sum \
  --period 300 \
  --threshold 10 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 1 \
  --alarm-actions $TOPIC_ARN
```

**ALB Target Response Time Alarm:**
```bash
aws cloudwatch put-metric-alarm \
  --alarm-name HighResponseTime \
  --alarm-description "Alert when P95 latency exceeds 500ms" \
  --metric-name TargetResponseTime \
  --namespace AWS/ApplicationELB \
  --statistic Average \
  --extended-statistic p95 \
  --period 300 \
  --threshold 0.5 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 \
  --dimensions Name=LoadBalancer,Value=app/your-alb/123456 \
  --alarm-actions $TOPIC_ARN
```

### Part 4: Test Alarm Triggering (10 min)

**Test CPU Alarm:**

**Method 1: Stress Test (Linux):**
```bash
# Install stress
sudo apt-get install stress -y

# Generate CPU load (run for 15 minutes to trigger 2 evaluation periods)
stress --cpu 4 --timeout 900s

# Monitor in another terminal
watch -n 10 "aws cloudwatch describe-alarms --alarm-names HighCPUUtilization --query 'MetricAlarms[0].StateValue'"
```

**Method 2: Python Script:**
```python
# cpu_load.py
import multiprocessing
import time

def busy_loop():
    while True:
        x = 0
        for i in range(10000000):
            x += i

if __name__ == '__main__':
    # Use all CPU cores
    processes = []
    for _ in range(multiprocessing.cpu_count()):
        p = multiprocessing.Process(target=busy_loop)
        p.start()
        processes.append(p)
    
    # Run for 15 minutes
    time.sleep(900)
    
    # Stop processes
    for p in processes:
        p.terminate()
```

**Test Error Rate Alarm:**
```bash
# Generate errors in your application
for i in {1..20}; do
  curl http://localhost:5000/error &
done

# Wait 5 minutes and check alarm state
aws cloudwatch describe-alarms --alarm-names HighErrorRate
```

### Part 5: Understand Alarm States (5 min)

**Alarm States:**
- **OK:** Metric is within threshold
- **ALARM:** Metric has breached threshold
- **INSUFFICIENT_DATA:** Not enough data to evaluate

**Check Alarm History:**
```bash
# Get alarm history
aws cloudwatch describe-alarm-history \
  --alarm-name HighCPUUtilization \
  --history-item-type StateUpdate \
  --max-records 10

# Get current state
aws cloudwatch describe-alarms \
  --alarm-names HighCPUUtilization \
  --query 'MetricAlarms[0].[StateValue, StateReason]'
```

**View Alarm in Console:**
1. Go to CloudWatch → Alarms
2. Click on alarm name
3. View graph with threshold line
4. See state change history

### Part 6: Tiered Alerting (Optional) (5 min)

**Create Warning and Critical Alarms:**

```bash
# Warning: CPU > 70%
aws cloudwatch put-metric-alarm \
  --alarm-name CPU-Warning \
  --alarm-description "Warning: CPU above 70%" \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --statistic Average \
  --period 300 \
  --threshold 70 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 \
  --dimensions Name=InstanceId,Value=$INSTANCE_ID \
  --alarm-actions $TOPIC_ARN

# Critical: CPU > 90%
aws cloudwatch put-metric-alarm \
  --alarm-name CPU-Critical \
  --alarm-description "CRITICAL: CPU above 90%" \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --statistic Average \
  --period 300 \
  --threshold 90 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 1 \
  --dimensions Name=InstanceId,Value=$INSTANCE_ID \
  --alarm-actions $TOPIC_ARN
```

## Reflection Questions

Answer these in your README:

1. **Why use 2 evaluation periods instead of 1?**
   - Hint: Avoid false alarms from temporary spikes

2. **What's the difference between Average, Maximum, and P95?**
   - Hint: Average can hide outliers, Maximum is too sensitive

3. **When should you use OK actions vs just alarm actions?**
   - Hint: Notify when problem resolves

4. **How do you determine appropriate thresholds?**
   - Hint: Baseline normal behavior first

5. **What's the cost of CloudWatch alarms?**
   - Hint: $0.10 per standard alarm per month

## Bonus Challenges

**+5 points each:**
- [ ] Create composite alarm (combines multiple alarms)
- [ ] Set up different SNS topics for different severity levels
- [ ] Create alarm that triggers Lambda function
- [ ] Implement alarm with anomaly detection
- [ ] Create dashboard showing all alarm states

## Troubleshooting

**Issue: Not receiving emails**
- Check spam folder
- Verify subscription is confirmed
- Test SNS topic directly
- Check email address is correct

**Issue: Alarm not triggering**
- Verify metric has data (check graph)
- Check evaluation periods and period
- Ensure threshold is correct
- Review "treat missing data" setting

**Issue: Too many false alarms**
- Increase evaluation periods
- Adjust threshold
- Use different statistic (P95 instead of Max)
- Add anomaly detection

## Resources

- [CloudWatch Alarms](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/AlarmThatSendsEmail.html)
- [SNS Documentation](https://docs.aws.amazon.com/sns/)
- [Alarm States and Actions](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/AlarmThatSendsEmail.html#alarm-states)
- [Composite Alarms](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Create_Composite_Alarm.html)

---

**Congratulations on setting up your first alerts!** 🚨
