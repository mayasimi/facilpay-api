# Payment Refunds

## Overview

The refund system allows operators to issue full or partial refunds against completed payments. Refunds are tracked separately and update the payment status accordingly.

## Endpoint

```
POST /payments/:id/refund
```

## Request Body

```json
{
  "amount": 50.00,  // Optional: defaults to full refund
  "reason": "Customer requested refund"  // Optional
}
```

## Payment Statuses

- **PENDING**: Initial state, cannot be refunded
- **COMPLETED**: Can be refunded
- **PARTIALLY_REFUNDED**: Some amount has been refunded, more can be refunded
- **REFUNDED**: Fully refunded, no more refunds allowed
- **FAILED**: Cannot be refunded

## Examples

### Full Refund

```bash
curl -X POST http://localhost:3000/payments/123e4567-e89b-12d3-a456-426614174000/refund \
  -H "Content-Type: application/json" \
  -d '{}'
```

Response:
```json
{
  "payment": {
    "id": "123e4567-e89b-12d3-a456-426614174000",
    "amount": "100.00",
    "currency": "USD",
    "status": "REFUNDED",
    "refundedAmount": "100.00",
    "createdAt": "2026-01-26T10:00:00.000Z",
    "updatedAt": "2026-01-26T11:00:00.000Z"
  },
  "refund": {
    "id": "456e7890-e89b-12d3-a456-426614174000",
    "paymentId": "123e4567-e89b-12d3-a456-426614174000",
    "amount": "100.00",
    "reason": null,
    "createdAt": "2026-01-26T11:00:00.000Z"
  }
}
```

### Partial Refund

```bash
curl -X POST http://localhost:3000/payments/123e4567-e89b-12d3-a456-426614174000/refund \
  -H "Content-Type: application/json" \
  -d '{"amount": 50, "reason": "Partial refund for damaged item"}'
```

Response:
```json
{
  "payment": {
    "id": "123e4567-e89b-12d3-a456-426614174000",
    "amount": "100.00",
    "currency": "USD",
    "status": "PARTIALLY_REFUNDED",
    "refundedAmount": "50.00",
    "createdAt": "2026-01-26T10:00:00.000Z",
    "updatedAt": "2026-01-26T11:00:00.000Z"
  },
  "refund": {
    "id": "456e7890-e89b-12d3-a456-426614174000",
    "paymentId": "123e4567-e89b-12d3-a456-426614174000",
    "amount": "50.00",
    "reason": "Partial refund for damaged item",
    "createdAt": "2026-01-26T11:00:00.000Z"
  }
}
```

### Multiple Partial Refunds

```bash
# First partial refund
curl -X POST http://localhost:3000/payments/123e4567-e89b-12d3-a456-426614174000/refund \
  -d '{"amount": 30}'

# Second partial refund
curl -X POST http://localhost:3000/payments/123e4567-e89b-12d3-a456-426614174000/refund \
  -d '{"amount": 20}'

# Third refund completes the full refund
curl -X POST http://localhost:3000/payments/123e4567-e89b-12d3-a456-426614174000/refund \
  -d '{"amount": 50}'
# Status changes to REFUNDED
```

## Retrieving Refunds

Get payment details including all refunds:

```bash
curl http://localhost:3000/payments/123e4567-e89b-12d3-a456-426614174000
```

Response includes refunds array:
```json
{
  "id": "123e4567-e89b-12d3-a456-426614174000",
  "amount": "100.00",
  "currency": "USD",
  "status": "PARTIALLY_REFUNDED",
  "refundedAmount": "50.00",
  "refunds": [
    {
      "id": "refund-1",
      "amount": "30.00",
      "reason": "First refund",
      "createdAt": "2026-01-26T11:00:00.000Z"
    },
    {
      "id": "refund-2",
      "amount": "20.00",
      "reason": "Second refund",
      "createdAt": "2026-01-26T11:05:00.000Z"
    }
  ]
}
```

## Error Responses

### 404 - Payment Not Found
```json
{
  "statusCode": 404,
  "message": "Payment with ID 123e4567-e89b-12d3-a456-426614174000 not found",
  "error": "Not Found"
}
```

### 409 - Cannot Refund Pending Payment
```json
{
  "statusCode": 409,
  "message": "Cannot refund a payment that is still pending",
  "error": "Conflict"
}
```

### 409 - Already Fully Refunded
```json
{
  "statusCode": 409,
  "message": "Payment is already fully refunded",
  "error": "Conflict"
}
```

### 409 - Refund Exceeds Remaining Amount
```json
{
  "statusCode": 409,
  "message": "Refund amount 60 exceeds remaining refundable amount 40",
  "error": "Conflict"
}
```

## Business Rules

1. Only COMPLETED or PARTIALLY_REFUNDED payments can be refunded
2. Total refunded amount cannot exceed the original payment amount
3. Full refund transitions status to REFUNDED
4. Partial refund transitions status to PARTIALLY_REFUNDED
5. All refunds are tracked in the refunds table with timestamps
6. Refunds are processed within a database transaction for consistency
