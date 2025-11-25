# Complete Flow: Meter Usage Ingestion → Wallet Balance Alerts

**Date**: 2025-11-25
**Analysis Scope**: End-to-end flow from meter event ingestion to wallet balance alert generation

## Overview

Wallet balance alerts are **indirectly tied** to meter usage ingestion through a multi-stage data processing pipeline. This document traces the complete architecture from event ingestion to alert webhook publishing.

---

## 🔄 The Complete Alert Pipeline

### Stage 1: Meter Event Ingestion

**Entry Point**: `/internal/api/v1/events.go:49-73`

```
User/System → POST /events → IngestEvent() → EventService.CreateEvent()
```

**What Happens**:
1. Event is validated and published to **Kafka topic** for async processing
2. Event contains: `event_name`, `external_customer_id`, `properties`, `timestamp`
3. Returns immediately with `202 Accepted` (async processing)

**Location**: `/internal/service/event.go:61-78`

```go
func (s *eventService) CreateEvent(ctx context.Context, createEventRequest *dto.IngestEventRequest) error {
    event := createEventRequest.ToEvent(ctx)

    if err := s.publisher.Publish(ctx, event); err != nil {
        // Log error but don't fail request
        s.logger.Error("failed to publish event", "error", err)
    }

    return nil
}
```

---

### Stage 2: Feature Usage Processing (Kafka Consumer)

**Processing Service**: `FeatureUsageTrackingService`
**Location**: `/internal/service/feature_usage_tracking.go:173-200`

**Message Flow**:
```
Kafka Topic: feature_usage_tracking
      ↓
Consumer (with rate limiting)
      ↓
processMessage() handler
      ↓
Maps events to subscriptions & features
      ↓
Inserts into ClickHouse: feature_usage table
```

**Key Processing Steps** (line 315):
```go
s.featureUsageRepo.BulkInsertProcessedEvents(ctx, featureUsage)
```

**ClickHouse Table**: `feature_usage`
- Contains: `subscription_id`, `feature_id`, `meter_id`, `qty_total`, `customer_id`
- Used for aggregating usage charges

**Schema**: `/migrations/clickhouse/000003_create_feature_usage_table.up.sql`
```sql
CREATE TABLE feature_usage (
    id String,
    subscription_id String,
    feature_id String,
    meter_id String,
    qty_total Decimal(20, 9),
    timestamp DateTime64(3),
    ...
) ENGINE = ReplacingMergeTree()
```

---

### Stage 3: Real-Time Balance Calculation (On-Demand)

**Trigger**: When balance is queried OR during cron alert checks

**Flow in Wallet Service** (`/internal/service/wallet.go`):

```
GetWalletBalanceV2(walletID)
    ↓
Step 1: Get Wallet Details (line 1727)
    wallet = WalletRepo.GetWalletByID(walletID)
    ↓
Step 2: Check if wallet allows usage charges (line 1745)
    shouldIncludeUsage = wallet.Config.AllowedPriceTypes contains USAGE or ALL
    ↓
Step 3: Get Customer's Active Subscriptions (line 1753)
    subscriptions = SubRepo.ListByCustomerID(w.CustomerID)
    Filter by currency matching wallet
    ↓
Step 4: For Each Subscription:
    ↓
    4a. Get Feature Usage (line 1782-1786)
        usage = subscriptionService.GetFeatureUsageBySubscription()
           ↓
           Queries ClickHouse feature_usage table
           Aggregates by subscription_id, feature_id
    ↓
    4b. Calculate Usage Charges (line 1792)
        usageCharges = billingService.CalculateUsageCharges(sub, usage, ...)
           ↓
           Matches usage to subscription line items
           Applies pricing tiers
           Returns totalUsageCost
    ↓
    4c. Accumulate Charges (line 1802)
        totalPendingCharges += usageTotal
    ↓
Step 5: Calculate Real-Time Balance (line 1807)
    realTimeBalance = wallet.Balance - totalPendingCharges
    realTimeCreditBalance = realTimeBalance / conversionRate
    ↓
Return WalletBalanceResponse {
    RealTimeBalance: realTimeBalance,
    RealTimeCreditBalance: realTimeCreditBalance,
    CurrentPeriodUsage: totalPendingCharges
}
```

**Code Reference**: `/internal/service/wallet.go:1719-1826`

---

### Stage 4: Alert Evaluation (Cron Job)

**Cron Handler**: `/internal/api/cron/wallet.go:118-411`

**Schedule**: Periodic (configured by system admin)

