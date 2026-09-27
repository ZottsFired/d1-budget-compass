# Budget Compass: usability improvements brief

This brief is ready to paste into a coding agent working in the **Budget Compass app source**.

> **Gap:** this repository (`d1-budget-compass`) contains only the Cloudflare Workers + D1 starter template (`/api/beverages`). The app source is not here. The brief therefore describes behaviour and data changes against the app's **backup format v2**, not file-level edits. Before starting, locate:
> - the backup import/export code
> - the forecast engine that expands recurrences (for example `monthlyPattern: "nth-last-working-day"`)
> - the reconciliation flow (transactions with `adjustmentTarget: "account-balance"`)
> - the account editor (`repayment`, `reserveTarget`, `reserveSweep*`)
> - the bill editor (`lineItems`, `sharing`, `funding`)

## Objective

1. Make it quick to keep the app accurate.
2. Make "what can I spend this week, and am I on track?" obvious on the dashboard.
3. Don't rework systems that already work well: recurrences, payday planning cycles, bills with line items, joint-bill sharing, bill funding, and reserve sweeps.

## Data model to reuse (backup v2)

Top-level keys: `accounts`, `categories`, `bills`, `incomeStreams`, `recurringTransfers`, `transactions`, `planning`, `settings`, `meta` (`meta.version: 2`).

- **Account:** `type` is one of `current`, `savings`, `credit-card` or `loan`. It also has `openingBalance`, `actualBalance`, `isClosed`, `includeInSafeToSpend`, and a `repayment` object: `{ mode: none|fixed|percentage, fromAccountId, fixedAmount, recurrence, startDate }`.
- **Bill:** `amount`, `recurrence`, `startDate`, `endDate?`, `accountId`, `lineItems[]`, `sharing: { isJoint, yourSharePercent, partnerName }`, and `funding: { enabled, fromAccountId, leadDays }`.
- **Transaction:** `type` is one of `expense`, `income`, `transfer` or `adjustment`. It also has `status` (`actual` or `planned`), `linkedRecurringId`, `linkedOccurrenceKey`, `adjustmentTarget`, `roundup`, `tag` and `toAccountId`.
- **Settings:** `planningCycleMode: "payday"`, `planningCycleRecurrence`, `defaultJointSharePercent`, `lowBalanceThreshold`, `expenseRoundupDefaults`.

## Friction seen in a real backup (anonymised)

| # | What happens today | Why it matters |
|---|---|---|
| 1 | About 38% of ledger entries are balance reconciliations in a catch-all category. For months, almost no spending was itemised. | The app knows balances but not where the money went. |
| 2 | A credit card used for everyday purchases appears only as periodic reconciliation adjustments. | Nothing warns that card debt is rebuilding. |
| 3 | Instalment-plan repayments are typed in by hand as planned transfers, including a duplicate £0 entry. | Easy to miss or double-count a repayment. |
| 4 | One monthly bill with a changing amount (childcare) is stored as nine separate one-off bills. | Nine records to maintain for one cost. |
| 5 | Joint bills use two different share percentages because the default changed and older bills weren't updated. | Forecast costs drift from the real split. |
| 6 | A loan is recorded at its total repayable amount because there is nowhere to enter APR or term. | Debt looks higher than it is, and overpayments can't be modelled. |
| 7 | A windfall rule ("over £X: 75% to debt, 25% keep") exists only in a free-text note. | The app never acts on it. |
| 8 | An active bill points at a closed account. | The bill may silently drop out of the forecast. |

## Scope, in priority order

### 1. Bank statement import (Monzo first)

- **Input formats:**
  - Accept CSV and XLSX.
  - For Monzo XLSX, sheets are named `Personal Account Transactions` and `Joint Account Transactions`.
  - Columns: `Transaction ID, Date, Time, Type, Name, Emoji, Category, Amount, Currency, Local amount, Local currency, Notes and #tags, Address, Receipt, Description, Category split`.
- **Target account:** the user picks a target account for each file or sheet. Remember the choice.
- **De-duplication:** store the bank ID on the transaction as a new optional field `externalId`. Re-importing the same file must create nothing new.
- **Rows to ignore:** zero-amount rows (Monzo "Active card check").
- **Categories:** map bank categories to app categories through an editable table stored in `settings`. Add payee rules on top, for example `Name contains "Tesco"` → Groceries.
- **Own-account movements become transfers, not expenses:**
  - Pot transfers, mapped per pot. Savings-pot round-ups become `transfer` with `roundup` set.
  - Transfers to the joint account.
  - Credit-card payments (Monzo `Type = Flex`).
  - Loan payments (`Type = Instalment loan`).
