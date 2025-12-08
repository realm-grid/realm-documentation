# Subscription Lifecycle

## Subscription Model

RealmGrid uses **monthly subscriptions** with automatic renewal via Mollie.

### Subscription States

```
┌──────────────────────────────────────────────────────────────────────────┐
│                        SUBSCRIPTION LIFECYCLE                             │
└──────────────────────────────────────────────────────────────────────────┘

    ┌─────────┐     Payment      ┌─────────┐
    │ PENDING │ ───────────────► │ ACTIVE  │ ◄─────────────────┐
    └─────────┘    Successful    └─────────┘                   │
                                      │                        │
                                      │                   Renewal
                                      │                   Success
                          ┌───────────┼───────────┐           │
                          │           │           │           │
                          ▼           ▼           ▼           │
                   ┌──────────┐ ┌──────────┐ ┌──────────┐     │
                   │ PAUSED   │ │ PAST_DUE │ │CANCELING │     │
                   └──────────┘ └──────────┘ └──────────┘     │
                          │           │           │           │
                          │           │           │           │
              Resume      │    Payment│    Period │           │
                          │    Retry  │    Ends   │           │
                          │           │           │           │
                          ▼           ▼           ▼           │
                   ┌──────────────────────────────────────────┘
                   │                  │
                   │           ┌──────────┐
                   │           │ CANCELED │
                   │           └──────────┘
                   │                  │
                   │                  ▼
                   │           Server Stopped
                   │           Data Retained 30 days
                   │                  │
                   │                  ▼
                   │           Server Deleted
                   │           Data Purged
                   │
                   ▼
              Server Starts
```

### Status Definitions

| Status | Server State | Description |
|--------|--------------|-------------|
| `pending` | Not created | Awaiting payment confirmation |
| `active` | Running | Subscription is paid and active |
| `past_due` | Stopped | Payment failed, retry in progress |
| `paused` | Stopped | Manually paused by user |
| `canceling` | Running | Cancel scheduled, runs until period end |
| `canceled` | Stopped | Subscription ended |
| `trialing` | Running | Free trial period (if enabled) |

## Monthly Billing Cycle

```
Day 1: Subscription starts
        │
        ├── Initial payment charged
        ├── Server provisioned
        └── `currentPeriodEnd` set to Day 31
                │
Day 31: Renewal date
        │
        ├── Mollie creates recurring payment
        ├── webhook_mollie receives payment.paid
        ├── `currentPeriodEnd` extended to Day 61
        └── Server continues running
                │
Day 61: Next renewal...
```

## Cancellation Flow

### Who Can Cancel?

| Actor | Function/Endpoint | Immediate? | End of Period? |
|-------|-------------------|------------|----------------|
| User (web) | `subscription_cancel` | ❌ No | ✅ Yes |
| User (app) | `subscription_cancel` | ❌ No | ✅ Yes |
| Admin (panel) | `cancelSubscriptionAdmin` | ✅ Yes | ✅ Yes |
| System (fraud) | `cancelSubscriptionAdmin` | ✅ Yes | ❌ No |

### User Cancellation (End of Period)

When a user cancels, we **do NOT stop immediately**. They've paid for the current period.

```
User clicks "Cancel Subscription"
         │
         ▼
subscription_cancel function called
         │
         ├── Cancel Mollie subscription (stops future payments)
         ├── Set status = "canceling"
         ├── Calculate cancelAt = midnight AFTER currentPeriodEnd (user's timezone)
         ├── Store userTimezone for reference
         └── Server keeps running until cancelAt passes
                  │
                  ▼
         subscription_expire timer (runs hourly)
         Checks: status = "canceling" AND cancelAt <= now
```

### Admin Cancellation

Admins have additional options for cancellation:

```
Admin clicks "Cancel Subscription" in realm-admin
         │
         ▼
cancelSubscriptionAdmin() called via billing.server.ts
         │
         ├── Options:
         │   ├── immediately: true/false
         │   ├── reason: string (logged)
         │   ├── refundLastPayment: true/false
         │   └── suspendServer: true/false (stops server immediately)
         │
         ├── POST /admin/billing/subscriptions/{id}/cancel
         │
         ▼
realm-web API handles cancellation
         │
         ├── If immediately = true:
         │   ├── Set status = "canceled"
         │   ├── Set canceledAt = now
         │   ├── Stop server immediately
         │   └── Process refund if requested
         │
         └── If immediately = false:
             ├── Set status = "canceling"
             ├── Set cancelAt = currentPeriodEnd + 1 day midnight
             └── Server runs until period ends
```