**CheckAlerts() Flow**:

```
For each tenant:
    For each environment:
        ↓
        Step 1: Get Active Wallets with Alerts Enabled (line 162-166)
            wallets = walletService.GetWallets({
                Status: ACTIVE,
                AlertEnabled: true
            })
        ↓
        Step 2: Get Features with Alert Settings (line 184-199)
            features = featureService.GetFeatures()
            Filter: features with alert_settings.enabled = true
        ↓
        Step 3: For Each Wallet (line 209):
            ↓
            3a. Get Real-Time Balance (line 235)
                balance = walletService.GetWalletBalanceV2(wallet.ID)
                    ↓
                    [This triggers the entire flow from Stage 3 above!]
                    ↓ ↓ ↓
                    Queries subscriptions → Queries feature_usage → Calculates charges
            ↓
            3b. Extract Balances (line 246-250)
                currentBalance = wallet.CreditBalance (stored)
                ongoingBalance = balance.RealTimeBalance (calculated with pending usage)
            ↓
            3c. Check Feature-Specific Alerts (line 254-315)
                For each feature with alerts:
                    alertStatus = feature.AlertSettings.AlertState(ongoingBalance)
                    LogAlert(entityType=FEATURE, alertStatus)
                        ↓
                        Publishes webhook if status changed
            ↓
            3d. Check Wallet-Level Alert (line 327-388)
                threshold = wallet.AlertConfig.Threshold.Value
                isOngoingBalanceBelowThreshold = ongoingBalance <= threshold

                If below threshold:
                    alertStatus = IN_ALARM
                Else:
                    alertStatus = OK

                LogAlert(entityType=WALLET, alertType=LOW_ONGOING_BALANCE, alertStatus)
                    ↓
                    If status changed → Publish webhook
            ↓
            3e. Update Wallet Alert State (line 391-405)
                walletService.UpdateWalletAlertState(wallet.ID, alertStatus)
```

**Code Snippet** (line 327-340):
```go
// Check ongoing balance
isOngoingBalanceBelowThreshold := ongoingBalance.LessThanOrEqual(threshold)

// Determine alert status based on balance check
var alertStatus types.AlertState
if isOngoingBalanceBelowThreshold {
    alertStatus = types.AlertStateInAlarm
} else {
    alertStatus = types.AlertStateOk
}
```

---

### Stage 5: Alert Logging & Webhook Publishing

**Alert Service**: `/internal/service/alertlogs.go:74-250`

**LogAlert() Logic**:

```go
Step 1: Get Latest Alert from DB (line 88-102)
    existingAlert = AlertLogsRepo.GetLatestAlert(entityType, entityID, alertType)

Step 2: Determine if Alert Should Be Created (line 126-178)
    Rule 1: No existing alert + OK status → SKIP (healthy from start)
    Rule 2: No existing alert + problem status → CREATE (first problem)
    Rule 3: Existing alert + status changed → CREATE (state transition)
    Rule 4: Existing alert + unchanged → SKIP (no change)

Step 3: Create Alert Log (line 184-210)
    Insert into alert_logs table with:
        - entity_type, entity_id
        - alert_type, alert_status
        - alert_info (threshold, current value, timestamp)

Step 4: Publish Webhook (line 222-250)
    For wallet alerts:
        walletService.PublishEvent(webhookEventName, wallet)
            ↓
            Creates webhook payload with:
                - wallet_id, balance details
                - alert_state, threshold
                - current_balance, credit_balance
            ↓
            Publishes to webhook queue
```

**State Transition Logic** (line 135-178):
```go
if existingAlert == nil {
    // No previous alert exists
    if req.AlertStatus == types.AlertStateOk {
        // System is healthy from the start - no need to log
        return nil
    } else {
        // Problem state detected for first time - create alert
        shouldCreateLog = true
        webhookEventName = getWebhookEventName(req.AlertType, req.AlertStatus)
    }
} else if existingAlert.AlertStatus != req.AlertStatus {
    // Previous alert exists BUT status is different - state changed
    shouldCreateLog = true
    webhookEventName = getWebhookEventName(req.AlertType, req.AlertStatus)
} else {
    // Previous alert exists AND status is the same - no change, skip
    return nil
}
```

---

### Stage 6: Real-Time Alert Logging (Transactional)

**Alternative Path**: Alerts can also trigger **immediately** after wallet transactions

**Location**: `/internal/service/wallet.go:1147-1153`

