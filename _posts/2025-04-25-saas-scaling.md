---
layout: blog
title: "Scaling a SaaS Platform to 10K+ Monthly Orders: Architecture Lessons from Production"
description: "Architecture lessons from scaling a SaaS platform to 10K+ monthly orders, covering performance bottlenecks, data integrity, and operational trade-offs."
date: 2025-04-25
categories: [backend, architecture, scalability]
tags: [saas, django, postgresql, celery, performance, scaling]
read_time: true
toc: true
toc_sticky: true
classes: wide
---

## The Journey: From Idea to 10K+ Monthly Orders

Three years ago, we launched Reshyne—a SaaS platform for e-commerce order management. What started as a weekend project to solve our own operational pain point became a multi-million dollar business processing 10,000+ orders per month.

The path to scale wasn't smooth. We hit numerous bottlenecks:

- **Payment Processing:** Stripe integration that couldn't handle simultaneous checkout
- **Shipping Automation:** Manual carrier API integrations that broke regularly
- **Data Integrity:** Race conditions in order status updates
- **Performance:** Database queries taking 30+ seconds during peak hours
- **Operations:** Spending 60+ hours weekly on support issues

This article documents what we learned building a SaaS product that scales—from architectural decisions to operational best practices.

---

## Architecture: What Actually Works at Scale

### The Stack We Chose (and Why)

```
Frontend (React) → Django REST API → PostgreSQL
                       ↓
              Celery (Background Jobs)
                       ↓
              Stripe (Payments) + Carrier APIs (Shipping)
                       ↓
              Redis (Caching & Job Queue)
```

**Why Django?**
- Mature ecosystem (10+ years proven)
- Built-in ORM prevents N+1 queries
- Django REST Framework validates inputs automatically
- Admin interface saves days of development

**Why PostgreSQL?**
- ACID transactions prevent data corruption
- jsonb for flexible order metadata
- Excellent for e-commerce (relational data)
- PostGIS for location-based features

**Why Celery?**
- Decouples payment processing from request/response cycle
- Handles retries and failure automatically
- Scales independently from web server

### Critical Database Design Decisions

The biggest performance bottleneck we hit was the `orders` table hitting 10M+ rows. We weren't prepared.

**Original Schema (The Mistake):**

```sql
CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    customer_id BIGINT,
    created_at TIMESTAMP,
    status VARCHAR(50),
    total_amount DECIMAL,
    payment_status VARCHAR(50),
    shipping_status VARCHAR(50),
    items JSONB,
    address JSONB,
    -- 20+ more columns...
);

-- This query would scan 10M rows!
SELECT * FROM orders 
WHERE customer_id = 123 
AND created_at > NOW() - INTERVAL '30 days'
ORDER BY created_at DESC;
```

**The Fix:**

```sql
-- Partition by time (monthly)
CREATE TABLE orders (
    id BIGSERIAL NOT NULL,
    customer_id BIGINT NOT NULL,
    created_at TIMESTAMP NOT NULL,
    status VARCHAR(50) NOT NULL,
    -- ... columns
    PRIMARY KEY (id, created_at)
) PARTITION BY RANGE (created_at);

-- Create partitions for each month
CREATE TABLE orders_2024_01 PARTITION OF orders
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');
CREATE TABLE orders_2024_02 PARTITION OF orders
    FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');
-- ... etc

-- Add indexes that actually get used
CREATE INDEX idx_orders_customer_created 
ON orders (customer_id, created_at DESC)
WHERE status != 'archived';

-- Optimize the query
SELECT id, customer_id, total_amount, status, created_at
FROM orders
WHERE customer_id = 123 
  AND created_at > NOW() - INTERVAL '30 days'
  AND status != 'archived'
ORDER BY created_at DESC
LIMIT 100;

-- Time: 5ms (vs 30 seconds before)
```

**Key Insights:**
1. **Partition by time** - Queries naturally filter by date
2. **Select only needed columns** - Don't fetch 20 columns if you need 5
3. **Index correctly** - Index by query pattern, not just cardinality
4. **Archive old data** - Keep hot table small

### ORM Query Optimization

Django ORM can hide database inefficiencies. We learned to debug every query:

```python
# BAD: N+1 Query Problem
orders = Order.objects.filter(customer_id=customer_id)
for order in orders:
    print(order.customer.name)  # QUERY PER ORDER!
    for item in order.items.all():  # ANOTHER QUERY PER ITEM!
        print(item.product.title)  # MORE QUERIES!

# Total queries: 1 + N + M (where N = orders, M = items)

# GOOD: Prefetch related
from django.db.models import Prefetch

orders = Order.objects.filter(
    customer_id=customer_id
).select_related(
    'customer'  # JOIN customer table
).prefetch_related(
    'items__product'  # Separate query, but batched
).values(
    'id', 'customer__name', 'item__product__title'
)

# Total queries: 2-3 (regardless of N or M)
```

**Django ORM Rules We Live By:**

```python
# Rule 1: Use select_related() for ForeignKey (1:1)
orders = Order.objects.select_related('customer')

# Rule 2: Use prefetch_related() for reverse ForeignKey (1:N)
customers = Customer.objects.prefetch_related('orders')

# Rule 3: Use only() to limit columns
orders = Order.objects.only('id', 'total_amount', 'status')

# Rule 4: Use exists() instead of count() when checking presence
if Order.objects.filter(order_id=123).exists():
    # Better than: if Order.objects.filter(order_id=123).count() > 0
    pass

# Rule 5: Use bulk_create() for inserts
items = [
    OrderItem(order_id=1, product_id=10, qty=2),
    OrderItem(order_id=1, product_id=11, qty=1),
]
OrderItem.objects.bulk_create(items, batch_size=1000)
```

---

## Payment Processing: How We Handle $1M+/Month Reliably

Stripe integration is deceptively complex. We learned this the hard way during our first Black Friday ($50K in one day).

### The Challenge: Idempotent Payment Processing

What happens if:
- Customer clicks "Pay" twice?
- Network request times out before response?
- Webhook notification is delivered twice?

Without idempotency, customers get charged twice. Or refunds fail. It's a nightmare.

### Our Solution: Idempotent Keys + Webhooks

```python
import hashlib
from django.db import transaction

class PaymentProcessor:
    def process_payment(self, order_id, amount, card_token):
        """Process payment idempotently using Stripe's idempotency key."""
        
        order = Order.objects.get(id=order_id)
        
        # Generate idempotency key (same for every request)
        idempotency_key = f"order-{order_id}-{order.created_at.timestamp()}"
        
        # Check if already processed
        existing_charge = StripeCharge.objects.filter(
            order=order,
            status__in=['succeeded', 'pending']
        ).first()
        
        if existing_charge:
            logger.warning(f"Order {order_id} already processed")
            return existing_charge.stripe_id
        
        try:
            # Stripe won't process same key twice
            charge = stripe.Charge.create(
                amount=int(amount * 100),  # Convert to cents
                currency='usd',
                source=card_token,
                idempotency_key=idempotency_key,
                metadata={'order_id': order_id},
                statement_descriptor='RESHYNE ORDER'
            )
            
            # Save charge immediately (before webhook)
            with transaction.atomic():
                charge_obj = StripeCharge.objects.create(
                    order=order,
                    stripe_id=charge.id,
                    amount=amount,
                    status='pending',
                    response=charge
                )
                
                order.payment_status = 'pending'
                order.stripe_charge_id = charge.id
                order.save(update_fields=['payment_status', 'stripe_charge_id'])
            
            logger.info(f"Charge created: {charge.id} for order {order_id}")
            return charge.id
            
        except stripe.error.CardError as e:
            # Card was declined
            logger.error(f"Card declined: {e.user_message}")
            raise PaymentError(e.user_message)
        
        except stripe.error.RateLimitError:
            # Too many requests - retry later
            logger.warning("Stripe rate limit hit, queuing for retry")
            raise PaymentRetryableError("Payment system overloaded, please retry")
        
        except stripe.error.InvalidRequestError as e:
            # Invalid parameters - log and fail
            logger.error(f"Invalid Stripe request: {e}")
            raise PaymentError("Payment configuration error")
```

### Webhook Handler (The Real Truth)

Webhooks are how Stripe actually tells you if payment succeeded:

```python
from django.views.decorators.csrf import csrf_exempt
from django.views.decorators.http import require_http_methods
import stripe

@csrf_exempt
@require_http_methods(["POST"])
def stripe_webhook(request):
    """
    Stripe sends webhooks for events:
    - charge.succeeded
    - charge.failed
    - charge.dispute.created
    """
    
    payload = request.body
    signature = request.META.get('HTTP_STRIPE_SIGNATURE')
    
    # Verify signature (CRITICAL for security)
    try:
        event = stripe.Webhook.construct_event(
            payload,
            signature,
            settings.STRIPE_WEBHOOK_SECRET
        )
    except ValueError:
        return JsonResponse({'error': 'Invalid payload'}, status=400)
    except stripe.error.SignatureVerificationError:
        return JsonResponse({'error': 'Invalid signature'}, status=400)
    
    # Idempotent processing (webhook may be delivered twice)
    if WebhookEvent.objects.filter(
        stripe_event_id=event['id'],
        processed=True
    ).exists():
        logger.info(f"Webhook already processed: {event['id']}")
        return JsonResponse({'status': 'ok'})
    
    # Record webhook to prevent duplicates
    webhook_event = WebhookEvent.objects.create(
        stripe_event_id=event['id'],
        event_type=event['type'],
        data=event
    )
    
    try:
        if event['type'] == 'charge.succeeded':
            handle_charge_succeeded(event['data']['object'])
        
        elif event['type'] == 'charge.failed':
            handle_charge_failed(event['data']['object'])
        
        elif event['type'] == 'charge.dispute.created':
            handle_dispute(event['data']['object'])
        
        # Mark webhook as processed
        webhook_event.processed = True
        webhook_event.save()
        
        logger.info(f"Webhook processed: {event['type']}")
        return JsonResponse({'status': 'ok'})
    
    except Exception as e:
        logger.error(f"Webhook handling failed: {str(e)}", exc_info=True)
        # Return 500 so Stripe retries
        return JsonResponse({'error': str(e)}, status=500)

def handle_charge_succeeded(charge):
    """Update order when payment succeeds."""
    
    order_id = charge['metadata'].get('order_id')
    if not order_id:
        logger.error("Charge missing order_id metadata")
        return
    
    order = Order.objects.get(id=order_id)
    
    with transaction.atomic():
        charge_obj = StripeCharge.objects.get(stripe_id=charge['id'])
        charge_obj.status = 'succeeded'
        charge_obj.save()
        
        order.payment_status = 'completed'
        order.save(update_fields=['payment_status'])
        
        # Queue shipping task
        schedule_shipping_preparation.delay(order_id)
        
        # Send confirmation email
        send_order_confirmation_email.delay(order_id)
    
    logger.info(f"Order {order_id} payment succeeded")
```

### Payment Processing Results

**Before:**
- Manual refund handling: 5+ hours/week
- Failed webhooks: 20-30 per month
- Customer complaints: "I was charged twice!"
- Stripe dispute rate: 2.5%

**After:**
- Automated idempotent processing
- Webhook retry: Automatic
- Customer complaints: 0
- Stripe dispute rate: 0.3%
- **99.9% successful transaction rate**

---

## Shipping Automation: From Manual to Automated

We integrated with 5 carriers (FedEx, UPS, DHL, USPS, ShipStation). Each has different APIs, formats, error handling.

### The Problem: Fragile API Integration

Each carrier API is unique:

```python
# FedEx rate API
fedex_response = requests.post(
    'https://apis.fedex.com/rate/v1/rates/quotes',
    json={
        'accountNumber': '...',
        'shippingChargesPayment': {...},
        'requestedShipment': {
            'shipper': {...},
            'recipient': {...},
            'pickupType': 'BUSINESS_SERVICE_CENTER',
            'rateRequestType': ['ACCOUNT', 'LIST'],
            # ... 15 more required fields
        }
    },
    headers={'Authorization': f'Bearer {fedex_token}'}
)

# UPS rate API (completely different format)
ups_response = requests.post(
    'https://onlinetools.ups.com/rate/v2/Shop',
    json={
        'UPSOnlineToolsRequest': {
            'Request': {'TransactionReference': {...}},
            'Shipment': {
                'Shipper': {...},
                'ShipTo': {...},
                'ShipFrom': {...},
                'Package': [...],
                # ... different structure entirely
            }
        }
    },
    auth=('username', 'password')
)

# Both can fail in different ways - timeouts, rate limits, auth errors
```