### Timezone-Aware Cancellation

Server stops are calculated based on the user's timezone to be "polite":

```python
# subscription_cancel/__init__.py

def get_end_of_period_midnight(period_end: str, timezone: str) -> str:
    """
    Calculate when to stop the server:
    Midnight on the day AFTER the billing period ends
    
    Example: 
    - Period ends: 2025-01-08T12:34:56Z
    - User timezone: Europe/Amsterdam
    - cancelAt: 2025-01-09T00:00:00 Amsterdam time
    """
```

**Supported Timezones:**
- `Europe/Amsterdam` (default)
- `Europe/London`
- `Europe/Berlin`
- `Europe/Paris`
- `Europe/Brussels`
- `UTC`

### Expiration Processing

```
         Timer triggers for expired subscription:
         ├── Set status = "canceled"
         ├── Set canceledAt = now
         ├── Stop server (scale to 0)
         ├── Set dataRetentionEnd = now + 30 days
         └── Email user: "Your subscription has ended"
                  │
                  ▼
         subscription_cleanup timer (runs daily at 3 AM)
         Checks: status = "canceled" AND dataRetentionEnd <= now
                  │
                  ▼
         30 days later - cleanup triggers:
         ├── Delete K8s Deployment
         ├── Delete K8s Service  
         ├── Delete PVC (world data deleted!)
         ├── Delete backups from blob storage
         ├── Delete server document from Cosmos
         └── Mark subscription as purged = true
```

### Scheduled Jobs

| Function | Schedule | Purpose |
|----------|----------|---------|
| `subscription_expire` | Every hour (`0 0 * * * *`) | Stop servers when `cancelAt` passes |
| `subscription_cleanup` | Daily 3 AM (`0 0 3 * * *`) | Delete data after 30-day retention |

## Edge Cases & Error Handling

### Double Cancellation Attempt

**Scenario:** User tries to cancel an already canceling/canceled subscription.

```python
# subscription_cancel/__init__.py
if sub_doc.get("status") in ["canceling", "canceled"]:
    return error_response("Subscription already canceled or canceling", 400)
```

**Admin behavior:** Admin panel shows disabled cancel button for canceled subscriptions.

### Cancel During Paused State

**Scenario:** User wants to cancel while subscription is paused.

```
Subscription status = "paused"
         │
         ▼
User requests cancel
         │
         ├── Status → "canceling"
         ├── cancelAt set to next period end
         └── Server remains stopped (was already stopped from pause)
                  │
                  ▼
         Period ends → status = "canceled"
```

**Note:** Paused subscriptions still have `currentPeriodEnd` so cancellation works normally.

### Cancel During Past Due State

**Scenario:** User cancels while payment is failing.

```
Subscription status = "past_due"
         │
         ▼
User requests cancel
         │
         ├── Cancel Mollie subscription (stops retry attempts)
         ├── Status → "canceled" (immediate, no period remaining)
         └── Server stays stopped
```

**Rationale:** If payment already failed, user hasn't paid for current period - cancel immediately.

### Cancel Before First Payment Completes

**Scenario:** User cancels during `pending` state.

```
Subscription status = "pending"
         │
         ▼
User requests cancel
         │
         └── Return error: "Cannot cancel pending subscription"
```

**Admin option:** Admin can force-cancel pending subscriptions.

### Mollie Subscription Not Found

**Scenario:** Mollie subscription was already canceled or doesn't exist.

```python
# subscription_cancel/__init__.py
if not sub_doc.get("mollieSubscriptionId"):
    return error_response("No Mollie subscription found", 400)
```

**Admin override:** Admin can cancel with `skipMollie: true` to update only local state.

### Timer Miss (subscription_expire delayed)

**Scenario:** Azure Functions timer is delayed, subscription should have expired hours ago.