```go
// After every debit/credit operation:
processWalletOperation() completes
    ↓
Publish transaction webhook
    ↓
logCreditBalanceAlert(wallet, newCreditBalance) // Line 1147
    ↓
    Calculate alert status: newBalance < threshold?
    ↓
    alertService.LogAlert({
        EntityType: WALLET,
        AlertType: LOW_CREDIT_BALANCE,
        AlertStatus: IN_ALARM or OK
    })
        ↓
        Publishes webhook if status changed
```

**Code Reference** (line 635-703):
```go
func (s *walletService) logCreditBalanceAlert(ctx context.Context, w *wallet.Wallet, newCreditBalance decimal.Decimal) error {
    // Get wallet threshold or use default (0)
    if w.AlertConfig != nil && w.AlertConfig.Threshold != nil {
        thresholdValue = w.AlertConfig.Threshold.Value
    } else {
        thresholdValue = decimal.Zero
    }

    // Determine alert status based on balance vs threshold
    if newCreditBalance.LessThan(thresholdValue) {
        alertStatus = types.AlertStateInAlarm
    } else {
        alertStatus = types.AlertStateOk
    }

    // Log the alert
    alertService := NewAlertLogsService(s.ServiceParams)
    return alertService.LogAlert(ctx, &LogAlertRequest{
        EntityType:  types.AlertEntityTypeWallet,
        EntityID:    w.ID,
        AlertType:   types.AlertTypeLowCreditBalance,
        AlertStatus: alertStatus,
        AlertInfo:   alertInfo,
    })
}
```

**This provides near-instantaneous alerts** when transactions push balance below threshold.

---

## 🔗 The Critical Connection: Meters → Wallet Alerts

### How Meter Events Flow to Alerts

```
┌──────────────────────┐
│  1. Meter Event      │
│  POST /events        │ (User sends event with meter data)
└──────────┬───────────┘
           │
           v
┌──────────────────────┐
│  2. Kafka Queue      │ (Async processing)
│  feature_usage topic │
└──────────┬───────────┘
           │
           v
┌──────────────────────┐
│  3. Feature Usage    │ (Consumer processes events)
│  Processing          │ Maps to subscriptions & features
└──────────┬───────────┘
           │
           v
┌──────────────────────┐
│  4. ClickHouse DB    │ (Aggregated usage data)
│  feature_usage table │ subscription_id, feature_id, qty_total
└──────────┬───────────┘
           │
           v
┌──────────────────────┐
│  5. Balance Calc     │ (On wallet query or cron)
│  GetWalletBalance    │ Queries: Subscriptions → Usage → Charges
└──────────┬───────────┘
           │
           v
┌──────────────────────┐
│  6. Alert Check      │ (Compares balance to threshold)
│  Cron: CheckAlerts() │ realTimeBalance <= threshold?
└──────────┬───────────┘
           │
           v
┌──────────────────────┐
│  7. Alert Log        │ (Only if status changed)
│  LogAlert()          │ Creates alert_logs record
└──────────┬───────────┘
           │
           v
┌──────────────────────┐
│  8. Webhook Publish  │ (External notification)
│  wallet.balance.low  │ Sent to configured webhook endpoints
└──────────────────────┘
```

---

## 📊 Data Dependencies

### Per-Meter Linkage

Individual meters affect wallet balances through this chain:

```
Meter → Event → Feature Usage → Subscription Line Item → Usage Charge → Wallet Balance → Alert
```

**Example Flow**:
1. **Meter**: `api_calls` (aggregation: COUNT)
2. **Event**: `{ event_name: "api_call", external_customer_id: "cust_123" }`
3. **Feature Usage**: Records qty=1 for feature_id linked to `api_calls` meter
4. **Subscription**: Has line item with usage-based pricing ($0.01/call)
5. **Usage Charge**: 1000 calls × $0.01 = $10.00 pending
6. **Wallet Balance**: Balance $50 - Pending $10 = $40 real-time
7. **Alert Check**: If threshold=$45, alert status = OK. If threshold=$40, alert status = IN_ALARM

---

## ⚡ Key Characteristics

### 1. Indirect Coupling
- **No direct reference** from wallet alerts to specific meters
- Coupling is **data-driven** through aggregated usage charges
- Meter changes affect alerts through **subscription pricing configuration**

### 2. Dual Trigger Modes

| Trigger | When | Latency | Accuracy |
|---------|------|---------|----------|
| **Cron-Based** | Periodic (e.g., every 15 min) | Minutes | Uses latest usage data |
| **Transaction-Based** | After each debit/credit | Immediate | Uses stored balance only |

