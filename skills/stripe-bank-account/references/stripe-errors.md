# Stripe Error Codes Reference

Read this file when you encounter a Stripe error not covered in the main SKILL.md.

## Card Errors (`type: card_error`)

| Code | User-Facing Message |
|------|-------------------|
| `card_declined` | The card was declined. Ask the customer to use a different card. |
| `expired_card` | The card has expired. Ask the customer for an updated card. |
| `incorrect_cvc` | The CVC code is incorrect. Ask the customer to re-enter. |
| `incorrect_number` | The card number is incorrect. |
| `insufficient_funds` | The card has insufficient funds. |
| `invalid_expiry_month` | The expiry month is invalid. |
| `invalid_expiry_year` | The expiry year is invalid. |
| `invalid_number` | The card number is not a valid credit card number. |
| `postal_code_invalid` | The ZIP/postal code failed validation. |
| `processing_error` | An error occurred while processing the card. Try again. |
| `stolen_card` | The card has been reported stolen. Do not retry. |
| `do_not_honor` | The card issuer has blocked this payment. |

## Invalid Request Errors (`type: invalid_request_error`)

| Code | Meaning |
|------|---------|
| `amount_too_large` | Amount exceeds the maximum allowed. |
| `amount_too_small` | Amount is below the minimum ($0.50 USD). |
| `currency_not_supported` | That currency isn't supported for this operation. |
| `missing` | A required parameter is missing. Check required fields. |
| `parameter_invalid_integer` | An amount or integer field has an invalid value. |
| `resource_missing` | The object (customer, card, account) doesn't exist. |
| `balance_insufficient` | Your Stripe balance doesn't have enough funds for this payout/transfer. |

## Authentication Errors (`type: authentication_error`)

| Code | Meaning |
|------|---------|
| (401) | The API key is invalid or expired. Ask the user to re-enter their key from the Stripe Dashboard. |

## Rate Limit Errors (`type: rate_limit_error`)

Wait at least 1 second, then retry. If retrying more than 3 times, inform the user and suggest trying again later.

## API Errors (`type: api_error`)

Stripe-side issue. Retry once after 2 seconds. If it persists, direct the user to status.stripe.com.