```python
# subscription_expire/__init__.py
if timer.past_due:
    logger.warning("Timer is past due, running anyway")
```

**Behavior:** Timer still processes all expired subscriptions when it runs. User may get slightly more time.

### Server Stop Failure

**Scenario:** K8s cluster unreachable when trying to stop server.

```python
# subscription_expire/__init__.py
try:
    await stop_server(server_id)
except Exception as e:
    logger.error(f"Failed to stop server {server_id}: {e}")
    # Continue anyway - subscription is still canceled
```

**Recovery:** `subscription_cleanup` will delete K8s resources after 30 days regardless. Orphaned pods will be cleaned up.

### Immediate Cancel with Refund

**Scenario:** Admin processes refund request with immediate cancellation.

```typescript
// realm-admin/app/lib/billing.server.ts
await cancelSubscriptionAdmin(subscriptionId, {
  immediately: true,
  reason: "Refund requested",
  refundLastPayment: true,
  suspendServer: true,
});
```

**Mollie handling:** 
1. Cancel subscription first
2. Fetch last payment
3. Create refund via Mollie API
4. Log refund in transactions

### Cancel During Tier Change

**Scenario:** User has pending tier upgrade and cancels.

```
Subscription with pendingTier = "medium"
         │
         ▼
User requests cancel
         │
         ├── Clear pendingTier (won't upgrade)
         ├── Status → "canceling"
         └── Keep current tier until period ends
```

### Multiple Subscriptions Same User

**Scenario:** User has multiple game servers/subscriptions.

```
User has:
  - sub_1 (Minecraft, active)
  - sub_2 (Valheim, active)
         │
         ▼
User cancels sub_1
         │
         ├── Only sub_1 affected
         ├── sub_2 continues normally
         └── Each subscription is independent
```

### Auto-Cancel After 14 Days Past Due

**Scenario:** Payment keeps failing, no user action.

```python
# subscription_expire/__init__.py
# Check for past_due subscriptions older than 14 days
past_due_cutoff = datetime(now.year, now.month, now.day - 14).isoformat() + "Z"

past_due_query = """
    SELECT * FROM c 
    WHERE c.status = 'past_due' 
    AND c.updatedAt <= @cutoff
"""
```

**Behavior:**
1. After 14 days of `past_due` status
2. Subscription auto-cancels
3. `cancelReason = "Payment failed - auto-canceled after 14 days"`
4. User notified via email

## Refunds & Chargebacks

### What is "Last Payment"?

When admin requests `refundLastPayment: true`, the system identifies the payment:

```typescript
// realm-admin/app/lib/billing.server.ts

// "Last payment" = the most recent SUCCESSFUL payment
// Stored in subscription.lastPaymentId and subscription.lastPaymentAt

// NOT considered as "last payment":
// - Failed payments
// - Pending payments  
// - Already refunded payments
// - Chargebacked payments
```

**Subscription fields:**

| Field | Description |
|-------|-------------|
| `lastPaymentId` | Mollie payment ID of most recent successful charge |
| `lastPaymentAt` | Timestamp of last successful payment |
| `initialPaymentId` | The very first payment (for subscription creation) |

### Refund Flow

```
Admin clicks "Refund Last Payment"
         │
         ▼
Check: lastPaymentId exists?
         │
    ┌────┴────┐
    No        Yes
    │         │
    ▼         ▼
 Error     Check: Already refunded?
 "No              │
 payment    ┌─────┴─────┐
 found"     Yes         No
            │           │
            ▼           ▼
         Error      Create Mollie refund
         "Already   via POST /v2/payments/{id}/refunds
         refunded"       │
                         ▼
                  Update payment status = "refunded"
                  Create refund transaction in Cosmos
                         │
                         ▼
                  If immediate cancel: stop server
                  If end-of-period: continue until cancelAt
```

### Chargeback Scenarios

A **chargeback** occurs when a customer disputes a charge with their bank.

#### Chargeback on Active Subscription

**Chargebacks trigger cancellation at the end of the billing period.** The user has disputed the charge, so we schedule cancellation but let them use what they've already paid for.

