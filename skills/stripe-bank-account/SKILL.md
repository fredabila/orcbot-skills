---
name: stripe-bank
description: >
  Manage a real Stripe account like a bank — check live balances, view transaction history,
  create charges, and send payouts/transfers — all via the Stripe API. Use this skill whenever
  the user wants to interact with their Stripe account, check how much money they have, see
  what they've earned, charge a customer, pay someone out, or move funds. Trigger on phrases
  like "check my Stripe balance", "charge a customer", "send a payout", "how much did I make",
  "view my transactions", "transfer funds", or any mention of Stripe and money in the same breath.
  Always use this skill before trying to call Stripe APIs from memory — it has the correct
  endpoints, error handling patterns, and output formats.
---

# Stripe Bank Skill

This skill lets an agent operate a Stripe account as a real bank: read live balances,
browse transaction history, create charges, and send payouts or transfers.

---

## Step 0: Get the API Key

**Always do this first.** If you don't already have the user's Stripe secret key in context:

> Ask the user:
> "Please provide your Stripe secret key to continue. It starts with `sk_test_` (test mode)
> or `sk_live_` (live mode). You can find it at dashboard.stripe.com → Developers → API keys."

Store it as `STRIPE_KEY` for use in all requests below.

⚠️ Never log, store, or display the full key back to the user. Confirm receipt with just
the first 12 characters: e.g. `sk_test_AbCd...`.

---

## Base Request Pattern

All Stripe API calls follow this pattern:

```
Method: GET or POST
Base URL: https://api.stripe.com/v1/
Auth: Bearer <STRIPE_KEY>  (Authorization header)
Content-Type: application/x-www-form-urlencoded  (for POST)
Amounts: always in cents (integer). $10.00 = 1000
```

---

## Operations

### 1. Check Balance

**Endpoint:** `GET /v1/balance`

Returns available and pending balances, broken down by currency.

**Display format:**
```
💰 Stripe Balance
─────────────────────────────
Available:  $1,250.00 USD
Pending:    $340.00 USD
─────────────────────────────
(available = can be paid out now)
(pending = in transit, not yet settled)
```

If multiple currencies exist, show each on its own line.

---

### 2. View Transaction History

**Endpoint:** `GET /v1/balance_transactions`

**Useful query params:**
| Param | Default | Notes |
|-------|---------|-------|
| `limit` | 10 | Max 100 |
| `type` | (all) | charge, payout, transfer, refund, etc. |
| `created[gte]` | — | Unix timestamp: start date |
| `created[lte]` | — | Unix timestamp: end date |

**Display format (table):**
```
DATE         TYPE        AMOUNT       FEE      NET       DESCRIPTION
Mar 9 2026   charge      +$50.00     -$1.75   +$48.25   Payment from customer
Mar 8 2026   payout      -$200.00    $0.00    -$200.00  Payout to bank
```

- Amounts come back in cents — always divide by 100 for display
- Positive = funds added to balance, negative = funds removed
- Show `description` or `source` ID if no description

---

### 3. Create a Charge

**Endpoint:** `POST /v1/charges`

Required fields:
```
amount=<cents>
currency=usd
source=<token_or_card_id>   OR   customer=<customer_id>
description=<optional text>
```

**Before charging:** confirm with the user:
> "You're about to charge [amount] to [source/customer]. Confirm? (yes/no)"

**On success**, display:
```
✅ Charge created
ID:          ch_XXXXXXXXXX
Amount:      $50.00 USD
Status:      succeeded
Description: [description]
```

**Common errors:**
| Code | Meaning | Action |
|------|---------|--------|
| `card_declined` | Card was declined | Ask user for different card |
| `insufficient_funds` | Card has no funds | Inform user |
| `invalid_request_error` | Bad params | Show error message from Stripe |

---

### 4. Send a Payout (to your own bank account)

**Endpoint:** `POST /v1/payouts`

Required fields:
```
amount=<cents>
currency=usd
```

Optional:
```
description=<text>
statement_descriptor=<text shown on bank statement, max 22 chars>
```

**Before paying out:** confirm with the user:
> "You're about to send $[amount] to your bank account on file. Confirm? (yes/no)"

**On success:**
```
✅ Payout initiated
ID:            po_XXXXXXXXXX
Amount:        $200.00 USD
Arrival:       [estimated_arrival_date]
Status:        pending
```

Payout statuses: `pending` → `in_transit` → `paid` (or `failed`)

**Note:** Requires sufficient available balance. If insufficient, show current balance and shortfall.

---

### 5. Send a Transfer (to a connected Stripe account)

**Endpoint:** `POST /v1/transfers`

Required fields:
```
amount=<cents>
currency=usd
destination=<connected_account_id>   (starts with acct_)
```

Optional:
```
description=<text>
```

**Before transferring:** confirm with the user:
> "Transfer $[amount] to Stripe account [destination]? (yes/no)"

**On success:**
```
✅ Transfer sent
ID:          tr_XXXXXXXXXX
Amount:      $100.00 USD
To:          acct_XXXXXXXXXX
```

---

## Error Handling

Always parse Stripe's error response body:

```json
{
  "error": {
    "type": "...",
    "code": "...",
    "message": "Human-readable message"
  }
}
```

Show the `message` field to the user in plain language. Don't expose raw HTTP codes alone.

| HTTP Status | Meaning |
|-------------|---------|
| 401 | Bad API key — ask user to re-enter |
| 402 | Payment failed — show error.message |
| 429 | Rate limited — wait and retry |
| 500 | Stripe server error — try again shortly |

---

## Safety Rules

1. **Always confirm** before any write operation (charge, payout, transfer).
2. **Never display the full API key** — only show the first 12 chars as confirmation.
3. **Live key warning:** If key starts with `sk_live_`, prepend a warning to every write operation:
   > ⚠️ You are in **live mode**. This will move real money.
4. **Amounts:** Always convert cents ↔ dollars correctly. $10 = 1000 cents. Never mix them up.
5. **Idempotency:** For POST requests, add an `Idempotency-Key` header (e.g. a UUID) to avoid
   duplicate charges if the user retries.

---

## Quick Reference

| User says | Operation | Endpoint |
|-----------|-----------|----------|
| "check my balance" | Check balance | GET /v1/balance |
| "show transactions / history" | List transactions | GET /v1/balance_transactions |
| "charge a customer" | Create charge | POST /v1/charges |
| "send a payout / pay myself" | Send payout | POST /v1/payouts |
| "transfer to account" | Send transfer | POST /v1/transfers |

---

## Reference Files

- `references/stripe-errors.md` — Full list of Stripe error codes and recommended responses
  (read this if you encounter an error not listed above)
