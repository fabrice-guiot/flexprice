# Usage-Based Credits Implementation: Wallet & Events Ingestion

**Date**: 2025-11-25
**Scope**: Credit grant system, wallet integration, and relationship to events ingestion

## Executive Summary

The Flexprice system implements a **credit grant system** that provides prepaid credits to customer wallets. Despite the term "usage-based," credits are **NOT granted based on meter events or actual usage**. Instead:

1. **Credits are TIME-BASED**: Granted periodically (recurring) or once (one-time) based on subscription lifecycle
2. **Credits PAY FOR usage**: Used to cover usage-based pricing charges from meter events
3. **No event-triggered credits**: Meter ingestion does NOT trigger credit allocations

---

## 🎯 Terminology Clarification

### What "Usage-Based Credits" Actually Means

| Term | What It DOES Mean | What It DOESN'T Mean |
|------|-------------------|----------------------|
| **Usage-Based Credits** | Credits that can be **spent on** usage-based charges | ❌ Credits **granted based on** meter usage |
| **Credit Grant** | Prepaid allocation to wallet | ❌ Post-paid refund for usage |
| **Recurring Credits** | Periodic wallet top-ups | ❌ Usage-triggered allocations |

---

## 📊 Credit Grant System Architecture

### 1. Core Entities

#### 1.1 Credit Grant Entity
**Location**: `/internal/domain/creditgrant/model.go`

```go
type CreditGrant struct {
    ID                     string                    // Unique identifier
    Name                   string                    // Display name
    Scope                  CreditGrantScope          // PLAN or SUBSCRIPTION
    PlanID                 *string                   // For PLAN-scoped grants
    SubscriptionID         *string                   // For SUBSCRIPTION-scoped grants
    Credits                decimal.Decimal           // Amount of credits
    Cadence                CreditGrantCadence        // ONETIME or RECURRING
    Period                 *CreditGrantPeriod        // MONTHLY, ANNUAL, etc.
    PeriodCount            *int                      // Number of periods
    ExpirationType         CreditGrantExpiryType     // NEVER, DURATION, BILLING_CYCLE
    ExpirationDuration     *int                      // Expiry time value
    ExpirationDurationUnit *CreditGrantExpiryDurationUnit // DAY, WEEK, MONTH, YEAR
    Priority               *int                      // Application order
}
```

#### Key Properties:

**Scope Types**:
```go
const (
    CreditGrantScopePlan         CreditGrantScope = "PLAN"         // Template for all subscriptions
    CreditGrantScopeSubscription CreditGrantScope = "SUBSCRIPTION" // Specific subscription override
)
```

**Cadence Types**:
```go
const (
    CreditGrantCadenceOneTime   CreditGrantCadence = "ONETIME"   // Single allocation
    CreditGrantCadenceRecurring CreditGrantCadence = "RECURRING" // Periodic allocation
)
```

**Period Types** (for recurring grants):
```go
const (
    CREDIT_GRANT_PERIOD_DAILY       CreditGrantPeriod = "DAILY"
    CREDIT_GRANT_PERIOD_WEEKLY      CreditGrantPeriod = "WEEKLY"
    CREDIT_GRANT_PERIOD_MONTHLY     CreditGrantPeriod = "MONTHLY"
    CREDIT_GRANT_PERIOD_QUARTERLY   CreditGrantPeriod = "QUARTERLY"
    CREDIT_GRANT_PERIOD_HALF_YEARLY CreditGrantPeriod = "HALF_YEARLY"
    CREDIT_GRANT_PERIOD_ANNUAL      CreditGrantPeriod = "ANNUAL"
)
```

**Expiration Types**:
```go
const (
    CreditGrantExpiryTypeNever        CreditGrantExpiryType = "NEVER"         // Credits never expire
    CreditGrantExpiryTypeDuration     CreditGrantExpiryType = "DURATION"      // Expire after X days/months
    CreditGrantExpiryTypeBillingCycle CreditGrantExpiryType = "BILLING_CYCLE" // Expire at period end
)
```