### The Solution: Adapter Pattern + Resilience

```python
from abc import ABC, abstractmethod
from datetime import datetime, timedelta
import backoff

class ShippingAdapter(ABC):
    """Base class for all carrier integrations."""
    
    @abstractmethod
    def get_rates(self, shipment):
        pass
    
    @abstractmethod
    def create_shipment(self, shipment, service_code):
        pass
    
    @abstractmethod
    def track_shipment(self, tracking_number):
        pass

class FedExAdapter(ShippingAdapter):
    """FedEx-specific integration."""
    
    def __init__(self, api_key, secret):
        self.api_key = api_key
        self.secret = secret
        self.base_url = 'https://apis.fedex.com'
    
    @backoff.on_exception(
        backoff.expo,
        (requests.Timeout, requests.ConnectionError),
        max_tries=3,
        factor=2
    )
    @backoff.on_predicate(
        backoff.expo,
        lambda r: r.status_code == 429,  # Rate limited
        max_tries=5,
        factor=2
    )
    def get_rates(self, shipment):
        """Get shipping rates with automatic retry on failure."""
        
        token = self._get_auth_token()
        
        payload = {
            'accountNumber': self.account_number,
            'requestedShipment': {
                'shipper': {
                    'address': {
                        'streetLines': [shipment['from_address']['street']],
                        'city': shipment['from_address']['city'],
                        'stateOrProvinceCode': shipment['from_address']['state'],
                        'postalCode': shipment['from_address']['zip'],
                        'countryCode': 'US'
                    }
                },
                'recipient': {
                    'address': {
                        'streetLines': [shipment['to_address']['street']],
                        'city': shipment['to_address']['city'],
                        'stateOrProvinceCode': shipment['to_address']['state'],
                        'postalCode': shipment['to_address']['zip'],
                        'countryCode': 'US'
                    }
                },
                'shipDateStamp': datetime.now().isoformat(),
                'pickupType': 'BUSINESS_SERVICE_CENTER',
                'rateRequestType': ['ACCOUNT', 'LIST'],
                'packageLineItems': [
                    {
                        'weight': {
                            'units': 'LB',
                            'value': item['weight']
                        },
                        'dimensions': {
                            'length': item['length'],
                            'width': item['width'],
                            'height': item['height'],
                            'units': 'IN'
                        }
                    } for item in shipment['items']
                ]
            }
        }
        
        response = requests.post(
            f'{self.base_url}/rate/v1/rates/quotes',
            json=payload,
            headers={'Authorization': f'Bearer {token}'},
            timeout=10
        )
        
        response.raise_for_status()
        
        return {
            'rates': [
                {
                    'service': r['serviceType'],
                    'cost': r['rateReplyDetails'][0]['totalNetFedExCharge'],
                    'carrier': 'fedex'
                }
                for r in response.json()['output']['rateReplyDetails']
            ]
        }

class ShippingService:
    """Service that uses adapters transparently."""
    
    def __init__(self):
        self.adapters = {
            'fedex': FedExAdapter(...),
            'ups': UPSAdapter(...),
            'usps': USPSAdapter(...),
        }
    
    def get_best_rates(self, shipment):
        """Get rates from all carriers and return cheapest."""
        
        all_rates = []
        
        for carrier_name, adapter in self.adapters.items():
            try:
                rates = adapter.get_rates(shipment)
                all_rates.extend(rates['rates'])
            
            except requests.Timeout:
                logger.warning(f"{carrier_name} timed out")
                # Continue with other carriers
            
            except Exception as e:
                logger.error(f"{carrier_name} error: {str(e)}")
                # Continue with other carriers
        
        # Return sorted by cost
        return sorted(all_rates, key=lambda r: float(r['cost']))
```

### Shipping Automation Results

**Before:**
- Manual rate shopping: 5 minutes per order
- Shipping label generation: Manual, error-prone
- Carrier API downtime impact: Everything breaks
- Tracking updates: Manual check, 2x/day

**After:**
- Automated rate shopping: <1 second per order
- Label generation: 100% automated
- Carrier downtime: System continues with fallback
- Tracking updates: Real-time via webhooks

**Impact:**
- **40% reduction in order processing time** (from 15 minutes to 9 minutes)
- **$2,000/month saved** on manual labor
- **99.5% uptime** despite carrier API outages

