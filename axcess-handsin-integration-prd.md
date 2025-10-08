# Product Requirements Document (PRD)
# Axcess - Hands In Integration

**Document Version:** 1.0  
**Date:** October 8, 2025  
**Owner:** Hands In Integration Team  
**Reviewers:** Engineering, Business Development, Customer Success  

---

## Executive Summary

This document outlines the technical and business requirements for integrating Axcess payment gateway with the Hands In payment orchestration platform. This is a **bidirectional integration** where:

1. **Hands In → Axcess**: Hands In will integrate Axcess as a payment gateway option for processing multi-card and group payments
2. **Axcess → Hands In**: Axcess will integrate Hands In's payment orchestration capabilities into their merchant offerings

### Key Integration Points
- **API Integration**: Multi-card and group payment session creation via Hands In APIs
- **SDK Integration**: JavaScript widgets for seamless checkout experiences
- **Webhook System**: Real-time event notifications using EventData interface
- **Payment Gateway Integration**: Axcess as a processing option within Hands In
- **Event Filtering**: Critical reconciliation logic to prevent double-processing
- **Split Payment Reconciliation**: Handling multi-card payment settlement flows

### Business Value Proposition
- **For Axcess Merchants**: Access to innovative multi-card and group payment capabilities
- **For Hands In**: Expanded payment processor network and merchant reach
- **For End Users**: Flexible payment options with reliable processing through Axcess gateway

---

## Project Overview

### Business Objectives

**Primary Objectives:**
1. Enable Axcess merchants to offer Hands In multi-card and group payment options
2. Integrate Axcess as a payment processor within Hands In's orchestration platform
3. Implement robust event filtering to prevent payment double-processing
4. Establish clear reconciliation processes for split payments

**Secondary Objectives:**
1. Provide seamless merchant onboarding experience
2. Enable real-time payment status updates and reconciliation
---

## Technical Requirements

### 1. API Integration

Axcess will implement Hands In API integration to create payment sessions:

#### Multi-Card Payment Creation
Enables customers to split a single purchase across multiple payment methods.

#### Group Payment Creation
Enables multiple users to collaboratively contribute to a single payment.

### 2. SDK Integration

Axcess merchants will embed Hands In widgets in their checkout flows:

#### Widget Types
- **Multi-Card Widget**: For individual split payment sessions
- **Group Payment Widget**: For collaborative payment sessions

#### Integration Approach
- JavaScript SDK loaded asynchronously
- Customizable styling to match merchant branding
- Mobile-responsive design
- PCI DSS compliant (no sensitive data touches merchant servers)

### 3. Webhook Integration

**Critical Requirement**: Axcess must implement webhook handlers for Hands In events to maintain payment state synchronization.

#### EventData Interface
All Hands In webhooks follow this standardized structure:

```typescript
interface EventData {
  id: string;                    // Unique event identifier
  eventType: EventType;          // Event type from enum
  merchantId: string;            // Axcess merchant identifier
  createdAt: Date;              // Event timestamp
  
  // Resource-specific IDs (populated based on event type)
  multiCardId?: string;
  groupPaymentId?: string;
  paymentId?: string;
  refundId?: string;
  customerId?: string;
  checkoutId?: string;
  itemId?: string;
  
  data?: object;                // Event-specific payload
  metaData?: MetaData;          // Additional context
}
```

#### Required Event Types to Handle

**Multi-Card Payment Events:**
- `MULTI_CARD_CREATED` - Payment session initiated
- `MULTI_CARD_UPDATED` - Payment session modified
- `MULTI_CARD_APPROVED` - Customer approved payment split
- `MULTI_CARD_COMPLETED` - All cards processed successfully
- `MULTI_CARD_CANCELLED` - Payment cancelled by customer
- `MULTI_CARD_EXPIRED` - Payment session timed out

**Group Payment Events:**
- `GROUP_PAYMENT_CREATED` - Group payment session created
- `GROUP_PAYMENT_UPDATED` - Group members or amounts changed
- `GROUP_PAYMENT_APPROVED` - Group owner approved payment
- `GROUP_PAYMENT_COMPLETED` - All group members paid successfully
- `GROUP_PAYMENT_CANCELLED` - Group payment cancelled
- `GROUP_PAYMENT_EXPIRED` - Group payment session expired

