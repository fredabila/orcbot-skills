---
name: bank-account
description: >
  Maintain and manage a virtual bank account for an agent or user. Use this skill whenever
  the user or agent needs to: track a balance, make deposits or withdrawals, transfer funds,
  view transaction history, check account status, or simulate any banking operation. Trigger
  this skill for any prompt involving money management, account balances, spending tracking,
  budgeting, financial transactions, or phrases like "add funds", "withdraw", "check my balance",
  "send money", "how much do I have", "pay for", or "debit/credit account". Use even if the
  user doesn't explicitly say "bank account" — intent to track or move money is enough.
---

# Bank Account Skill

This skill enables an agent to act as a fully functional virtual bank — maintaining account
state across turns, processing transactions, and producing clear financial summaries.

---

## Storage

All account state is persisted using `window.storage` (key-value, async).

### Key schema

| Key | Value | Notes |
|-----|-------|-------|
| `account:meta` | `{ id, owner, created_at, currency }` | Created once |
| `account:balance` | `number` | Current balance (float, 2dp) |
| `account:txns` | `Transaction[]` | Full history, append-only |
| `account:overdraft_limit` | `number` | Default 0 (no overdraft) |

### Transaction shape

```json
{
  "id": "txn_<timestamp>_<random4>",
  "type": "deposit" | "withdrawal" | "transfer_in" | "transfer_out" | "fee" | "adjustment",
  "amount": 50.00,
  "balance_after": 150.00,
  "description": "Human-readable note",
  "timestamp": "2026-03-09T14:00:00Z",
  "ref": "optional external reference"
}
```

---

## Operations

### 1. Open account

Called when no `account:meta` key exists yet.

```
- Generate id: "ACC" + 6 random digits
- Set owner from user input (default "Account Holder")
- Set currency (default "USD")
- Set balance to opening deposit (default 0)
- Write account:meta, account:balance, account:txns = []
- If opening deposit > 0, create an initial deposit transaction
```

**Output:** Confirmation card showing account number, owner, opening balance, currency.

---

### 2. Deposit

```
- Read current balance
- Add amount (must be > 0)
- Append transaction (type: "deposit")
- Write new balance + updated txns
```

**Output:** "Deposited $X. New balance: $Y."

---

### 3. Withdraw

```
- Read current balance + overdraft_limit
- Check: balance + overdraft_limit >= amount
  - If insufficient: return error, do NOT modify state
- Subtract amount
- Append transaction (type: "withdrawal")
- Write new balance + updated txns
```

**Output:** "Withdrew $X. New balance: $Y." or "Insufficient funds. Balance: $Y, requested: $X."

---

### 4. Transfer

Transfers require a destination label (can be freeform text like "Alice" or "Savings").

```
- Treat as a withdrawal from this account
- Append transaction (type: "transfer_out", description: "Transfer to <dest>")
- If transferring between two tracked accounts in the same artifact, also append transfer_in to the other account
```

**Output:** "Transferred $X to <dest>. New balance: $Y."

---

### 5. Check balance

```
- Read account:balance
- Read last 3 transactions for context
```

**Output:** Balance summary card (see Display section).

---

### 6. Transaction history

Parameters: optional `limit` (default 10), optional `type` filter.

```
- Read account:txns
- Apply filters
- Reverse sort (newest first)
- Paginate
```

**Output:** Formatted table of transactions.

---

### 7. Set overdraft limit

```
- Write account:overdraft_limit
```

**Output:** Confirmation.

---

### 8. Close / reset account

```
- Confirm with user first (destructive)
- Delete all account:* keys
```

---

## Display Conventions

### Balance card (inline text)
```
┌─────────────────────────────┐
│  Account: ACC123456          │
│  Owner:   Frederick Abila    │
│  Balance: $1,250.00 USD      │
│  Updated: Mar 9, 2026        │
└─────────────────────────────┘
```

### Transaction row
```
[DATE]       [TYPE]        [+/- AMOUNT]    [BALANCE AFTER]    [DESCRIPTION]
Mar 9 14:00  Deposit       +$500.00        $1,250.00          Paycheck
Mar 8 09:30  Withdrawal    -$45.00         $750.00            Coffee & lunch
```

Always show amounts with 2 decimal places and currency symbol.
Use **+** for inflows (deposit, transfer_in) and **−** for outflows.
Show balance_after in every row so the user can trace state.

---

## Error Handling

| Situation | Response |
|-----------|----------|
| Insufficient funds | Block transaction, show current balance and shortfall |
| Negative amount | Reject, ask for positive value |
| Account doesn't exist | Prompt to open account first |
| Amount is not a number | Ask for clarification |
| Duplicate transaction detected | Warn user, ask to confirm |

---

## Multi-account Support

If the agent needs to manage more than one account (e.g., "checking" and "savings"):

- Use prefixed keys: `account:<name>:balance`, `account:<name>:txns`, etc.
- List all accounts via `window.storage.list("account:")` and filter for `:meta` keys
- Clearly label which account each operation affects

---

## Implementation Notes for Artifact Builders

When building an Artifact (React/HTML) using this skill:

1. **Load state on mount** — read `account:balance` and `account:txns` in a `useEffect`.
2. **Optimistic UI** — update displayed balance immediately, then persist async.
3. **Error display** — show errors inline near the action, not as alerts.
4. **Amounts** — always store as floats rounded to 2dp: `Math.round(x * 100) / 100`.
5. **IDs** — `"txn_" + Date.now() + "_" + Math.random().toString(36).slice(2,6)`.
6. **Currency formatting** — use `Intl.NumberFormat('en-US', { style: 'currency', currency: 'USD' })`.

---

## Sample Interaction Flows

### Opening a new account
> "Create a bank account for me with $500"
→ Open account, deposit $500, show balance card.

### Day-to-day use
> "I just spent $42 on groceries"
→ Withdraw $42, description "Groceries", show new balance.

> "I got paid $1800"
→ Deposit $1800, description "Paycheck", show new balance.

> "What's my balance?"
→ Show balance card + last 3 transactions.

> "Show me everything I spent last week"
→ Filter txns by type=withdrawal and date range, display table.

### Transfers
> "Move $200 from checking to savings"
→ transfer_out from checking, transfer_in to savings.

---

## Quick Reference

| User says | Operation |
|-----------|-----------|
| "deposit / add / received / got paid" | deposit |
| "withdraw / spent / paid / bought" | withdrawal |
| "transfer / move / send" | transfer |
| "balance / how much / what's left" | check balance |
| "history / transactions / statement" | transaction history |
| "overdraft / allow negative" | set overdraft limit |
| "close / reset / start over" | close account |