#### 1.2 Credit Grant Application (CGA)
**Location**: `/internal/domain/creditgrantapplication/model.go`

Tracks the application of credit grants to subscriptions:

```go
type CreditGrantApplication struct {
    ID                              string                        // Unique ID
    CreditGrantID                   string                        // Which grant
    SubscriptionID                  string                        // Which subscription
    ScheduledFor                    time.Time                     // When to apply
    PeriodStart                     *time.Time                    // Period start (recurring)
    PeriodEnd                       *time.Time                    // Period end (recurring)
    ApplicationStatus               CreditGrantApplicationStatus  // PENDING, APPLIED, FAILED, etc.
    ApplicationReason               CreditGrantApplicationReason  // Why applied
    SubscriptionStatusAtApplication SubscriptionStatus            // Sub status when applied
    AppliedAt                       *time.Time                    // When successfully applied
    FailureReason                   *string                       // Error details if failed
    RetryCount                      int                           // Retry attempts
    Credits                         decimal.Decimal               // Amount applied
    IdempotencyKey                  string                        // Prevent duplicates
}
```

**Application Statuses**:
```go
const (
    ApplicationStatusPending   ApplicationStatus = "PENDING"   // Scheduled, not yet applied
    ApplicationStatusApplied   ApplicationStatus = "APPLIED"   // Successfully applied to wallet
    ApplicationStatusFailed    ApplicationStatus = "FAILED"    // Application failed
    ApplicationStatusSkipped   ApplicationStatus = "SKIPPED"   // Skipped (e.g., cancelled sub)
    ApplicationStatusCancelled ApplicationStatus = "CANCELLED" // Manually cancelled
)
```

---

## 🔄 Credit Grant Lifecycle

### Flow 1: One-Time Credit Grant (Subscription Creation)

```
┌─────────────────────────┐
│ User Creates            │
│ Subscription            │
│ with Plan               │
└───────────┬─────────────┘
            │
            v
┌─────────────────────────┐
│ Get Plan's              │
│ Credit Grants           │ (PLAN-scoped grants)
└───────────┬─────────────┘
            │
            v
┌─────────────────────────┐
│ For Each Grant:         │
│ Create CGA Record       │ (Status: PENDING)
└───────────┬─────────────┘
            │
            v
┌─────────────────────────┐
│ Check Subscription      │
│ Status                  │
└───────────┬─────────────┘
            │
            ├─── ACTIVE ───────────────┐
            │                          v
            │                  ┌─────────────────┐
            │                  │ Apply Credit    │
            │                  │ Immediately     │
            │                  └────────┬────────┘
            │                           │
            ├─── INCOMPLETE ────────────┤
            │                           │
            │                           v
            │                  ┌─────────────────┐
            │                  │ Defer Credit    │
            │                  │ (Wait for       │
            │                  │  Active Status) │
            │                  └────────┬────────┘
            │                           │
            └───────────────────────────┘
                                        │
                                        v
                                ┌─────────────────┐
                                │ Top-Up Wallet   │
                                │ with Credits    │
                                └─────────────────┘
```

**Code Reference**: `/internal/service/subscription.go:839-938`

```go
func (s *subscriptionService) handleCreditGrants(
    ctx context.Context,
    subscription *subscription.Subscription,
    creditGrantRequests []dto.CreateCreditGrantRequest,
) error {
    for _, grantReq := range creditGrantRequests {
        // Create credit grant in DB
        createdGrant, err := creditGrantService.CreateCreditGrant(ctx, grantReq)

        // Determine action based on subscription status
        stateHandler := NewSubscriptionStateHandler(subscription, createdGrant.CreditGrant)
        action, err := stateHandler.DetermineCreditGrantAction()

        if action == StateActionApply {
            // Apply immediately to wallet
            err = creditGrantService.ApplyCreditGrant(ctx, createdGrant.CreditGrant, subscription, metadata)
        } else if action == StateActionDefer {
            // Create scheduled CGA for later processing
            _, err = creditGrantService.CreateScheduledCreditGrantApplication(ctx, createdGrant.CreditGrant, subscription, metadata)
        }
    }
}
```

