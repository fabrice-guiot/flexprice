# Dependency Analysis: Wallet Service ↔ Subscription & Feature Objects

**Date**: 2025-11-25
**Analysis Scope**: Flexprice codebase dependency evaluation

## Executive Summary

The Wallet Service has **moderate to high dependencies** on Subscription objects and **minimal direct dependencies** on Feature objects. The dependencies are primarily:
- **Subscription**: Direct service-level and data-level dependencies for balance calculation and proration credits
- **Feature**: Indirect dependencies through Subscription's feature usage tracking

---

## 1. Wallet Service → Subscription Dependencies

### 1.1 Direct Service Dependencies (HIGH)

#### Balance Calculation (Critical Dependency)
**Location**: `/internal/service/wallet.go:747-798` and `1752-1803`

The Wallet Service has a **critical runtime dependency** on Subscription data for calculating real-time wallet balances:

```go
// GetWalletBalance and GetWalletBalanceV2 both:
// 1. Retrieve all customer subscriptions
subscriptions, err := s.SubRepo.ListByCustomerID(ctx, w.CustomerID)

// 2. Filter by currency
filteredSubscriptions := make([]*subscription.Subscription, 0)
for _, sub := range subscriptions {
    if sub.Currency == w.Currency {
        filteredSubscriptions = append(filteredSubscriptions, sub)
    }
}

// 3. Calculate pending charges from subscription usage
usage, err := subscriptionService.GetUsageBySubscription(ctx, &dto.GetUsageBySubscriptionRequest{
    SubscriptionID: sub.ID,
    StartTime:      periodStart,
    EndTime:        periodEnd,
})

usageCharges, usageTotal, err := billingService.CalculateUsageCharges(ctx, sub, usage, periodStart, periodEnd)
totalPendingCharges = totalPendingCharges.Add(usageTotal)

// 4. Deduct from balance
realTimeBalance := w.Balance.Sub(totalPendingCharges)
```

**Impact**:
- Every wallet balance query potentially triggers subscription queries
- Usage-based price types require active subscription usage calculations
- Performance impact: O(n) subscriptions per balance check

#### Proration Credits (Medium Dependency)
**Location**: `/internal/service/wallet.go:1604-1717`

```go
// TopUpWalletForProratedCharge is called by subscription changes
func (s *walletService) TopUpWalletForProratedCharge(
    ctx context.Context,
    customerID string,
    amount decimal.Decimal,
    currency string
) error
```

**Called from**: `/internal/service/subscription.go:1185-1186`
```go
walletService := NewWalletService(s.ServiceParams)
err = walletService.TopUpWalletForProratedCharge(ctx, subscription.CustomerID, totalCreditAmount.Abs(), subscription.Currency)
```

**Impact**: Subscription plan changes directly trigger wallet top-ups with reason `SUBSCRIPTION_CREDIT_GRANT`

### 1.2 Data-Level Dependencies

#### Repository Access
**Location**: `/internal/service/factory.go:72-75`

```go
type ServiceParams struct {
    SubRepo                  subscription.Repository    // Line 72
    WalletRepo               wallet.Repository          // Line 75
    // ...
}
```

WalletService has direct access to `SubRepo` for querying subscription data.

#### Transaction Reasons (Enumeration Coupling)
**Location**: `/internal/types/wallet.go:55`

```go
const (
    TransactionReasonSubscriptionCredit  TransactionReason = "SUBSCRIPTION_CREDIT_GRANT"
    // Used in wallet.go:1689
)
```

### 1.3 Configuration-Based Dependencies

#### Wallet Price Type Restrictions
**Location**: `/internal/service/wallet.go:739-741`

```go
// Wallets can restrict which price types they pay for
shouldIncludeUsage := len(w.Config.AllowedPriceTypes) == 0 ||
    lo.Contains(w.Config.AllowedPriceTypes, types.WalletConfigPriceTypeUsage) ||
    lo.Contains(w.Config.AllowedPriceTypes, types.WalletConfigPriceTypeAll)
```

This determines whether subscription usage charges are included in balance calculations.

---

## 2. Subscription Service → Wallet Dependencies

### 2.1 Reverse Dependencies (LOW-MEDIUM)

#### Proration Credit Application
**Location**: `/internal/service/subscription.go:1183-1186`

When subscriptions change plans, the Subscription Service:
1. Calculates proration credits
2. Calls WalletService to apply credits

#### Credit Grant Application
**Location**: `/internal/service/subscription.go:3428-3431`

```go
err = creditGrantService.ApplyCreditGrantToWallet(ctx, creditGrant.CreditGrant, sub, cga)
```

Credit grants associated with subscriptions are applied to wallets.

---

## 3. Wallet Service → Feature Dependencies

### 3.1 Direct Dependencies (MINIMAL)

**No direct imports or calls** from Wallet Service to Feature Service or repositories.

### 3.2 Indirect Dependencies via Subscription (MEDIUM)

The dependency chain is:
```
Wallet → Subscription → Feature Usage → Feature
```

**Flow**:
1. **Wallet balance calculation** calls `subscriptionService.GetUsageBySubscription()`
2. **Subscription service** calls `s.FeatureUsageRepo.GetFeatureUsageBySubscription()`
   (Location: `/internal/service/subscription.go:3970`)