- **Matching planned items:** match imported rows to planned occurrences (same account, ±3 days, amount within tolerance). Mark the occurrence actual using the existing `linkedRecurringId` and `linkedOccurrenceKey` instead of adding a duplicate.
- **Preview before commit:**
  - Counts of new, duplicate, matched and uncategorised rows.
  - Bulk re-categorise.
  - A single undoable commit.
- **Balance check:** after import, if the user enters the statement balance, show the unexplained difference. Do not auto-create a catch-all adjustment.

**Acceptance checks**
- [ ] Importing the same statement twice adds zero transactions.
- [ ] Round-ups, pot moves and card repayments import as transfers.
- [ ] Planned bills and salary are matched, not duplicated.
- [ ] A v2 backup without `externalId` still imports and exports losslessly.

### 2. "Left to spend" on the dashboard

- **Calculation:** for the current payday cycle:
  - start from income received plus income still expected;
  - subtract planned bills, transfers and debt payments;
  - subtract the savings or overpayment goal;
  - subtract the reserve target;
  - divide by the weeks remaining.
- **Display:** show this week's allowance, spent so far, and a pace indicator.
- **Credit cards:** purchases on credit cards count as spending against the allowance. Paying the card does not count again.

**Acceptance checks**
- [ ] Changing a bill or goal updates the allowance immediately.
- [ ] A card purchase and its later repayment are counted once.

### 3. Instalment plans on credit cards (BNPL / Flex-style)

- **Plan fields:** a credit-card account gets `plans[]` entries of the form `{ id, description, purchaseDate, amount, months (1|3|6|12), apr, firstPaymentDate }`.
- **Forecast:** generate each monthly payment as a planned transfer from `repayment.fromAccountId`. This replaces hand-typed transfers.
- **Display:**
  - committed future payments by month;
  - new borrowing since last payday;
  - interest-bearing plans flagged.
- **Warning:** when the card's balance owed is higher than at the same point last cycle.

**Acceptance checks**
- [ ] A 3-month 0% plan yields three equal planned payments that sum to the purchase.
- [ ] Deleting a plan removes its future planned payments only.

### 4. Loan terms and overpayment planner

- **Loan fields:** `principal`, `apr`, `termMonths`, `startDate`, `paymentDay` and an optional `earlySettlementDaysInterest`.
- **Schedule:** build an amortisation schedule.
- **Display:** show the balance owed (principal plus accrued interest) separately from the total repayable.
- **Simulator:** take a one-off and/or a monthly overpayment and show the new debt-free date and interest saved.

**Acceptance checks**
- [ ] With no overpayment, the schedule ends on the contract end date, and total paid equals payment × term (within rounding).

### 5. Variable-amount recurring bills

- **Overrides:** allow per-occurrence amount overrides on a recurring bill, plus an end date.
- **Merge helper (optional):** offer to merge a run of one-off bills that share a name prefix into one recurring bill with overrides. It must only run after the user confirms.

**Acceptance check**
- [ ] The forecast for the merged bill equals the sum of the original one-off bills.

### 6. Household split

- **Flag:** add a `followsHouseholdDefault` flag to `bill.sharing`.
- **Changing the default:** when `settings.defaultJointSharePercent` changes, list the bills that follow it and offer to update them from an effective date. Past occurrences must not be rewritten.

### 7. Windfall rules on income streams

- **Rule:** an income stream can carry `{ threshold, toDebtPercent, debtAccountId, toSavingsPercent, savingsAccountId }`.
- **Behaviour:** on the occurrence date, generate planned transfers for the portion above the threshold.

### 8. Data health check (Settings → "Check my data")

Flag each of these, with a one-tap fix:

- bills, income or transfers pointing at closed accounts;
- zero-amount planned transactions;
- duplicate planned transactions (same date, account, payee and amount);
- `planned` items dated in the past;
- joint bills whose share differs from the household default.

## Out of scope

- Replacing the forecast engine, recurrence rules, storage or sync, or theming.
- Open Banking connections. File import is enough for now.

## Global acceptance checks

- [ ] Existing v2 backups import unchanged. Export followed by re-import is lossless. Bump `meta.version` only if a migration is needed, and migrate v2 automatically.
- [ ] For a backup that doesn't use the new features, forecast totals are identical before and after the change (snapshot test).
- [ ] All new UI works at phone width.

## Report back

For each item:

- what was built, and which files changed;
- schema changes and migration;
- tests added;
- anything skipped, and why.

Also report genuine gaps. For example: Monzo's auto-export covers the current and joint accounts only. Flex purchases need their own import source, or will stay as reconciliations.