```
Mollie webhook: chargeback opened
         │
         ▼
webhook_mollie/handle_chargeback()
         │
         ├── Set subscription status = "canceling"
         ├── Set cancelAt = currentPeriodEnd
         ├── Set cancelReason = "Chargeback received"
         ├── Flag user (hasChargeback = true)
         └── Create chargeback record for admin
                  │
                  ▼
         Server continues until period end
         (subscription_expire timer handles stop)
                  │
                  ▼
         Admin reviews chargeback
                  │
         ┌───────┴───────┐
         │               │
    Submit evidence  Accept loss
    to dispute       (no action)
         │               │
         ▼               ▼
    Under review    Status = "lost"
         │          Amount deducted
         │
    ┌────┴────┐
    │         │
   Won       Lost
    │         │
    ▼         ▼
 Consider   Already scheduled
 refund?    for cancellation
```

**Rationale:** Even with a chargeback, the user may have legitimately paid for previous months. We honor the current period they paid for, then stop service. This is fair and reduces disputes.

#### Chargeback on Canceled Subscription

```
Subscription already canceled
         │
         ▼
Chargeback received for old payment
         │
         ├── Flag payment as "charged_back"
         ├── Record in chargeback log
         └── Affects revenue metrics only
                  │
                  ▼
         Admin decides: fight or accept
```

#### Chargeback After Refund (Double Dip)

```
User requests refund → Refund processed
         │
         ▼
User ALSO opens chargeback (fraud attempt)
         │
         ├── System detects: payment already refunded
         ├── Auto-submit refund receipt as evidence
         └── Usually wins dispute automatically
```

### Chargeback Status Flow

```
┌─────────────────┐     Evidence      ┌──────────────────┐
│ needs_response  │ ───────────────► │   under_review   │
└─────────────────┘    Submitted     └──────────────────┘
        │                                     │
        │ Accept Loss                   Bank decides
        │                                     │
        ▼                            ┌───────┴───────┐
┌─────────────────┐                  │               │
│      lost       │ ◄────────────────┤               │
└─────────────────┘    Dispute       │               │
                       Lost          │               │
                                     ▼               ▼
                              ┌───────────┐  ┌───────────┐
                              │    won    │  │   lost    │
                              └───────────┘  └───────────┘
```

### Chargeback Evidence Types

| Type | Description | When to Use |
|------|-------------|-------------|
| `receipt` | Payment confirmation | Always |
| `refund_policy` | Terms of service | Subscription disputes |
| `customer_communication` | Emails, support tickets | Service complaints |
| `service_documentation` | Server logs, uptime proof | "Service not received" |
| `other` | Any supporting docs | Edge cases |

### Refund vs Chargeback Decision

| Situation | Action | Reason |
|-----------|--------|--------|
| User asks nicely for refund | Refund | Good customer relations |
| Within 14-day cooling-off | Refund | EU law requirement |
| User files chargeback first | Fight it | Don't reward bad behavior |
| User threatens chargeback | Refund quickly | Avoid fees + hassle |
| Obvious fraud pattern | Cancel + ban | Protect business |
| Legitimate service issue | Refund | It's the right thing |

### Admin Chargeback Page

Located at: `realm-admin/app/routes/billing.chargebacks.tsx`

Features:
- View all open chargebacks
- Submit evidence documents
- Accept loss (give up dispute)
- View refund requests
- Approve/reject refund requests
- Track win/loss rates

## API Alignment Matrix

| Scenario | Function App | Admin Panel | Web API |
|----------|-------------|-------------|---------|
| User cancel (end of period) | `subscription_cancel` | N/A | `/api/subscriptions/:id/cancel` |
| Admin cancel (end of period) | `subscription_cancel` | `cancelSubscriptionAdmin` | `/admin/billing/subscriptions/:id/cancel` |
| Admin cancel (immediate) | `subscription_cancel` | `cancelSubscriptionAdmin` | `/admin/billing/subscriptions/:id/cancel` |
| Timer expiration | `subscription_expire` | N/A | N/A |
| Cleanup after 30 days | `subscription_cleanup` | N/A | N/A |
| Pause subscription | N/A | `pauseSubscription` | `/admin/billing/subscriptions/:id/pause` |
| Resume subscription | N/A | `resumeSubscription` | `/admin/billing/subscriptions/:id/resume` |
| Process refund | N/A | `processRefund` | `/admin/billing/refunds/:id/process` |
| Handle chargeback | `webhook_mollie` | `submitChargebackEvidence` | `/admin/billing/chargebacks/:id/evidence` |