3. **Feature usage data** is aggregated and used to calculate charges
4. **Charges** are deducted from wallet balance

#### Feature Usage Repository
**Location**: `/internal/service/factory.go:66`

```go
type ServiceParams struct {
    FeatureUsageRepo  events.FeatureUsageRepository  // Shared by Wallet (via Sub) and Feature services
}
```

### 3.3 Data Schema Dependencies

#### Feature Usage Tracking
**Location**: `/internal/repository/clickhouse/feature_usage.go`

ClickHouse stores feature usage with subscription_id as a key field:
```sql
-- From migrations/clickhouse/000003_create_feature_usage_table.up.sql
CREATE TABLE feature_usage (
    id String,
    subscription_id String,
    feature_id String,
    meter_id String,
    qty_total Decimal(20, 9),
    ...
) ENGINE = ReplacingMergeTree()
```

When wallets calculate pending charges, this table is queried via the subscription ID.

---

## 4. Feature Service → Wallet Dependencies

### 4.1 No Direct Dependencies

Feature Service does **not** import or use Wallet Service or repositories directly.

---

## 5. Shared Dependencies & Coupling Points

### 5.1 Billing Service (Critical Mediator)
**Location**: `/internal/service/billing.go`

The BillingService acts as a bridge:

```go
// Called by Wallet for balance calculation
usageCharges, usageTotal, err := billingService.CalculateUsageCharges(
    ctx, sub, usage, periodStart, periodEnd
)
```

```go
// BillingService aggregates feature entitlements
aggregatedEntitlements, err := subscriptionService.GetAggregatedSubscriptionEntitlements(ctx, sub.ID, nil)

// Maps by meter ID
entitlementsByMeterID := make(map[string]*dto.AggregatedEntitlement)
for _, feature := range aggregatedEntitlements.Features {
    if feature.Feature != nil && feature.Feature.MeterID != "" {
        entitlementsByMeterID[feature.Feature.MeterID] = feature.Entitlement
    }
}
```

### 5.2 Common Repository Access
All three services share `ServiceParams` providing access to:
- `SubRepo` (subscription.Repository)
- `WalletRepo` (wallet.Repository)
- `FeatureRepo` (feature.Repository)
- `FeatureUsageRepo` (events.FeatureUsageRepository)

---

## 6. Database Schema Dependencies

### 6.1 No Foreign Keys Between Tables

**Wallet Schema** (`/ent/schema/wallet.go`):
- `customer_id` (references Customer, not Subscription)
- No direct subscription_id or feature_id fields

**Subscription Schema** (`/ent/schema/subscription.go`):
- `customer_id` (references Customer)
- No wallet_id field

**Feature Usage Table** (ClickHouse):
- `subscription_id` (used for aggregation)
- `feature_id` (links to features)
- No wallet_id

### 6.2 Logical Relationships

```
Customer (1) ─── (N) Wallet
         └──────── (N) Subscription ─── (N) Feature Usage ─── (N) Feature
```

Wallets and Subscriptions are **independently linked to Customers**, not to each other.

---

## 7. Summary & Dependency Metrics

| Dependency Type | Wallet → Subscription | Wallet → Feature | Subscription → Wallet |
|-----------------|----------------------|------------------|----------------------|
| **Direct Service Calls** | ✅ High (2 methods) | ❌ None | ✅ Medium (1 method) |
| **Repository Access** | ✅ Direct (SubRepo) | ❌ None (via Sub) | ✅ Direct (WalletRepo) |
| **Database FK** | ❌ None | ❌ None | ❌ None |
| **Shared Enums/Types** | ✅ TransactionReason | ❌ None | ✅ WalletConfig |
| **Runtime Coupling** | 🔴 **High** (balance calc) | 🟡 Medium (indirect) | 🟡 Medium (proration) |

### Key Findings:

1. **High Runtime Dependency**: Wallet balance calculations have a **critical dependency** on Subscription data for calculating pending charges from usage-based pricing.

2. **Indirect Feature Coupling**: Wallet depends on Features **indirectly** through Subscription's feature usage tracking system.

3. **Loose Schema Coupling**: At the database level, there are **no foreign keys** between Wallet, Subscription, and Feature tables. All relationships are mediated through the Customer entity.

4. **Bidirectional Service Dependency**:
   - Wallet Service calls Subscription Service (for usage/balance)
   - Subscription Service calls Wallet Service (for proration credits)

5. **Shared Service Layer**: The BillingService acts as a critical mediator, aggregating data from Subscriptions, Features, and feeding it to Wallet balance calculations.

---

## 8. Recommendations for Decoupling (Optional)

If reducing dependencies is desired:

1. **Cache Subscription Usage Data**: Pre-calculate and cache pending charges to avoid real-time subscription queries during balance checks

2. **Event-Driven Updates**: Use events to update wallet balances when subscription usage changes, rather than calculating on-demand

3. **Separate Balance Calculation Service**: Extract balance logic into a dedicated service that both Wallet and Subscription depend on

4. **API Gateway Pattern**: Route all cross-service calls through a unified API layer rather than direct service calls
