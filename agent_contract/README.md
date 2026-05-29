# SynxFintech — Wallet Agent Contract

A modular Synx contract that orchestrates fintech operations (transfers,
investments, fraud, FX, Pix, KYC) for a single agent. The main contract is kept
small; all logic lives in `library` files that are pulled in via `import` and
linked into one compilation unit at deploy time.

## Layout

| File | Kind | Contents |
|------|------|----------|
| `wallet_agent.snx` | contract | Imports + the `agent FintechOrchestrator` declaration |
| `fintech_policy.snx` | library `SynxFintechPolicies` | `policy FintechPolicy` — all thresholds and allow-lists |
| `fintech_types.snx` | library `FintechTypes` | The 12 input `type` declarations |
| `fintech_transfers.snx` | library `FintechTransfers` | `verifyTransferAccounts`, `executeTransfer` |
| `fintech_investments.snx` | library `FintechInvestments` | `verifyMarketWindow`, `placeInvestmentOrder` |
| `fintech_fraud.snx` | library `FintechFraud` | `assessFraudRisk`, `recordReviewDecision` |
| `fintech_fx.snx` | library `FintechFx` | `quoteExchange`, `executeExchange` |
| `fintech_pix.snx` | library `FintechPix` | `verifyScheduleWindow`, `commitPixSchedule` |
| `fintech_kyc.snx` | library `FintechKyc` | `verifyIdentity`, `recordKycLevel` |

## How the import / library system works

`wallet_agent.snx` is the **main file** (the `contract`). Each `library` file is
a **module** that is appended onto the main file before compilation.

The model is **FLAT**: declarations inside a `library` register under their own
names — there is **no alias prefix**. The import identifier
(`FintechTransfers`, etc.) only names the dependency; you call its functions and
reference its types by their bare name:

```
import FintechTransfers from "./fintech_transfers.snx"

contract SynxFintech {
  fn someFlow(request: TransferRequest): Event {   // type from FintechTypes
    verifyTransferAccounts(request)                // fn from FintechTransfers
  }
}
```

### Important: import order matters

The function libraries reference `FintechPolicy.*` and the `FintechTypes`.
Because modules are folded in import order and slots are allocated as they are
compiled, **the policy and types must be imported before the function
libraries** — otherwise `FintechPolicy` would bind to the wrong storage slot.
Keep the order as written in `wallet_agent.snx`:

```
import SynxFintechPolicies ...   # policy first
import FintechTypes ...          # then types
import FintechTransfers ...      # then the function libraries
import FintechInvestments ...
import FintechFraud ...
import FintechFx ...
import FintechPix ...
import FintechKyc ...
```

## Agent

```
agent FintechOrchestrator {
  version: "1.0.0"
  owner: 0xA1B2C3D4
  purpose: "fintech_orchestration"
}
```

## Policy — `FintechPolicy`

| Field | Value | Used by |
|-------|-------|---------|
| `minTransferAmount` / `maxTransferAmount` | 1 / 50000 | transfers |
| `maxDailyTransfer` | 200000 | transfers (daily limit) |
| `allowedAssets` | PETR4, VALE3, ITUB4, MGLU3, BTC, USD | investments |
| `minOrderTicket` / `maxOrderTicket` | 100 / 100000 | investments |
| `maxPositionExposure` | 500000 | investments |
| `marketOpenHour` / `marketCloseHour` | 10 / 17 | investments |
| `fraudReviewThreshold` / `fraudBlockThreshold` | 60 / 85 | fraud |
| `allowedCurrencyPairs` | BRL/USD, USD/BRL, BRL/EUR, EUR/BRL | FX |
| `minFxAmount` / `maxFxAmount` | 50 / 500000 | FX |
| `fxSpreadBps` | 50 | FX |
| `fxQuoteValiditySeconds` | 60 | FX |
| `pixMinDelaySeconds` / `pixMaxDelaySeconds` | 60 / 2592000 | Pix |
| `minAge` | 18 | KYC |
| `kycBasicMinScore` / `kycFullMinScore` | 50 / 80 | KYC |
| `blockedCpfs` | [] | KYC |

## Functions

Each function takes a single typed input object and emits an `Event`. Validation
failures are raised with `Error(code, message)` and caught in a `try/catch` that
emits a structured rejection event.

| Function | Input type | Success event(s) |
|----------|-----------|------------------|
| `verifyTransferAccounts` | `TransferRequest` | `TransferAccountsVerified` |
| `executeTransfer` | `TransferSettlement` | `TransferExecuted` |
| `verifyMarketWindow` | `InvestmentOrder` | `MarketWindowVerified` |
| `placeInvestmentOrder` | `OrderExecution` | `InvestmentOrderPlaced` |
| `assessFraudRisk` | `FraudActivity` | `FraudReviewRequired` / `FraudRiskCleared` |
| `recordReviewDecision` | `ReviewDecision` | `ReviewDecisionApproved` / `ReviewDecisionDenied` |
| `quoteExchange` | `FxRequest` | `ExchangeQuoted` |
| `executeExchange` | `FxExecution` | `ExchangeExecuted` |
| `verifyScheduleWindow` | `PixSchedule` | `PixScheduleWindowVerified` |
| `commitPixSchedule` | `PixCommit` | `PixScheduled` |
| `verifyIdentity` | `IdentityCheck` | `IdentityVerified` |
| `recordKycLevel` | `KycResult` | `KycLevelAssigned` / `KycLevelRejected` |

### Input types (`FintechTypes`)

```
TransferRequest    { from, to: Address; amount, dailyTotal: UInt; currency: String }
TransferSettlement { transferId: String; from, to: Address; amount: UInt }
InvestmentOrder    { client: Address; symbol, side: String; quantity, unitPrice, currentHour, currentExposure: UInt }
OrderExecution     { orderId: String; client: Address; symbol, side: String; quantity, totalAmount: UInt }
FraudActivity      { client: Address; action: String; amount, accountAgeDays, geoRisk, deviceTrust: UInt }
ReviewDecision     { riskId: String; client, reviewer: Address; riskScore, approved: UInt }
FxRequest          { client: Address; pair: String; amount, marketRate: UInt }
FxExecution        { quoteId: String; client: Address; pair: String; amount, finalRate, quoteTime, currentTime: UInt }
PixSchedule        { payer, receiver: Address; amount, scheduledAt, currentTime: UInt; description: String }
PixCommit          { scheduleId: String; payer, receiver: Address; amount, scheduledAt: UInt }
IdentityCheck      { client: Address; cpf: String; age, faceMatchScore, documentScore: UInt }
KycResult          { client: Address; cpf: String; totalScore: UInt }
```

## Notes & current limitations

- **FLAT namespace** — function and type names are global across all modules.
  Two libraries declaring the same name will collide (last compiled wins). There
  is no visibility/export marker yet.
- **Library bodies are not semantically analyzed** — only the `contract` body is
  validated; errors inside library functions surface at compile/runtime.
- **Import order is load-bearing** — see the ordering note above.
