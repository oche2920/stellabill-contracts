# Subscription Lifecycle Events

This document describes the event schemas emitted by the `subscription_vault` contract for indexing and monitoring subscription lifecycle actions.

## Event Overview

All events are emitted using Soroban's native event system and can be consumed by indexers, backends, and monitoring tools. Events are emitted exactly once per action with minimal redundancy.

## Event Schemas

### SubscriptionCreatedEvent

**Topic:** `sub_new`

Emitted when a new subscription is created.

**Fields:**
- `subscription_id` (u32): Unique identifier for the subscription
- `subscriber` (Address): Address of the subscriber
- `merchant` (Address): Address of the merchant receiving payments
- `amount` (i128): Payment amount per billing interval (in token base units)
- `interval_seconds` (u64): Billing interval in seconds

**Indexing Strategy:**
- Index by `subscription_id` for lookup
- Index by `subscriber` and `merchant` for filtering user/merchant subscriptions
- Track creation timestamp from ledger metadata

**Example Use Cases:**
- Build subscriber dashboard showing all subscriptions
- Merchant analytics on new subscriptions
- Monitoring subscription creation rate

---

### FundsDepositedEvent

**Topic:** `deposit`

Emitted when a subscriber deposits funds to their subscription vault.

**Fields:**
- `subscription_id` (u32): Subscription receiving the deposit
- `subscriber` (Address): Address making the deposit
- `amount` (i128): Amount deposited (in token base units)
- `new_balance` (i128): Total prepaid balance after deposit

**Indexing Strategy:**
- Index by `subscription_id` to track balance history
- Aggregate deposits per subscriber for analytics
- Monitor `new_balance` for low-balance alerts

**Example Use Cases:**
- Display deposit history in subscriber UI
- Alert subscribers when balance is low
- Track total value locked in the contract

---

### SubscriptionChargedEvent

**Topic:** `charged`

Emitted when a subscription is charged for a billing interval.

**Fields:**
- `subscription_id` (u32): Subscription that was charged
- `merchant` (Address): Merchant receiving the payment
- `token` (Address): Settlement token used for this charge
- `amount` (i128): Gross amount charged (in token base units)
- `merchant_amount` (i128): Net amount credited to merchant (after protocol fee)
- `fee_amount` (i128): Protocol fee amount (0 if no fee configured)
- `remaining_balance` (i128): Subscriber's prepaid balance remaining after charge
- `lifetime_charged` (i128): Cumulative total charged over subscription lifetime

**Indexing Strategy:**
- Index by `subscription_id` for payment history
- Index by `merchant` to track merchant revenue
- Use `amount` (gross), `fee_amount`, `merchant_amount` for fee reconciliation
- Monitor `remaining_balance` for insufficient balance warnings

**Example Use Cases:**
- Generate merchant revenue reports (use `merchant_amount` for net revenue)
- Track protocol fee collection (use `fee_amount`)
- Track subscription payment history
- Trigger notifications when balance is insufficient for next charge

---

### SubscriptionPausedEvent

**Topic:** `paused`

Emitted when a subscription is paused (no charges until resumed).

**Fields:**
- `subscription_id` (u32): Subscription that was paused
- `authorizer` (Address): Address that authorized the pause (subscriber or merchant)

**Indexing Strategy:**
- Index by `subscription_id` to track status changes
- Track pause duration by comparing with resume events

**Example Use Cases:**
- Display paused status in UI
- Analytics on pause frequency and duration
- Notify relevant parties of status change

---

### SubscriptionResumedEvent

**Topic:** `resumed`

Emitted when a paused subscription is resumed.

**Fields:**
- `subscription_id` (u32): Subscription that was resumed
- `authorizer` (Address): Address that authorized the resume (subscriber or merchant)

**Indexing Strategy:**
- Index by `subscription_id` to track status changes
- Calculate pause duration by comparing with pause events

**Example Use Cases:**
- Update subscription status in UI
- Track subscription lifecycle metrics
- Resume billing operations

---

### SubscriptionCancelledEvent

**Topic:** `cancelled`

Emitted when a subscription is cancelled by subscriber or merchant.

**Fields:**
- `subscription_id` (u32): Subscription that was cancelled
- `authorizer` (Address): Address that authorized the cancellation
- `refund_amount` (i128): Remaining prepaid balance available for refund