**Refund Events:**
- `REFUND_CREATED` - Refund initiated
- `REFUND_UPDATED` - Refund status changed
- `REFUND_COMPLETED` - Refund successfully processed
- `REFUND_REJECTED` - Refund rejected by processor
- `REFUND_FAILED` - Refund processing failed

### 4. Axcess Gateway Integration (Hands In Side)

Hands In will integrate Axcess as a payment processor option:

#### Gateway Configuration
- Merchant authentication via API credentials
- Support for test and production environments
- Configurable settlement options
- Multi-currency support (if supported by Axcess)

#### Payment Processing Flow
1. Customer selects payment method in Hands In widget
2. Hands In creates payment session with Axcess gateway
3. Axcess processes individual card transactions
4. Hands In receives payment confirmations
5. Hands In sends webhooks to merchant (Axcess)
6. Merchant updates order status

---

## Critical: Event Filtering and Reconciliation

### The Double-Processing Problem

When Axcess merchants use Hands In, payments will flow through **both** systems:
1. Hands In creates payment sessions
2. Axcess processes the actual transactions
3. **Both systems generate webhooks**

Without proper filtering, merchants would receive **duplicate webhook events** for the same payment, potentially causing:
- Double order fulfillment
- Incorrect inventory deductions
- Duplicate accounting entries
- Customer support issues

### Solution: Event Filtering Implementation

#### Axcess Webhook Handler Modification

Axcess must modify their existing webhook handlers to detect and skip Hands In-originated payments:

```javascript
// Axcess existing payment processor webhook handler
async function handleAxcessPaymentWebhook(webhookPayload) {
  // CRITICAL: Check if payment was created by Hands In
  if (webhookPayload.metadata?.createdBy === 'handsin') {
    // Skip processing - this will be handled by Hands In webhooks
    console.log('Skipping Hands In originated payment:', {
      paymentId: webhookPayload.id,
      orderId: webhookPayload.metadata?.orderId,
      source: 'handsin'
    });
    
    // Log for reconciliation tracking
    await logPaymentEvent({
      paymentId: webhookPayload.id,
      source: 'handsin',
      action: 'skipped_duplicate',
      timestamp: new Date()
    });
    
    return { status: 'skipped', reason: 'handsin_originated' };
  }
  
  // Process normal Axcess payments
  return await processAxcessPayment(webhookPayload);
}
```

#### Hands In Metadata Standard

All Hands In payments sent through Axcess will include:

```javascript
{
  metadata: {
    createdBy: "handsin",
    handsInMultiCardId: "{multi_card_id}",      // If multi-card payment
    handsInGroupPaymentId: "{group_payment_id}", // If group payment
    orderId: "{merchant_order_id}",
    customerId: "{merchant_customer_id}",
    sessionType: "multi-card" | "group-payment"
  }
}
```

#### Hands In Webhook Handler (Primary Event Source)

Merchants should process **only** Hands In webhooks for Hands In-originated payments:

```javascript
app.post('/webhooks/handsin', express.json(), async (req, res) => {
  try {
    const signature = req.headers['x-handsin-signature'];
    
    // Verify webhook signature
    if (!verifyHandsInWebhookSignature(req.body, signature)) {
      return res.status(401).send('Unauthorized');
    }
    
    const { eventType, data, multiCardId, groupPaymentId, metaData } = req.body;
    
    switch (eventType) {
      case 'MULTI_CARD_COMPLETED':
        await handleMultiCardCompleted({
          multiCardId,
          orderId: metaData?.orderId,
          totalAmount: data.totalAmount,
          cardPayments: data.cardPayments, // Array of individual card charges
          axcessTransactionIds: data.processorTransactionIds
        });
        break;
        
      case 'GROUP_PAYMENT_COMPLETED':
        await handleGroupPaymentCompleted({
          groupPaymentId,
          orderId: metaData?.orderId,
          totalAmount: data.totalAmount,
          memberPayments: data.memberPayments, // Array of member contributions
          axcessTransactionIds: data.processorTransactionIds
        });
        break;
        
      case 'MULTI_CARD_CANCELLED':
      case 'MULTI_CARD_EXPIRED':
      case 'GROUP_PAYMENT_CANCELLED':
      case 'GROUP_PAYMENT_EXPIRED':
        await handlePaymentFailed({
          paymentId: multiCardId || groupPaymentId,
          orderId: metaData?.orderId,
          reason: eventType
        });
        break;
        
      case 'REFUND_COMPLETED':
        await handlePaymentRefunded({
          refundId: data.refundId,
          originalPaymentId: data.originalPaymentId,
          orderId: metaData?.orderId,
          amount: data.amount,
          axcessRefundIds: data.processorRefundIds
        });
        break;
        
      default:
        console.log(`Unhandled webhook event: ${eventType}`);
    }
    
    res.status(200).send('OK');
  } catch (error) {
    console.error('Webhook processing error:', error);
    res.status(500).send('Internal Server Error');
  }
});
```