### Cancellation API

**Request:**
```http
POST /api/subscriptions/{subscriptionId}/cancel
Content-Type: application/json

{
    "immediate": false,  // false = end of period, true = now
    "reason": "Too expensive"  // Optional feedback
}
```

**Response:**
```json
{
    "success": true,
    "data": {
        "subscriptionId": "sub_tr_xyz123",
        "status": "canceling",
        "cancelAt": "2025-01-08T00:00:00Z",
        "serverStopsAt": "2025-01-08T00:00:00Z"
    },
    "message": "Subscription will be canceled on 2025-01-08"
}
```

### Immediate Cancellation

Only used for:
- Refund requests within cooling-off period
- Admin-initiated cancellations
- Fraud/abuse cases

```http
POST /api/subscriptions/{subscriptionId}/cancel
{
    "immediate": true,
    "reason": "Refund requested"
}
```

This immediately:
1. Sets status to `canceled`
2. Stops the server
3. Triggers prorated refund (if applicable)

## Reactivation

A canceled subscription can be reactivated within the data retention period:

```
User on "canceled" status
         │
         ▼
User clicks "Reactivate"
         │
         ▼
New payment created via Mollie
         │
         ▼
Payment successful
         │
         ├── Status → "active"
         ├── Server starts (existing data)
         └── New billing period begins
```

## Payment Failure Handling

```
Mollie payment fails
         │
         ▼
webhook_mollie receives payment.failed
         │
         ├── Increment failedPaymentAttempts
         ├── Log failure reason
         └── Mollie will retry automatically
                  │
         ┌───────┴───────┐
         │               │
    Retry succeeds   3 retries fail
         │               │
         ▼               ▼
    Status stays    Status → "past_due"
    "active"        Server stopped
                         │
                         ▼
                    Email sent to user
                    "Update payment method"
                         │
         ┌───────────────┼───────────────┐
         │               │               │
    User updates    No action       14 days pass
    payment              │               │
         │               │               │
         ▼               │               ▼
    New payment          │          Status → "canceled"
    succeeds             │          Begin data retention
         │               │
         ▼               ▼
    Status → "active"   Wait...
    Server starts
```

## Subscription Document (Cosmos DB)

```json
{
    "id": "sub_tr_xyz123",
    "userId": "auth0|abc123",
    "serverId": "srv_a1b2c3d4",
    "status": "active",
    "tier": "medium",
    "gameType": "minecraft",
    "maxPlayers": 25,
    
    "amount": 999,
    "currency": "EUR",
    "interval": "monthly",
    
    "provider": "mollie",
    "providerSubscriptionId": "sub_abc123",
    "providerCustomerId": "cst_xyz789",
    
    "currentPeriodStart": "2024-12-08T00:00:00Z",
    "currentPeriodEnd": "2025-01-08T00:00:00Z",
    
    "cancelAt": null,
    "canceledAt": null,
    "cancelReason": null,
    
    "failedPaymentAttempts": 0,
    "lastPaymentAt": "2024-12-08T12:34:56Z",
    
    "createdAt": "2024-12-08T12:34:56Z",
    "updatedAt": "2024-12-08T12:34:56Z"
}
```

## Tier Changes

Users can upgrade or downgrade their tier:

### Upgrade (immediate)
```
User on "small" tier
         │
         ▼
Requests upgrade to "medium"
         │
         ▼
Prorated charge calculated:
  Days remaining × (medium_price - small_price) / 30
         │
         ▼
Mollie payment created
         │
         ▼
Payment successful
         │
         ├── Update subscription tier
         ├── Patch K8s deployment with new resources
         └── Pod restarts with more CPU/RAM
```

### Downgrade (next period)
```
User on "heavy" tier
         │
         ▼
Requests downgrade to "medium"
         │
         ▼
Set pendingTier = "medium"
         │
         ▼
At next renewal:
         │
         ├── Charge medium price
         ├── Update tier to "medium"
         ├── Patch K8s deployment
         └── Pod restarts with less resources
```