---

### Flow 2: Recurring Credit Grant (Periodic Processing)

```
┌─────────────────────────┐
│ Cron Job (Every 15 min) │
│ ProcessScheduled        │
│ CreditGrant             │
│ Applications()          │
└───────────┬─────────────┘
            │
            v
┌─────────────────────────┐
│ Query DB for            │
│ Pending CGAs            │
│ WHERE scheduled_for     │
│   <= NOW()              │
└───────────┬─────────────┘
            │
            v
┌─────────────────────────┐
│ For Each CGA:           │
│ 1. Get Subscription     │
│ 2. Get Credit Grant     │
│ 3. Check Sub Status     │
└───────────┬─────────────┘
            │
            v
┌─────────────────────────┐
│ State-Based Action      │
│ Determination           │
└───────────┬─────────────┘
            │
            ├─── APPLY ──────────────┐
            │                        v
            │               ┌──────────────────┐
            │               │ Execute          │
            │               │ Transaction:     │
            │               │ 1. Top-up Wallet │
            │               │ 2. Update CGA    │
            │               │    to APPLIED    │
            │               │ 3. Create Next   │
            │               │    Period CGA    │
            │               │    (if recurring)│
            │               └──────────────────┘
            │
            ├─── DEFER ──────────────┐
            │                        v
            │               ┌──────────────────┐
            │               │ Update CGA       │
            │               │ scheduled_for    │
            │               │ to future date   │
            │               └──────────────────┘
            │
            ├─── SKIP ───────────────┐
            │                        v
            │               ┌──────────────────┐
            │               │ Update CGA to    │
            │               │ SKIPPED status   │
            │               └──────────────────┘
            │
            └─── CANCEL ─────────────┐
                                    v
                           ┌──────────────────┐
                           │ Cancel Future    │
                           │ Applications     │
                           └──────────────────┘
```

**Cron Job Handler**: `/internal/api/cron/creditgrant.go:24-34`

```go
func (h *CreditGrantCronHandler) ProcessScheduledCreditGrantApplications(c *gin.Context) {
    h.logger.Infow("starting credit grant scheduled applications cron job")

    resp, err := h.creditGrantService.ProcessScheduledCreditGrantApplications(c.Request.Context())

    c.JSON(http.StatusOK, resp)
}
```

**Service Implementation**: `/internal/service/creditgrant.go:524-563`

```go
func (s *creditGrantService) ProcessScheduledCreditGrantApplications(ctx context.Context) (*dto.ProcessScheduledCreditGrantApplicationsResponse, error) {
    // Find all scheduled applications (WHERE scheduled_for <= NOW() AND status = PENDING)
    applications, err := s.CreditGrantApplicationRepo.FindAllScheduledApplications(ctx)

    // Process each application
    for _, cga := range applications {
        err := s.processScheduledApplication(ctx, cga)
        if err != nil {
            response.FailedApplicationsCount++
        } else {
            response.SuccessApplicationsCount++
        }
    }

    return response, nil
}
```

---

### Flow 3: Credit Application to Wallet (Atomic Transaction)

**Location**: `/internal/service/creditgrant.go:354-484`