**Indexing Strategy:**
- Index by `subscription_id` for final status
- Track cancellation rate by subscriber/merchant
- Monitor `refund_amount` for refund processing

**Example Use Cases:**
- Process refunds to subscribers
- Calculate churn rate and cancellation analytics
- Archive cancelled subscriptions

---

### MerchantWithdrawalEvent

**Topic:** `withdrawn`

Emitted when a merchant withdraws accumulated funds.

**Fields:**
- `merchant` (Address): Merchant withdrawing funds
- `token` (Address): Token bucket debited for the withdrawal
- `amount` (i128): Amount withdrawn (in token base units)
- `remaining_balance` (i128): Merchant's accumulated balance remaining after withdrawal

**Indexing Strategy:**
- Index by `merchant` to track withdrawal history
- Aggregate total withdrawals per merchant
- Monitor withdrawal frequency

**Example Use Cases:**
- Display merchant withdrawal history
- Track merchant payout schedules
- Reconcile merchant balances

---

### ContractInitializedEvent

**Topic:** `initialized`

Emitted when the contract is initialized.

**Fields:**
- `token` (Address): Primary settlement token address
- `admin` (Address): Initial admin address
- `min_topup` (i128): Minimum deposit threshold
- `grace_period` (u64): Grace period in seconds
- `timestamp` (u64): Ledger timestamp when initialized

---

### PlanTemplateCreatedEvent

**Topic:** `plan_template_created`

Emitted when a merchant creates a new plan template.

**Fields:**
- `plan_template_id` (u32): Newly allocated plan template ID
- `merchant` (Address): Merchant that created the plan
- `token` (Address): Settlement token for subscriptions from this plan
- `amount` (i128): Recurring charge amount per interval
- `interval_seconds` (u64): Billing interval in seconds
- `usage_enabled` (bool): Whether usage-based charging is enabled
- `lifetime_cap` (Option<i128>): Optional lifetime cap
- `timestamp` (u64): Ledger timestamp when created

---

### OneOffChargedEvent

**Topic:** `oneoff_ch`

Emitted when a merchant applies a one-off charge.

**Fields:**
- `subscription_id` (u32): Subscription that was charged
- `merchant` (Address): Merchant that applied the charge
- `token` (Address): Settlement token used
- `amount` (i128): Amount charged
- `remaining_balance` (i128): Subscriber's prepaid balance after charge
- `timestamp` (u64): Ledger timestamp

---

### SubscriberWithdrawalEvent

**Topic:** `sub_withdrawn`

Emitted when a subscriber withdraws their remaining prepaid balance after cancellation.

**Fields:**
- `subscription_id` (u32): Subscription being withdrawn from
- `subscriber` (Address): Subscriber receiving the funds
- `token` (Address): Settlement token returned
- `amount` (i128): Amount withdrawn
- `timestamp` (u64): Ledger timestamp

---

### PartialRefundEvent

**Topic:** `partial_refund`

Emitted when an admin processes a partial refund to a subscriber.

**Fields:**
- `subscription_id` (u32): Subscription receiving the refund
- `subscriber` (Address): Subscriber receiving the refund
- `token` (Address): Settlement token returned
- `amount` (i128): Amount refunded
- `remaining_balance` (i128): Subscriber's prepaid balance after refund
- `timestamp` (u64): Ledger timestamp

---

### AdminRotationEvent

**Topic:** `admin_rotation`

Emitted when the contract admin is rotated to a new address. Only the current admin can perform rotation.

**Fields:**
- `current_admin` (Address): The admin address that initiated the rotation (now revoked)
- `new_admin` (Address): The new admin address that received privileges
- `timestamp` (u64): Ledger timestamp when the rotation occurred

**Indexing Strategy:**
- Index by `current_admin` and `new_admin` for audit trails
- Track timestamp for rotation history
- Maintain current admin state for authorization checks

**Example Use Cases:**
- Build admin rotation audit log
- Alert on admin changes for security monitoring
- Update off-chain systems with current admin address

---

### RecoveryEvent

**Topic:** `recovery`

Emitted when the admin recovers stranded funds from the contract (e.g., accidental transfers, deprecated flows).