### Split Payment Reconciliation

#### Multi-Card Payment Structure

A single Hands In multi-card payment may split across multiple cards:

**Example Scenario:**
- Order Total: $150.00
- Card 1 (Visa): $100.00 → Axcess Transaction ID: `axc_tx_001`
- Card 2 (Mastercard): $50.00 → Axcess Transaction ID: `axc_tx_002`
- Hands In Multi-Card ID: `mc_abc123`

#### Reconciliation Requirements

**Daily Reconciliation Process:**

1. **Transaction Mapping**
   ```javascript
   {
     handsInMultiCardId: "mc_abc123",
     merchantOrderId: "order_456",
     totalAmount: 150.00,
     axcessTransactions: [
       { id: "axc_tx_001", amount: 100.00, card: "****1234", status: "settled" },
       { id: "axc_tx_002", amount: 50.00, card: "****5678", status: "settled" }
     ],
     reconciledAt: "2025-10-08T10:30:00Z",
     status: "fully_reconciled"
   }
   ```

2. **Settlement Tracking**
   - Track each Axcess transaction individually
   - Aggregate settlements by Hands In session ID
   - Flag partial settlements for review
   - Alert on reconciliation discrepancies

3. **Reporting Requirements**
   - Daily settlement reports showing split payment details
   - Exception reports for unreconciled payments
   - Chargeback tracking across split payments
   - Refund tracking with original payment mapping


## Recommended Implementation Plan

### Phase 1: Foundation Setup (Weeks 1-3)

**Week 1: Account Setup & Documentation**
- [ ] Hands In creates Axcess merchant account in sandbox
- [ ] Axcess receives API keys and credentials
- [ ] Both teams review integration documentation
- [ ] Establish webhook endpoint URLs (test and production)
- [ ] Set up communication channels (Slack, email)

**Week 2: Basic Integration Development**
- [ ] Axcess implements Hands In API client for payment session creation
- [ ] Hands In implements Axcess gateway adapter
- [ ] Configure sandbox environments
- [ ] Set up basic logging and monitoring

**Week 3: Event Filtering Implementation**
- [ ] Axcess modifies webhook handlers to detect Hands In metadata
- [ ] Implement event filtering logic and testing
- [ ] Set up reconciliation database tables
- [ ] Create reconciliation logging system

### Phase 2: Core Integration Development (Weeks 4-8)

**Week 4-5: Multi-Card Payment Integration**
- [ ] Implement multi-card payment creation via Hands In API
- [ ] Integrate multi-card widget in Axcess checkout flow
- [ ] Implement webhook handlers for multi-card events
- [ ] Test split payment processing through Axcess gateway

**Week 6-7: Group Payment Integration**
- [ ] Implement group payment creation via Hands In API
- [ ] Integrate group payment widget
- [ ] Implement webhook handlers for group payment events
- [ ] Test collaborative payment flows

**Week 8: Reconciliation System**
- [ ] Build automated reconciliation processes
- [ ] Implement daily settlement report generation
- [ ] Create discrepancy detection and alerting
- [ ] Build reconciliation dashboard for operations team
---

## Technical Specifications

### 1. Multi-Card Payment API

#### Endpoint
```
POST https://api.handsin.com/v1/multi-card
```

#### Headers
```
Content-Type: application/json
x-api-key: {axcess_api_key}
```