```go
func (s *creditGrantService) ApplyCreditGrantToWallet(
    ctx context.Context,
    grant *creditgrant.CreditGrant,
    subscription *subscription.Subscription,
    cga *CreditGrantApplication,
) error {
    // 1. Find or create wallet for customer
    wallets, err := walletService.GetWalletsByCustomerID(ctx, subscription.CustomerID)

    var selectedWallet *dto.WalletResponse
    for _, w := range wallets {
        if types.IsMatchingCurrency(w.Currency, subscription.Currency) {
            selectedWallet = w
            break
        }
    }

    if selectedWallet == nil {
        // Create new wallet with config allowing USAGE price types
        walletReq := &dto.CreateWalletRequest{
            Name:       "Subscription Wallet",
            CustomerID: subscription.CustomerID,
            Currency:   subscription.Currency,
            Config: &types.WalletConfig{
                AllowedPriceTypes: []types.WalletConfigPriceType{
                    types.WalletConfigPriceTypeUsage, // Allows usage-based charges
                },
            },
        }
        selectedWallet, err = walletService.CreateWallet(ctx, walletReq)
    }

    // 2. Calculate expiry date
    var expiryDate *time.Time

    if grant.ExpirationType == types.CreditGrantExpiryTypeNever {
        expiryDate = nil // Credits never expire
    }

    if grant.ExpirationType == types.CreditGrantExpiryTypeDuration {
        // Expire after X days/weeks/months/years
        switch *grant.ExpirationDurationUnit {
        case types.CreditGrantExpiryDurationUnitDays:
            expiry := subscription.StartDate.AddDate(0, 0, *grant.ExpirationDuration)
            expiryDate = &expiry
        case types.CreditGrantExpiryDurationUnitMonths:
            expiry := subscription.StartDate.AddDate(0, *grant.ExpirationDuration, 0)
            expiryDate = &expiry
        // ... other units
        }
    }

    if grant.ExpirationType == types.CreditGrantExpiryTypeBillingCycle {
        expiryDate = &subscription.CurrentPeriodEnd // Expire at period end
    }

    // 3. Prepare top-up request
    topupReq := &dto.TopUpWalletRequest{
        CreditsToAdd:      cga.Credits,
        TransactionReason: types.TransactionReasonSubscriptionCredit, // Special reason code
        ExpiryDateUTC:     expiryDate,
        Priority:          grant.Priority,
        IdempotencyKey:    &cga.ID, // Prevent duplicate applications
        Metadata: map[string]string{
            "grant_id":        grant.ID,
            "subscription_id": subscription.ID,
            "cga_id":          cga.ID,
        },
    }

    // 4. Execute in atomic transaction
    err = s.DB.WithTx(ctx, func(txCtx context.Context) error {
        // Task 1: Apply credit to wallet
        _, err := walletService.TopUpWallet(txCtx, selectedWallet.ID, topupReq)
        if err != nil {
            return err // Rollback entire transaction
        }

        // Task 2: Update CGA status to APPLIED
        cga.ApplicationStatus = types.ApplicationStatusApplied
        cga.AppliedAt = &time.Now()
        cga.FailureReason = nil

        if err := s.CreditGrantApplicationRepo.Update(txCtx, cga); err != nil {
            return err // Rollback
        }

        // Task 3: Create next period application if recurring
        if grant.Cadence == types.CreditGrantCadenceRecurring {
            if err := s.createNextPeriodApplication(txCtx, grant, subscription, *cga.PeriodEnd); err != nil {
                return err // Rollback
            }
        }

        return nil // Commit all changes
    })

    return err
}
```

---

## 🔗 Integration with Wallet & Events

### How Credits CONSUME Usage (Not Vice Versa)

```
┌──────────────────┐
│ Meter Events     │  (User's API calls, resource usage, etc.)
└────────┬─────────┘
         │
         v
┌──────────────────┐
│ Kafka Queue      │  (Async processing)
└────────┬─────────┘
         │
         v
┌──────────────────┐
│ ClickHouse:      │  (Aggregated usage data)
│ feature_usage    │
└────────┬─────────┘
         │
         v
┌──────────────────┐
│ Invoice          │  (Calculates charges from usage)
│ Generation       │
└────────┬─────────┘
         │
         v
┌──────────────────┐
│ Wallet Debit     │  ← CREDITS ARE CONSUMED HERE
│ for Usage        │     (Using credits granted earlier)
└──────────────────┘
```

**Credit Flow** (Separate, time-based):
```
┌──────────────────┐
│ Subscription     │  (Time-based trigger)
│ Created/Renewed  │
└────────┬─────────┘
         │
         v
┌──────────────────┐
│ Credit Grant     │  (Allocate credits)
│ Applied          │
└────────┬─────────┘
         │
         v
┌──────────────────┐
│ Wallet Top-Up    │  ← CREDITS ARE ADDED HERE
│ Transaction      │     (Before usage occurs)
└──────────────────┘
```