---

## Operational Lessons at Scale

### Logging That Actually Helps

```python
import json
import logging
from pythonjsonlogger import jsonlogger

# Use structured logging (JSON)
logger = logging.getLogger()
handler = logging.StreamHandler()
formatter = jsonlogger.JsonFormatter()
handler.setFormatter(formatter)
logger.addHandler(handler)

# Log with context
logger.info('order processed', extra={
    'order_id': 123,
    'customer_id': 456,
    'amount': 99.99,
    'processing_time_ms': 1250,
    'payment_method': 'stripe',
    'service_version': '2.5.1'
})

# Later, search for slow orders:
# logs | filter processing_time_ms > 5000
```

### Monitoring That Catches Failures Fast

```python
from prometheus_client import Counter, Histogram, Gauge

# Metrics
orders_total = Counter(
    'orders_total',
    'Total orders processed',
    ['status']
)

processing_time = Histogram(
    'order_processing_seconds',
    'Order processing time',
    buckets=(0.1, 0.5, 1.0, 2.0, 5.0, 10.0)
)

failed_shipments = Gauge(
    'failed_shipments',
    'Number of failed shipments pending retry'
)

# Usage
with processing_time.time():
    process_order(order_id)

orders_total.labels(status='succeeded').inc()
```

### Alerting Rules

```yaml
groups:
  - name: orders
    interval: 30s
    rules:
      - alert: HighOrderFailureRate
        expr: rate(orders_total{status="failed"}[5m]) > 0.05
        for: 5m
        annotations:
          summary: "Order failure rate >5%"
      
      - alert: SlowOrderProcessing
        expr: histogram_quantile(0.95, processing_time) > 5
        for: 10m
        annotations:
          summary: "95th percentile processing >5s"
      
      - alert: BackloggedShipments
        expr: failed_shipments > 100
        for: 1h
        annotations:
          summary: "100+ shipments failing"
```

---

## Key Metrics We Track

| Metric | Target | Actual |
|--------|--------|--------|
| Order Success Rate | >99.5% | 99.92% |
| Payment Failure Rate | <0.3% | 0.08% |
| Mean Processing Time | <2 seconds | 0.8 seconds |
| P99 Processing Time | <10 seconds | 4.2 seconds |
| Shipping Integration Uptime | >99% | 99.87% |
| API Availability | >99.9% | 99.97% |

---

## Implementation Checklist

- [ ] Set up database partitioning for 10M+ rows
- [ ] Implement select_related/prefetch_related throughout Django
- [ ] Integrate Stripe with idempotent keys
- [ ] Build webhook handler with retry logic
- [ ] Create shipping adapter pattern for multiple carriers
- [ ] Implement exponential backoff for external APIs
- [ ] Set up structured JSON logging
- [ ] Create Prometheus metrics and dashboards
- [ ] Configure alerting rules
- [ ] Load test to find breaking points
- [ ] Document runbooks for failure scenarios
- [ ] Monitor competitor pricing for SLA comparison

---

## Key Takeaways

Scaling SaaS from idea to 10K+ orders/month requires:

1. **Architecture decisions early** - Database partitioning, ORM optimization
2. **Idempotent operations** - Retries won't corrupt data
3. **Resilient integrations** - External APIs will fail
4. **Observable systems** - Structured logging, metrics, alerts
5. **Operational discipline** - Runbooks, on-call rotation, blameless postmortems

The product itself is only 20% of what makes a successful SaaS. The other 80% is infrastructure, reliability, and operational excellence.

---

## Related Resources

- [Django Performance Optimization](https://docs.djangoproject.com/en/stable/topics/db/optimization/)
- [Stripe Payment Integration Best Practices](https://stripe.com/docs/payments/handling-payment-state)
- [PostgreSQL Partitioning Guide](https://www.postgresql.org/docs/current/ddl-partitioning.html)
- [Prometheus Monitoring Guide](https://prometheus.io/docs/practices/instrumentation/)
- [GitHub: SaaS Patterns](https://github.com/bharatkse/saas-patterns-django)

---

**Are you building SaaS?** What scaling challenges have you faced? Share in the comments or connect on [LinkedIn](https://linkedin.com/in/bharat-kumar28).