**Fields:**
- `admin` (Address): The admin who authorized the recovery
- `recipient` (Address): The destination address receiving the recovered funds
- `amount` (i128): Amount recovered (in token base units)
- `reason` (RecoveryReason): Enum—`AccidentalTransfer` (0), `DeprecatedFlow` (1), `UnreachableSubscriber` (2)
- `timestamp` (u64): Ledger timestamp when recovery was executed

**Indexing Strategy:**
- Index by `admin` to track recovery actions per admin
- Index by `recipient` for recipient-side history
- Aggregate amounts by reason for analytics

**Example Use Cases:**
- Audit trail for fund recoveries
- Monitor admin recovery activity
- Analytics on recovery reasons and amounts

---

## General Indexing Recommendations

### Event Consumption

1. **Subscribe to contract events** using Stellar RPC or Horizon API
2. **Filter by contract address** to get only subscription vault events
3. **Parse event topics** to identify event type
4. **Decode event data** using the schemas above

### Storage Strategy

- Store events in time-series database for historical analysis
- Maintain current state in relational database for fast queries
- Index by `subscription_id`, `subscriber`, and `merchant` addresses

### Error Handling

- Events are emitted after state changes succeed
- If a transaction fails, no event is emitted
- Monitor transaction status alongside events

### Privacy Considerations

- Events contain only addresses and amounts (no personal data)
- Addresses are pseudonymous but publicly visible on-chain
- Off-chain systems should implement additional privacy controls

---

## Example Event Flow

**Typical subscription lifecycle:**

1. `SubscriptionCreatedEvent` - Subscriber creates subscription
2. `FundsDepositedEvent` - Subscriber deposits initial funds
3. `SubscriptionChargedEvent` (recurring) - Billing engine charges subscription
4. `FundsDepositedEvent` (as needed) - Subscriber tops up balance
5. `SubscriptionPausedEvent` (optional) - Subscriber pauses temporarily
6. `SubscriptionResumedEvent` (optional) - Subscriber resumes
7. `SubscriptionCancelledEvent` - Subscriber or merchant cancels
8. `MerchantWithdrawalEvent` (periodic) - Merchant withdraws earnings

---

## Integration Examples

### Indexer Pseudocode

```rust
// Listen for events
for event in contract_events {
    match event.topic {
        "sub_new" => {
            let data: SubscriptionCreatedEvent = decode(event.data);
            db.insert_subscription(data);
        }
        "deposit" => {
            let data: FundsDepositedEvent = decode(event.data);
            db.update_balance(data.subscription_id, data.new_balance);
        }
        "charged" => {
            let data: SubscriptionChargedEvent = decode(event.data);
            db.record_payment(data);
        }
        "admin_rotation" => {
            let (current_admin, new_admin, timestamp) = decode(event.data);
            db.record_admin_rotation(current_admin, new_admin, timestamp);
        }
        "recovery" => {
            let data: RecoveryEvent = decode(event.data);
            db.record_recovery(data);
        }
        // ... handle other events
    }
}
```

### Backend Monitoring

```javascript
// Monitor for low balance
events.on('charged', (event) => {
  if (event.remaining_balance < event.amount * 2) {
    notifySubscriber(event.subscription_id, 'Low balance warning');
  }
});

// Track merchant revenue
events.on('charged', (event) => {
  analytics.recordRevenue(event.merchant, event.amount);
});
```

---

## Version History

- **v1.0** (2026-02-20): Initial event schema definitions for all lifecycle actions
- **v1.1** (2026-02-23): Added AdminRotationEvent and RecoveryEvent for indexers
- **v1.2** (2026-04-23): Complete event schema overhaul for indexer completeness
  - SubscriptionCreatedEvent: added `token` field
  - SubscriptionChargedEvent: added `token`, `merchant_amount`, `fee_amount`, `remaining_balance` fields for gross/fee/net split
  - UsageStatementEvent: added `remaining_balance` field
  - OneOffChargedEvent: added `token`, `remaining_balance`, `timestamp` fields
  - SubscriberWithdrawalEvent: added `token`, `timestamp` fields
  - PartialRefundEvent: added `token`, `remaining_balance` fields
  - MerchantWithdrawalEvent: now emits full struct (was raw amount)
  - Added ContractInitializedEvent struct
  - Added PlanTemplateCreatedEvent for plan creation tracking
  - Fixed duplicate struct definitions in types.rs