**Cron-Based Alert** (line 235):
```go
balance = walletService.GetWalletBalanceV2(wallet.ID)
// Includes pending subscription usage charges from ClickHouse
```

**Transaction-Based Alert** (line 1147):
```go
logCreditBalanceAlert(wallet, newCreditBalance)
// Uses new balance immediately after transaction
```

### 3. Alert Threshold Types

**Wallet-Level Alerts** (`/ent/schema/wallet.go:110-124`):
```go
field.JSON("alert_config", &types.AlertConfig{}).Optional()
field.Bool("alert_enabled").Optional().Default(true)
field.String("alert_state").Optional().Default(string(types.AlertStateOk))
```

Structure:
```go
type AlertConfig struct {
    Threshold *WalletAlertThreshold
}

type WalletAlertThreshold struct {
    Type  AlertThresholdType // "amount" or "percentage"
    Value decimal.Decimal     // e.g., 10.00
}
```

**Feature-Level Alerts** (checked per feature, tied to wallet):
- Evaluated against wallet's `ongoingBalance`
- Can have `critical` and `warning` thresholds
- Logged with `parent_entity_type=wallet`, `parent_entity_id=wallet_id`

### 4. Balance Calculation Modes

```go
// wallet.go:739-741
shouldIncludeUsage :=
    len(wallet.Config.AllowedPriceTypes) == 0 ||
    contains(wallet.Config.AllowedPriceTypes, USAGE) ||
    contains(wallet.Config.AllowedPriceTypes, ALL)
```

**Impact**: If `shouldIncludeUsage = false`, meter events **do NOT affect** the wallet balance used for alerts!

**Configuration Options**:
```go
type WalletConfig struct {
    AllowedPriceTypes []WalletConfigPriceType
}

type WalletConfigPriceType string
const (
    WalletConfigPriceTypeUsage WalletConfigPriceType = "USAGE"
    WalletConfigPriceTypeFixed WalletConfigPriceType = "FIXED"
    WalletConfigPriceTypeAll   WalletConfigPriceType = "ALL"
)
```

---

## 🎯 Summary Answer

### How are alerts tied to meter usage?

1. **Indirect Linkage**: Alerts monitor wallet balances, which are calculated from subscription usage charges, which are derived from meter event aggregations.

2. **Processing Pipeline**:
   - Meter events → Kafka → Feature usage table (ClickHouse) → Usage charge calculation → Real-time balance → Alert evaluation

3. **Cron-Driven**: The primary alert check is a scheduled cron job that:
   - Queries real-time wallet balance (includes pending meter usage)
   - Compares to configured threshold
   - Creates alert log only on **state transitions**
   - Publishes webhooks for external notifications

4. **Granularity**: Alerts are **not per-meter**, but rather:
   - **Per-wallet**: Total balance across all usage
   - **Per-feature** (optional): Balance sufficient for feature with alert settings

5. **Configuration-Dependent**: Whether meter usage affects alerts depends on:
   - `wallet.config.allowed_price_types` (must include USAGE or ALL)
   - `wallet.alert_enabled = true`
   - `wallet.alert_config.threshold` configured

---

## 📝 Key Files Reference

| Component | File Path | Lines |
|-----------|-----------|-------|
| Event Ingestion | `/internal/api/v1/events.go` | 49-73 |
| Event Publishing | `/internal/service/event.go` | 61-78 |
| Feature Usage Processing | `/internal/service/feature_usage_tracking.go` | 173-200 |
| Wallet Balance Calculation | `/internal/service/wallet.go` | 1719-1826 |
| Alert Cron Job | `/internal/api/cron/wallet.go` | 118-411 |
| Alert Logging | `/internal/service/alertlogs.go` | 74-250 |
| Transaction Alert | `/internal/service/wallet.go` | 635-703 |

---

## 🔧 Architecture Characteristics

This architecture provides:

1. **Scalable Processing**: Kafka-based async processing of meter events
2. **Eventual Consistency**: Alerts reflect usage with cron-job latency
3. **Real-Time Transactions**: Immediate alerts on wallet debit/credit
4. **Flexible Configuration**: Per-wallet alert thresholds and price type restrictions
5. **State-Based Alerting**: Only alerts on state transitions, avoiding spam
6. **Webhook Integration**: External system notifications via webhook events

This design enables **scalable, eventual-consistency alerts** without requiring synchronous processing of every meter event.
