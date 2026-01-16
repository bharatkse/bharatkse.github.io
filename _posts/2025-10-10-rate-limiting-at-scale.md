---
layout: blog
title: "Rate Limiting at Scale: Protecting Your Backend with AWS API Gateway"
description: "How we implemented scalable rate limiting with AWS API Gateway to protect backend services, maintain SLAs, and handle traffic spikes safely."
date: 2025-10-10
categories: [backend, aws, architecture]
tags: [rate-limiting, api-gateway, performance, reliability]
read_time: true
toc: true
toc_sticky: true
classes: wide
---


## The Problem: When Your Success Becomes Your Bottleneck

It was 2 PM on a Tuesday when our monitoring dashboard lit up like a Christmas tree. Request latency had jumped from 200ms to 2 seconds. Our database connection pool was exhausted. The Lambda functions that had been cruising along at 100ms execution time were now timing out.

The cause? A downstream service had started making aggressive API calls to our platform during their migration window. They hadn't coordinated with us, and we didn't have proper rate limiting in place.

We were getting hit with 5,000 requests per second during peak load. Our backend could handle maybe 2,000 RPS before degradation kicked in. Everything downstream suffered: database became slow, cache hit ratios dropped, and legitimate users started seeing errors.

This is a scenario many backend engineers face: success attracts traffic, but unmanaged traffic becomes a liability. Rate limiting isn't just about protecting your system—it's about guaranteeing SLA compliance and ensuring fair resource allocation across all consumers.

---

## What We Built: A Multi-Layer Rate Limiting Strategy

After incident post-mortems and architecture reviews, we implemented a three-layer rate limiting approach using AWS API Gateway:

1. **API Gateway Throttling** (Request level)
2. **CloudWatch Monitoring & Alerts** (Observability)
3. **Usage Plans & API Keys** (Consumer-based limiting)

### Layer 1: API Gateway Throttling

AWS API Gateway supports two types of throttling:

**Rate Limit:** Steady-state request rate (requests per second)  
**Burst Limit:** Temporary capacity to handle traffic spikes

```python
# CloudFormation template (YAML)
APIGatewayRestApi:
  Type: AWS::ApiGateway::RestApi
  Properties:
    Name: MyAPI
    Description: Production API with throttling

APIGatewayStage:
  Type: AWS::ApiGateway::Stage
  Properties:
    StageName: prod
    RestApiId: !Ref APIGatewayRestApi
    DeploymentId: !Ref APIGatewayDeployment
    ThrottleSettings:
      RateLimit: 2000      # 2000 requests per second
      BurstLimit: 5000     # 5000 concurrent requests
    MethodSettings:
      - ResourcePath: '/*'
        HttpMethod: '*'
        LoggingLevel: INFO
        DataTraceEnabled: false
        ThrottleSettings:
          RateLimit: 2000
          BurstLimit: 5000
    CloudWatchMetricsEnabled: true
```

**Key Insight:** Rate limit is the sustainable throughput (RPS), while burst limit handles traffic spikes. We set rate limit at 2000 RPS (our backend capacity) and burst at 5000 (to absorb sudden traffic).

When requests exceed these limits, API Gateway returns **HTTP 429 (Too Many Requests)** with a `Retry-After` header.

### Layer 2: Consumer-Based Rate Limiting with Usage Plans

Not all consumers are equal. Some are internal services, others are partners with SLAs. Usage plans let you define different rate limits for different API keys:

```python
# CloudFormation
UsagePlanBasic:
  Type: AWS::ApiGateway::UsagePlan
  Properties:
    UsagePlanName: BasicTier
    Description: Basic tier with 500 RPS limit
    Quota:
      Limit: 1000000        # 1M requests per day
      Period: DAY
    Throttle:
      RateLimit: 500        # 500 RPS
      BurstLimit: 1000      # 1000 concurrent
    ApiStages:
      - ApiId: !Ref APIGatewayRestApi
        Stage: prod

UsagePlanPremium:
  Type: AWS::ApiGateway::UsagePlan
  Properties:
    UsagePlanName: PremiumTier
    Description: Premium tier with 2000 RPS limit
    Quota:
      Limit: 100000000      # 100M requests per day
      Period: DAY
    Throttle:
      RateLimit: 2000       # 2000 RPS
      BurstLimit: 5000      # 5000 concurrent
    ApiStages:
      - ApiId: !Ref APIGatewayRestApi
        Stage: prod

# Create API Keys for each consumer
ConsumerAPIKey:
  Type: AWS::ApiGateway::ApiKey
  Properties:
    Name: consumer-acme-corp
    Description: API Key for ACME Corp (Premium tier)
    Enabled: true
    StageKeys:
      - RestApiId: !Ref APIGatewayRestApi
        StageName: prod

# Link API Key to Usage Plan
UsagePlanKey:
  Type: AWS::ApiGateway::UsagePlanKey
  Properties:
    KeyId: !Ref ConsumerAPIKey
    KeyType: API_KEY
    UsagePlanId: !Ref UsagePlanPremium
```

