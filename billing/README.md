# Billing & Payments

## Payment Provider: Mollie

We use [Mollie](https://www.mollie.com) for payments, supporting European payment methods.

### Supported Payment Methods

| Method | Countries | Description |
|--------|-----------|-------------|
| iDEAL | Netherlands | Bank transfer, instant |
| Bancontact | Belgium | Debit card |
| SOFORT | Germany, Austria | Bank transfer |
| Credit Card | All | Visa, Mastercard, Amex |
| PayPal | All | PayPal account |
| Klarna | Various | Pay later |

## Pricing Structure

### Tier Pricing (EUR)

| Tier | Monthly | Quarterly | Yearly |
|------|---------|-----------|--------|
| **Small** | €4.99 | €13.47 (10% off) | €47.90 (20% off) |
| **Medium** | €9.99 | €26.97 (10% off) | €95.88 (20% off) |
| **Heavy** | €24.99 | €67.47 (10% off) | €239.88 (20% off) |

### Billing Intervals

| Interval | Renewal | Discount |
|----------|---------|----------|
| Monthly | Every 30 days | None |
| Quarterly | Every 90 days | 10% |
| Yearly | Every 365 days | 20% |

## Checkout Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           CHECKOUT FLOW                                      │
└─────────────────────────────────────────────────────────────────────────────┘

Customer selects:
├── Game: Minecraft
├── Tier: Medium
└── Interval: Monthly
         │
         ▼
POST /api/checkout/create
{
    "userId": "auth0|123",
    "email": "user@example.com",
    "gameType": "minecraft",
    "tier": "medium",
    "interval": "monthly",
    "serverName": "My Server"
}
         │
         ▼
checkout_create function:
├── Validate tier + game
├── Calculate price (€9.99)
├── Create Mollie payment
│   {
│       "amount": {"value": "9.99", "currency": "EUR"},
│       "description": "RealmGrid Medium - minecraft",
│       "redirectUrl": "https://realmgrid.com/checkout/success",
│       "webhookUrl": "https://api.realmgrid.com/webhook/mollie",
│       "metadata": {
│           "userId": "auth0|123",
│           "gameType": "minecraft",
│           "tier": "medium",
│           "serverName": "My Server"
│       }
│   }
└── Return checkout URL
         │
         ▼
Customer redirected to Mollie
├── Selects payment method (iDEAL, card, etc.)
├── Completes payment
└── Redirected to success page
         │
         ▼
Mollie sends webhook
POST /webhook/mollie
{
    "id": "tr_xyz123"
}
         │
         ▼
webhook_mollie function:
├── Fetch payment details from Mollie
├── Verify payment status = "paid"
├── Create subscription in Cosmos
├── Create server in Cosmos
├── Call server_provision
└── Return 200 OK
```

## Webhook Events

### payment.paid
Received when a payment is successful (initial or renewal).

```json
{
    "id": "tr_xyz123",
    "status": "paid",
    "amount": {"value": "9.99", "currency": "EUR"},
    "metadata": {
        "userId": "auth0|123",
        "tier": "medium",
        "gameType": "minecraft"
    }
}
```

**Actions:**
1. Create/extend subscription
2. Start server (if new or was stopped)
3. Send confirmation email

### payment.failed
Received when a payment fails.

```json
{
    "id": "tr_xyz123",
    "status": "failed",
    "failureReason": "insufficient_funds"
}
```

**Actions:**
1. Increment failure count
2. If 3+ failures: set status = "past_due", stop server
3. Send notification email

### payment.expired
Payment was not completed within time limit.

**Actions:**
1. Mark checkout as expired
2. No subscription created

## Subscription Renewal

Mollie handles automatic renewals for recurring subscriptions:

```
Day 30: Mollie creates new payment
         │
         ▼
Payment succeeds
         │
         ▼
webhook_mollie receives payment.paid
├── subscriptionId in metadata
├── Extend currentPeriodEnd by 30 days
└── Log renewal
         │
         ▼
Customer charged, server continues
```

## Refunds

### Cooling-off Period (14 days)
EU consumers have a 14-day right to cancel. We honor this:

```python
# Check if within cooling-off period
days_since_purchase = (now - subscription.createdAt).days
if days_since_purchase <= 14:
    # Full refund
    mollie.refunds.create(payment_id, amount=full_amount)
    # Immediate cancellation
    cancel_subscription(subscription_id, immediate=True)
```

### Prorated Refunds
After 14 days, refunds are prorated:

```python
days_used = (now - currentPeriodStart).days
days_total = 30
days_remaining = days_total - days_used
refund_amount = (days_remaining / days_total) * monthly_price
```

## Invoice Generation

After each successful payment:

```json
{
    "id": "inv_abc123",
    "subscriptionId": "sub_tr_xyz123",
    "userId": "auth0|123",
    
    "status": "paid",
    "amount": 999,
    "currency": "EUR",
    "tax": 174,  // 21% BTW
    
    "lineItems": [
        {
            "description": "RealmGrid Medium - Minecraft",
            "quantity": 1,
            "unitPrice": 826,
            "tax": 174,
            "total": 999
        }
    ],
    
    "paymentId": "tr_xyz123",
    "paymentMethod": "ideal",
    
    "createdAt": "2024-12-08T12:34:56Z",
    "paidAt": "2024-12-08T12:35:00Z"
}
```

## Currency

All prices are in **EUR** (Euros) as we primarily serve the European market.

For display purposes:
- Always show €X.XX format
- Use comma for decimals in NL/DE/FR: €9,99
- Use period for decimals in UK/US: €9.99 (if supporting)