#### Request Body
```json
{
  "idempotencyKey": "axcess_{order_id}_{timestamp}",
  "amountMoney": {
    "amount": 15000,
    "currency": "USD"
  },
  "customerId": "axcess_customer_{customer_id}",
  "metaData": {
    "orderId": "{axcess_order_id}",
    "customerId": "{axcess_customer_id}",
    "createdBy": "axcess",
    "merchantName": "Axcess Merchant Name"
  },
  "pageOptions": {
    "returnUrl": "https://merchant.axcess.com/checkout/success",
    "cancelUrl": "https://merchant.axcess.com/checkout/cancel"
  },
  "gatewayConfig": {
    "preferredGateway": "axcess",
    "gatewayId": "{axcess_gateway_connection_id}"
  }
}
```

#### Response
```json
{
  "multiCardId": "mc_abc123xyz",
  "status": "created",
  "paymentUrl": "https://pay.handsin.com/mc/mc_abc123xyz",
  "expiresAt": "2025-10-08T12:00:00Z",
  "amountMoney": {
    "amount": 15000,
    "currency": "USD"
  },
  "metaData": {
    "orderId": "{axcess_order_id}",
    "customerId": "{axcess_customer_id}"
  }
}
```

### 2. Group Payment API

#### Endpoint
```
POST https://api.handsin.com/v1/group-payments
```

#### Headers
```
Content-Type: application/json
x-api-key: {axcess_api_key}
```

#### Request Body
```json
{
  "idempotencyKey": "axcess_{order_id}_{timestamp}",
  "amountMoney": {
    "amount": 20000,
    "currency": "USD"
  },
  "ownerId": "axcess_customer_{owner_id}",
  "metaData": {
    "orderId": "{axcess_order_id}",
    "ownerId": "{axcess_customer_id}",
    "createdBy": "axcess",
    "groupName": "Team Dinner Payment"
  },
  "inviteOptions": {
    "maxMembers": 10,
    "autoApprove": true
  },
  "gatewayConfig": {
    "preferredGateway": "axcess",
    "gatewayId": "{axcess_gateway_connection_id}"
  }
}
```

#### Response
```json
{
  "groupPaymentId": "gp_def456uvw",
  "status": "created",
  "paymentUrl": "https://pay.handsin.com/gp/gp_def456uvw",
  "shareUrl": "https://pay.handsin.com/join/abc123",
  "expiresAt": "2025-10-15T23:59:59Z",
  "amountMoney": {
    "amount": 20000,
    "currency": "USD"
  },
  "owner": {
    "id": "axcess_customer_{owner_id}",
    "email": "owner@example.com"
  },
  "inviteOptions": {
    "maxMembers": 10,
    "currentMembers": 1
  }
}
```

### 3. SDK Integration

#### Multi-Card Widget

```html
<!-- Include Hands In SDK -->
<script src="https://cdn.handsin.com/v1/handsin.js"></script>

<!-- Payment container -->
<div id="handsin-payment-container"></div>

<script>
  // Initialize and render multi-card widget
  window.handsin({
    merchantId: 'axcess_merchant_abc123',
    environment: 'production' // or 'sandbox' for testing
  }).renderMultiCard('handsin-payment-container', {
    live: true,
    merchantId: 'axcess_merchant_abc123',
    multiCardId: 'mc_abc123xyz', // From API response
    
    // Styling options
    theme: {
      primaryColor: '#0066cc',
      borderRadius: '8px',
      fontFamily: 'Inter, sans-serif'
    },
    
    // Event handlers
    onComplete: function(result) {
      console.log('Payment completed:', result);
      window.location.href = '/checkout/success?payment_id=' + result.multiCardId;
    },
    
    onError: function(error) {
      console.error('Payment error:', error);
      alert('Payment failed: ' + error.message);
    },
    
    onCancel: function() {
      console.log('Payment cancelled by user');
      window.location.href = '/checkout/cancel';
    }
  });
</script>
```

#### Group Payment Widget