### Critical Distinction

| Aspect | Credit Grants | Usage Events |
|--------|--------------|--------------|
| **Trigger** | Time-based (subscription lifecycle) | User actions (API calls, usage) |
| **Direction** | Adds credits TO wallet | Deducts credits FROM wallet |
| **Timing** | Proactive (prepaid) | Reactive (post-usage) |
| **Frequency** | Periodic or one-time | Continuous (real-time) |
| **Purpose** | Provide budget | Consume budget |

---

## 🎛️ Wallet Configuration for Usage-Based Credits

When credits are granted, wallets are configured to **accept usage-based charges**:

**Location**: `/internal/service/creditgrant.go:376-380`

```go
walletReq := &dto.CreateWalletRequest{
    Name:       "Subscription Wallet",
    CustomerID: subscription.CustomerID,
    Currency:   subscription.Currency,
    Config: &types.WalletConfig{
        AllowedPriceTypes: []types.WalletConfigPriceType{
            types.WalletConfigPriceTypeUsage, // Key configuration
        },
    },
}
```

**This configuration enables**:
- Wallet balance calculations to include pending usage charges
- Debit operations for usage-based pricing
- Alert triggers based on real-time balance (credits - pending usage)

**Reference**: See previous analysis in `meter-usage-to-wallet-alerts-flow.md` for how wallet balance includes pending usage.

---

## 🔄 State-Based Credit Grant Actions

**Location**: `/internal/service/subscription_state_handler.go`

The system uses a state handler to determine actions based on subscription status:

| Subscription Status | Action | Behavior |
|---------------------|--------|----------|
| **ACTIVE** | APPLY | Apply credits immediately to wallet |
| **TRIALING** | APPLY | Apply credits (trial users get credits) |
| **INCOMPLETE** | DEFER | Schedule for later (waiting for payment) |
| **PAST_DUE** | DEFER | Schedule for later (waiting for payment) |
| **CANCELLED** | CANCEL | Cancel future grants, no more credits |
| **UNPAID** | SKIP | Skip this period |

```go
func (h *SubscriptionStateHandler) DetermineCreditGrantAction() (StateAction, error) {
    switch h.subscription.SubscriptionStatus {
    case types.SubscriptionStatusActive, types.SubscriptionStatusTrialing:
        return StateActionApply, nil

    case types.SubscriptionStatusIncomplete, types.SubscriptionStatusPastDue:
        return StateActionDefer, nil

    case types.SubscriptionStatusCancelled:
        return StateActionCancel, nil

    case types.SubscriptionStatusUnpaid:
        return StateActionSkip, nil

    default:
        return "", errors.New("unknown subscription status")
    }
}
```

---

## 📅 Recurring Credit Grant Period Management

For recurring grants, the system automatically creates the next period's application:

**Location**: `/internal/service/creditgrant.go:459-462`

```go
// Inside transaction after applying current period
if grant.Cadence == types.CreditGrantCadenceRecurring {
    if err := s.createNextPeriodApplication(txCtx, grant, subscription, *cga.PeriodEnd); err != nil {
        return err // Rollback if next period creation fails
    }
}
```

**Next Period Calculation**:
- **Monthly**: Current period end + 1 month
- **Quarterly**: Current period end + 3 months
- **Annual**: Current period end + 1 year

This creates a chain of scheduled CGAs for future processing.

---

## 🔐 Idempotency & Data Integrity

### Preventing Duplicate Credits

**Idempotency Key Generation**:
```go
idempotencyKey := fmt.Sprintf(
    "recurring_%s_%s_%s_%s",
    grantID,
    subscriptionID,
    periodStart.Format("2006-01-02"),
    periodEnd.Format("2006-01-02"),
)
```

**Usage in Wallet Top-Up**:
```go
topupReq := &dto.TopUpWalletRequest{
    IdempotencyKey: &cga.ID, // CGA ID ensures uniqueness
    // ... other fields
}
```

The wallet service checks for existing transactions with the same idempotency key before creating new ones.

---

## 📊 Data Model Relationships