This approach lets you:
- Tier your API (Free, Basic, Premium)
- Track usage per consumer
- Enforce different SLAs
- Monetize if building a platform

### Layer 3: CloudWatch Monitoring & SNS Alerts

Rate limiting is only useful if you know it's happening. We integrated CloudWatch with SNS for real-time alerting:

```python
# Python code to set up monitoring (Boto3)
import boto3

cloudwatch = boto3.client('cloudwatch')
sns = boto3.client('sns')

# Create SNS topic for alerts
topic_response = sns.create_topic(Name='api-gateway-alerts')
topic_arn = topic_response['TopicArn']

# Subscribe to topic
sns.subscribe(
    TopicArn=topic_arn,
    Protocol='email',
    Endpoint='oncall@company.com'
)

# Create alarm for 429 errors
cloudwatch.put_metric_alarm(
    AlarmName='APIGateway-HighHTTP429-Rate',
    MetricName='Count',
    Namespace='AWS/ApiGateway',
    Statistic='Sum',
    Period=300,  # 5 minutes
    EvaluationPeriods=1,
    Threshold=100,  # Alert if >100 429s in 5 min
    ComparisonOperator='GreaterThanThreshold',
    Dimensions=[
        {
            'Name': 'ApiName',
            'Value': 'MyAPI'
        },
        {
            'Name': 'Stage',
            'Value': 'prod'
        }
    ],
    AlarmActions=[topic_arn],
    TreatMissingData='notBreaching'
)

# Create alarm for request latency (precursor to rate limiting)
cloudwatch.put_metric_alarm(
    AlarmName='APIGateway-HighLatency',
    MetricName='Latency',
    Namespace='AWS/ApiGateway',
    Statistic='Average',
    Period=300,
    EvaluationPeriods=2,
    Threshold=1000,  # Alert if avg latency >1 second
    ComparisonOperator='GreaterThanThreshold',
    Dimensions=[
        {
            'Name': 'ApiName',
            'Value': 'MyAPI'
        }
    ],
    AlarmActions=[topic_arn]
)
```

**Critical Monitoring Metrics:**

- **Count (429 errors):** How many requests are being throttled?
- **Latency:** Is the backend struggling before hitting rate limits?
- **4XXError & 5XXError:** Are legitimate errors spiking?
- **IntegrationLatency:** Backend response time (excludes API Gateway overhead)

We configured dashboards to visualize these in real-time:

```python
# Create CloudWatch dashboard
dashboard_body = {
    'widgets': [
        {
            'type': 'metric',
            'properties': {
                'metrics': [
                    ['AWS/ApiGateway', 'Count', {'stat': 'Sum'}],
                    ['.', 'Latency', {'stat': 'Average'}],
                    ['.', '4XXError', {'stat': 'Sum'}],
                    ['.', 'IntegrationLatency', {'stat': 'Average'}]
                ],
                'period': 300,
                'stat': 'Average',
                'region': 'us-east-1',
                'title': 'API Gateway Metrics'
            }
        }
    ]
}

cloudwatch.put_dashboard(
    DashboardName='APIGateway-Monitoring',
    DashboardBody=json.dumps(dashboard_body)
)
```

---

## The Results: What Changed

After implementing this three-layer strategy:

**Before:**
- HTTP 429 errors: ~0 (no rate limiting, system would degrade)
- Peak request latency: 2+ seconds
- Database connection pool exhaustion: Frequent
- SLA violations: Monthly
- Incident response time: 30+ minutes

**After:**
- HTTP 429 errors: Caught and alerted within 1 minute
- Peak request latency: 200-300ms (stable)
- Database connection pool: 60-70% max utilization
- SLA violations: 0
- Incident response time: <1 minute (from alert to mitigation)