```html
<script src="https://cdn.handsin.com/v1/handsin.js"></script>
<div id="handsin-group-payment-container"></div>

<script>
  window.handsin({
    merchantId: 'axcess_merchant_abc123',
    environment: 'production'
  }).renderGroupPayment('handsin-group-payment-container', {
    live: true,
    merchantId: 'axcess_merchant_abc123',
    groupPaymentId: 'gp_def456uvw',
    
    theme: {
      primaryColor: '#0066cc',
      borderRadius: '8px'
    },
    
    onComplete: function(result) {
      window.location.href = '/checkout/success?payment_id=' + result.groupPaymentId;
    },
    
    onMemberJoined: function(member) {
      console.log('New member joined:', member);
      updateGroupMembersList(member);
    },
    
    onError: function(error) {
      console.error('Payment error:', error);
    }
  });
</script>
```

### 4. Webhook Implementation

#### Webhook Endpoint Setup

```javascript
const express = require('express');
const crypto = require('crypto');

const app = express();

// Webhook signature verification
function verifyHandsInWebhookSignature(payload, signature) {
  const webhookSecret = process.env.HANDSIN_WEBHOOK_SECRET;
  const computedSignature = crypto
    .createHmac('sha256', webhookSecret)
    .update(JSON.stringify(payload))
    .digest('hex');
  
  return crypto.timingSafeEqual(
    Buffer.from(signature),
    Buffer.from(computedSignature)
  );
}

// Hands In webhook endpoint
app.post('/webhooks/handsin', express.json(), async (req, res) => {
  try {
    const signature = req.headers['x-handsin-signature'];
    
    if (!verifyHandsInWebhookSignature(req.body, signature)) {
      console.error('Invalid webhook signature');
      return res.status(401).send('Unauthorized');
    }
    
    const event = req.body; // EventData interface
    const { eventType, multiCardId, groupPaymentId, data, metaData, createdAt } = event;
    
    console.log('Received Hands In webhook:', {
      eventType,
      multiCardId,
      groupPaymentId,
      timestamp: createdAt
    });
    
    switch (eventType) {
      case 'MULTI_CARD_CREATED':
        await handleMultiCardCreated(event);
        break;
        
      case 'MULTI_CARD_APPROVED':
        await handleMultiCardApproved(event);
        break;
        
      case 'MULTI_CARD_COMPLETED':
        await handleMultiCardCompleted(event);
        break;
        
      case 'MULTI_CARD_CANCELLED':
      case 'MULTI_CARD_EXPIRED':
        await handleMultiCardFailed(event);
        break;
        
      case 'GROUP_PAYMENT_CREATED':
        await handleGroupPaymentCreated(event);
        break;
        
      case 'GROUP_PAYMENT_COMPLETED':
        await handleGroupPaymentCompleted(event);
        break;
        
      case 'GROUP_PAYMENT_CANCELLED':
      case 'GROUP_PAYMENT_EXPIRED':
        await handleGroupPaymentFailed(event);
        break;
        
      case 'REFUND_CREATED':
        await handleRefundCreated(event);
        break;
        
      case 'REFUND_COMPLETED':
        await handleRefundCompleted(event);
        break;
        
      case 'REFUND_FAILED':
      case 'REFUND_REJECTED':
        await handleRefundFailed(event);
        break;
        
      default:
        console.log(`Unhandled webhook event: ${eventType}`);
    }
    
    res.status(200).send('OK');
    
  } catch (error) {
    console.error('Webhook processing error:', error);
    res.status(500).send('Internal Server Error');
  }
});

// Event handlers
async function handleMultiCardCompleted(event) {
  const { multiCardId, data, metaData } = event;
  const orderId = metaData?.orderId;
  
  console.log('Multi-card payment completed:', {
    multiCardId,
    orderId,
    totalAmount: data.totalAmount,
    cardCount: data.cardPayments?.length
  });
  
  // Update order status in Axcess system
  await updateOrderStatus(orderId, 'paid');
  
  // Store reconciliation data
  await storeReconciliationRecord({
    handsInPaymentId: multiCardId,
    paymentType: 'multi-card',
    orderId: orderId,
    totalAmount: data.totalAmount,
    currency: data.currency,
    axcessTransactions: data.cardPayments.map(cp => ({
      transactionId: cp.axcessTransactionId,
      amount: cp.amount,
      cardLast4: cp.cardLast4,
      cardBrand: cp.cardBrand,
      status: 'settled'
    })),
    reconciliationStatus: 'complete'
  });
  
  // Fulfill order
  await fulfillOrder(orderId);
}

async function handleGroupPaymentCompleted(event) {
  const { groupPaymentId, data, metaData } = event;
  const orderId = metaData?.orderId;
  
  console.log('Group payment completed:', {
    groupPaymentId,
    orderId,
    totalAmount: data.totalAmount,
    memberCount: data.memberPayments?.length
  });
  
  await updateOrderStatus(orderId, 'paid');
  
  await storeReconciliationRecord({
    handsInPaymentId: groupPaymentId,
    paymentType: 'group-payment',
    orderId: orderId,
    totalAmount: data.totalAmount,
    currency: data.currency,
    axcessTransactions: data.memberPayments.map(mp => ({
      transactionId: mp.axcessTransactionId,
      amount: mp.amount,
      cardLast4: mp.cardLast4,
      cardBrand: mp.cardBrand,
      status: 'settled'
    })),
    reconciliationStatus: 'complete'
  });
  
  await fulfillOrder(orderId);
}

async function handleMultiCardFailed(event) {
  const { multiCardId, eventType, metaData } = event;
  const orderId = metaData?.orderId;
  
  console.log('Multi-card payment failed:', {
    multiCardId,
    orderId,
    reason: eventType
  });
  
  await updateOrderStatus(orderId, 'payment_failed');
  await notifyCustomer(orderId, 'payment_failed');
}

async function handleRefundCompleted(event) {
  const { data, metaData } = event;
  const orderId = metaData?.orderId;
  
  console.log('Refund completed:', {
    refundId: data.refundId,
    originalPaymentId: data.originalPaymentId,
    amount: data.amount
  });
  
  await updateOrderStatus(orderId, 'refunded');
  
  // Update reconciliation records
  await updateReconciliationForRefund({
    originalPaymentId: data.originalPaymentId,
    refundAmount: data.amount,
    axcessRefundIds: data.processorRefundIds
  });
}

app.listen(3000, () => {
  console.log('Webhook server running on port 3000');
});
```