```
┌─────────────┐
│ Plan        │ 1
└──────┬──────┘
       │
       │ N
       v
┌─────────────┐         ┌──────────────────┐
│ CreditGrant │ 1 ───→ N │ CreditGrant      │
│ (PLAN scope)│         │ Application (CGA)│
└─────────────┘         └────────┬─────────┘
                                 │
                                 │ N
                                 │
                                 v 1
┌─────────────┐         ┌──────────────────┐
│Subscription │ 1 ───→ N │ CreditGrant      │
└──────┬──────┘         │ (SUBSCRIPTION    │
       │                │  scope)          │
       │                └──────────────────┘
       │
       │ 1
       v
┌─────────────┐
│ Customer    │ 1
└──────┬──────┘
       │
       │ N
       v
┌─────────────┐         ┌──────────────────┐
│ Wallet      │ 1 ───→ N │ WalletTransaction│
└─────────────┘         │ (reason:         │
                        │  SUBSCRIPTION_   │
                        │  CREDIT_GRANT)   │
                        └──────────────────┘
```

---

## 🎯 Summary: "Usage-Based Credits" Explained

### What The System DOES:

1. **Grants credits time-based**: Credits allocated on subscription start/renewal
2. **Credits pay for usage**: Granted credits cover usage-based pricing charges
3. **Periodic processing**: Cron job applies scheduled recurring credits
4. **Atomic transactions**: Ensures credit application is all-or-nothing
5. **State-aware**: Subscription status determines credit grant actions
6. **Expiration support**: Credits can expire based on time or billing cycle
7. **Priority ordering**: Multiple credits consumed in priority order

### What The System DOES NOT:

1. ❌ Grant credits based on meter events
2. ❌ Increase credits when usage increases
3. ❌ Trigger credits from event ingestion
4. ❌ Calculate credits from ClickHouse feature_usage data
5. ❌ React to usage patterns for credit allocation

---

## 🔍 Key Differences: Commitment vs Credit Grants

The system also has a **commitment-based pricing** feature (separate from credit grants):

| Feature | Credit Grants | Commitment Pricing |
|---------|--------------|-------------------|
| **What it is** | Prepaid credits added to wallet | Minimum spend guarantee |
| **When applied** | Subscription start/renewal | During invoice calculation |
| **Purpose** | Provide budget for usage | Ensure minimum revenue |
| **Overage** | Usage beyond credits = invoiced | Usage beyond commitment = higher rate |
| **Refundable** | No (credits expire) | No (commitment charged regardless) |

**Commitment Example** (from `/docs/prds/commitment-overage.md`):
- Commitment: $1000/month
- Overage Factor: 1.5x
- Usage: $1200 → Bill: $1000 (commitment) + $200 * 1.5 (overage) = $1300

---

## 📁 Key Files Reference

| Component | File Path |
|-----------|-----------|
| Credit Grant Model | `/internal/domain/creditgrant/model.go` |
| CGA Model | `/internal/domain/creditgrantapplication/model.go` |
| Credit Grant Service | `/internal/service/creditgrant.go` |
| Subscription Handler | `/internal/service/subscription.go:839-938` |
| State Handler | `/internal/service/subscription_state_handler.go` |
| Cron Job | `/internal/api/cron/creditgrant.go` |
| Credit Grant Types | `/internal/types/creditgrant.go` |
| Credit Grant Schema | `/ent/schema/creditgrant.go` |
| CGA Schema | `/ent/schema/creditgrantapplication.go` |

---

## 🏗️ Architecture Principles

1. **Separation of Concerns**: Credit allocation is independent of usage tracking
2. **Idempotency**: All operations use idempotency keys to prevent duplicates
3. **Atomic Transactions**: Multi-step operations wrapped in DB transactions
4. **State-Based Logic**: Actions determined by subscription status
5. **Async Processing**: Scheduled grants processed by cron, not real-time
6. **Event-Driven**: Webhook events published for credit applications

This design ensures reliable, predictable credit allocation without coupling to the unpredictable nature of usage event ingestion.