**Quantified Improvements:**
- **40% increase in sustainable throughput** (from handling 2000 RPS gracefully to 2800 RPS)
- **30% reduction in MTTR** (mean time to resolution)
- **100% SLA compliance** (previously 98%)

The burst limit also proved valuable—legitimate traffic spikes were absorbed without error. Aggressive consumers were throttled with clear feedback (429 status code).

---

## Lessons Learned & Best Practices

### 1. Set Realistic Limits Based on Capacity Testing

Don't guess. Load test your backend first:

```bash
# Using Apache Bench to find breaking point
ab -n 100000 -c 1000 https://api.example.com/health

# OR using wrk for more detailed metrics
wrk -t12 -c400 -d30s https://api.example.com/endpoint
```

We learned our breaking point was 2200 RPS (with database latency). We set limits at 2000 for safety margin.

### 2. Communicate Limits to Consumers

Rate limit headers tell clients what's happening:

```python
# Your API response headers should include:
X-RateLimit-Limit: 2000
X-RateLimit-Remaining: 1867
X-RateLimit-Reset: 1704067200

# Clients can use this to implement smart backoff:
remaining = int(response.headers['X-RateLimit-Remaining'])
if remaining < 100:
    time.sleep(5)  # Back off before hitting limit
```

### 3. Implement Client-Side Exponential Backoff

Rate limiting is pointless if clients retry blindly. Use exponential backoff with jitter:

```python
import time
import random

def call_api_with_backoff(url, max_retries=5):
    for attempt in range(max_retries):
        response = requests.get(url)
        
        if response.status_code != 429:
            return response
        
        # Exponential backoff: 2^attempt + random jitter
        wait_time = (2 ** attempt) + random.uniform(0, 1)
        print(f"Rate limited. Retrying in {wait_time:.2f}s")
        time.sleep(wait_time)
    
    raise Exception("Max retries exceeded")
```

### 4. Don't Penalize Your Own Services

Use API keys to identify internal traffic and apply different limits:

```python
# Differentiate based on API key/source
if request.headers.get('X-Internal-Key'):
    # Internal service - don't rate limit
    apply_rate_limit = False
else:
    # External consumer - apply rate limit
    apply_rate_limit = True
```

### 5. Monitor the Monitors

Set up alarms for the alarms:

```python
# Alert if 429 rate drops to 0 (indicates monitoring failure)
cloudwatch.put_metric_alarm(
    AlarmName='APIGateway-NoThrottling',
    MetricName='Count',
    Statistic='Sum',
    Threshold=0,
    ComparisonOperator='LessThanOrEqualToThreshold',
    # This would indicate rate limiting is broken
)
```

---

## Implementation Checklist

- [ ] Load test backend to find sustainable RPS
- [ ] Configure API Gateway throttling (rate + burst limits)
- [ ] Set up usage plans for different consumer tiers
- [ ] Create API keys for major consumers
- [ ] Configure CloudWatch alarms (429 errors, latency)
- [ ] Set up SNS notifications to on-call team
- [ ] Create CloudWatch dashboard for monitoring
- [ ] Document rate limit policy and communicate to consumers
- [ ] Implement client-side exponential backoff in SDK/documentation
- [ ] Test rate limiting behavior with load test tools
- [ ] Monitor for 2 weeks and tune thresholds based on patterns

---

## Key Takeaways

Rate limiting isn't about being mean to your users—it's about protecting the system for everyone. A well-tuned rate limiting strategy:

1. **Prevents cascading failures** - One bad consumer can't take down the system
2. **Guarantees fair access** - Everyone gets their fair share of capacity
3. **Provides early warning** - 429 errors alert you before backend fails
4. **Enables monetization** - Different tiers for different customer values
5. **Reduces incident response time** - Automated alerts catch problems in <1 minute

At our scale (10K+ monthly orders, 2800+ peak RPS), rate limiting became as important as auto-scaling. It's a critical piece of infrastructure for any production system.

---

## Related Resources

- [AWS API Gateway Rate Limiting Documentation](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-request-throttling.html)
- [CloudWatch Custom Metrics Guide](https://docs.aws.amazon.com/AmazonCloudWatch/latest/events/WhatIsCloudWatch.html)
- [Exponential Backoff Best Practices](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/)
- [GitHub: AWS Rate Limiting Examples](https://github.com/bharatkse/aws-api-gateway-patterns)

---

**Questions or feedback?** Drop a comment below or reach out on [LinkedIn](https://linkedin.com/in/bharat-kumar28).