## Appendix

### A. Glossary

- **Multi-Card Payment**: A single payment split across multiple payment methods
- **Group Payment**: A collaborative payment where multiple users contribute
- **Event Filtering**: Logic to prevent duplicate webhook processing
- **Reconciliation**: Process of matching split payments to settlement records
- **Gateway**: Payment processor that handles card transactions
- **Webhook**: HTTP callback for real-time event notifications

### B. API Rate Limits

- Payment creation: 100 requests/minute per merchant
- Payment status queries: 1000 requests/minute per merchant
- Webhook retries: 3 attempts with exponential backoff (1s, 5s, 25s)

### C. Support Escalation Matrix

| Severity | Response Time | Channels |
|----------|--------------|----------|
| P0 - Critical (Payment processing down) | 15 minutes | Phone, Slack, Email |
| P1 - High (Significant impact) | 1 hour | Slack, Email |
| P2 - Medium (Limited impact) | 4 hours | Email, Support Portal |
| P3 - Low (Questions/Enhancement) | 24 hours | Support Portal |

### D. Related Documentation

- Hands In API Documentation: https://docs.handsin.com/api
- Hands In SDK Documentation: https://docs.handsin.com/sdk
- Hands In Webhook Guide: https://docs.handsin.com/webhooks
- EventType Enum Reference: `/apps/old-api/src/services/merchants/webhooks/types/EventType.ts`
- EventData Interface Reference: `/apps/old-api/src/services/merchants/webhooks/types/EventData.ts`
- Axcess Gateway Integration Guide: [To be created]
- Reconciliation Best Practices: [To be created]

---

## Approval and Sign-off

| Role | Name | Signature | Date |
|------|------|-----------|------|
| Integration Lead | | | |
| Engineering Manager | | | |
| Product Manager | | | |
| Business Development | | | |

---

**Document History:**

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | October 8, 2025 | Hands In Integration Team | Initial PRD creation |

---

**Next Steps:**

1. Review and approval by all stakeholders
2. Technical architecture design session
3. Project kickoff meeting with Axcess team
4. Environment setup and credential exchange
5. Begin Phase 1 implementation

---

*This document is confidential and intended for internal use and authorized partners only.*
