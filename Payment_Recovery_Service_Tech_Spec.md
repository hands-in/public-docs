# Product Specification: Payment Recovery Service
## Hands In Payment Recovery API & SDK - Integration Guide

**Document Version:** 1.0  
**Date:** October 6, 2025  
**Owner:** Hands In Product & Business Development Team  
**Status:** Draft - Design Phase  
**Audience:** External - Merchants, Integration Partners, Sales Team

---

## Executive Summary

**This document is designed for merchants and partners who want to integrate the Hands In Payment Recovery Service into their systems.** It provides everything you need to understand the product capabilities, integration requirements, and business value.

The Payment Recovery Service is a payment recovery API and SDK that enables merchants to automatically recover failed payments through intelligent routing and payment method optimization. When a customer's payment fails on a merchant's website, the merchant can hand off the recovery process to Hands In, which will use machine learning, alternative payment routing, and user-friendly recovery flows to maximize successful payment capture.


### Key Value Propositions
- **Automated Recovery**: Intelligent routing to alternative processors or payment methods
- **Higher Conversion**: Recover 15-30% of failed payments that would otherwise be lost
- **Seamless UX**: Customer-friendly recovery flows with minimal friction
- **Real-time Intelligence**: ML-powered decision engine for optimal routing strategies
- **Zero Integration Complexity**: Simple API call or SDK integration to hand off failed payments

---

## 1. System Architecture

### 1.1 High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         Merchant System                          │
│  ┌──────────────┐         ┌─────────────────────────────────┐  │
│  │   Checkout   │────────▶│  Payment Processor (Primary)     │  │
│  │    Flow      │         │  Returns: DECLINED/FAILED        │  │
│  └──────────────┘         └─────────────────────────────────┘  │
│         │                                                         │
│         │ Payment Failed                                          │
│         ▼                                                         │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │        Hands In Payment Recovery SDK/API                  │  │
│  │  sendFailedPayment(customer, payment, failureResponse)    │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│               Hands In Payment Recovery Service                  │
│                                                                   │
│  ┌────────────────────────────────────────────────────────┐    │
│  │              Recovery Intelligence Engine               │    │
│  │  • Failure Analysis                                     │    │
│  │  • Customer Profile Analysis                            │    │
│  │  • Processor Success Rate Analysis                      │    │
│  │  • ML-based Routing Decisions                           │    │
│  └────────────────────────────────────────────────────────┘    │
│                         │                                        │
│                         ▼                                        │
│  ┌────────────────────────────────────────────────────────┐    │
│  │              Recovery Strategy Selector                 │    │
│  │                                                          │    │
│  │  1. Alternative Processor Routing                       │    │
│  │  2. Alternative Payment Method                          │    │
│  │  3. Delayed Retry (Time-based)                          │    │
│  │  4. Installment/Split Payment Offer                     │    │
│  │  5. Manual Review Queue                                 │    │
│  └────────────────────────────────────────────────────────┘    │
│                         │                                        │
│                         ▼                                        │
│  ┌────────────────────────────────────────────────────────┐    │
│  │           Recovery Execution Engine                     │    │
│  │  • Route to alternative processors                      │    │
│  │  • Generate customer recovery UI                        │    │
│  │  • Schedule retries                                     │    │
│  │  • Send notifications                                   │    │
│  └────────────────────────────────────────────────────────┘    │
│                         │                                        │
│                         ▼                                        │
│  ┌────────────────────────────────────────────────────────┐    │
│  │         Alternative Payment Processors Pool             │    │
│  │  • Processor A, B, C... (Stripe, Square, etc.)         │    │
│  │  • Geographic optimization                              │    │
│  │  • Industry-specific processors                         │    │
│  └────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
                    ┌──────────────────┐
                    │   Customer       │
                    │  Recovery Flow   │
                    │  (Email/SMS/UI)  │
                    └──────────────────┘
```

### 1.2 Core Components

#### 1.2.1 API Gateway
- RESTful API for failed payment submission
- Authentication and rate limiting
- Request validation and sanitization
- Real-time response with recovery strategy

#### 1.2.2 Recovery Intelligence Engine
- Failure code analysis and classification
- Customer payment history analysis
- Processor success rate tracking
- ML model for optimal routing decisions
- Risk assessment and fraud detection

#### 1.2.3 Recovery Strategy Selector
- Strategy prioritization based on failure type
- Multi-strategy orchestration
- A/B testing framework for strategy effectiveness
- Merchant-specific rule configuration

#### 1.2.4 Recovery Execution Engine
- Alternative processor integration
- Customer notification system
- Recovery UI generation
- Retry scheduling and management
- Payment state management

#### 1.2.5 SDK Components
- JavaScript SDK for web integration
- Mobile SDKs (iOS/Android) for app integration
- Server-side SDK for backend integration
- Widget library for recovery flows

---

## 2. API Specifications

### 2.1 Failed Payment Submission API

#### Endpoint
```
POST https://api.handsin.com/v1/payment-recovery
```

#### Authentication
```
x-api-key: {merchant_api_key}
Content-Type: application/json
```

#### Request Schema
```typescript
interface PaymentRecoveryRequest {
  // Unique identifier for this recovery attempt
  idempotencyKey: string;
  
  // Merchant information
  merchantId: string;
  merchantOrderId: string;
  
  // Customer information
  customer: {
    id: string;
    email: string;
    phone?: string;
    name: {
      firstName: string;
      lastName: string;
    };
    billingAddress?: Address;
    shippingAddress?: Address;
    // Customer history with merchant
    customerMetrics?: {
      totalOrders: number;
      totalSpent: number;
      accountAge: number; // days
      previousFailedPayments?: number;
    };
  };
  
  // Payment details
  payment: {
    amount: {
      value: number; // in cents
      currency: string;
    };
    // Original payment method that failed
    paymentMethod: {
      type: 'card' | 'bank_account' | 'digital_wallet';
      card?: {
        last4: string;
        brand: string; // visa, mastercard, amex, etc.
        expiryMonth: string;
        expiryYear: string;
        bin?: string; // First 6 digits for BIN analysis
      };
      bankAccount?: {
        last4: string;
        accountType: 'checking' | 'savings';
        routingNumber?: string;
      };
      digitalWallet?: {
        provider: 'apple_pay' | 'google_pay' | 'paypal';
        accountIdentifier?: string;
      };
    };
    // Original processor used
    processor: {
      name: string;
      processorTransactionId?: string;
    };
  };
  
  // Failure information
  failure: {
    timestamp: string; // ISO 8601
    code: string; // Processor-specific decline code
    message: string; // Processor decline message
    category?: 'insufficient_funds' | 'card_declined' | 'fraud_suspected' | 
               'expired_card' | 'invalid_card' | 'do_not_honor' | 
               'processing_error' | 'authentication_failed' | 'other';
    // Raw response from processor
    rawResponse?: object;
    // Number of retry attempts already made by merchant
    previousAttempts?: number;
  };
  
  // Recovery options and preferences
  recoveryOptions?: {
    // Enable/disable specific recovery strategies
    allowedStrategies?: Array<
      'alternative_processor' | 'alternative_payment_method' |  'split_payments' |
      'delayed_retry' | 'installments' | 'customer_contact'
    >;
    // Merchant-specific recovery flow customization
    customization?: {
      brandColor?: string;
      logoUrl?: string;
      returnUrl?: string;
      cancelUrl?: string;
    };
    // Notification preferences
    notifications?: {
      email?: boolean;
      sms?: boolean;
      push?: boolean;
    };
    // Maximum time to attempt recovery (in hours)
    recoveryWindow?: number; // default: 72 hours
  };
  
  // Additional context
  metadata?: {
    orderItems?: Array<{
      name: string;
      quantity: number;
      price: number;
    }>;
    isSubscription?: boolean;
    subscriptionPeriod?: string;
    urgency?: 'high' | 'medium' | 'low';
    [key: string]: any;
  };
}

interface Address {
  line1: string;
  line2?: string;
  city: string;
  state?: string;
  postalCode: string;
  country: string; // ISO 3166-1 alpha-2
}
```

#### Response Schema
```typescript
interface PaymentRecoveryResponse {
  // Recovery session ID
  recoveryId: string;
  
  // Current status
  status: 'analyzing' | 'recovery_initiated' | 'retry_scheduled' | 
          'customer_action_required' | 'not_recoverable' | 'recovered';
  
  // Recovery strategy selected
  strategy: {
    primary: RecoveryStrategy;
    fallback?: RecoveryStrategy[];
    confidence: number; // 0-1, likelihood of success
    estimatedTimeToRecovery?: number; // minutes
  };
  
  // Immediate actions
  actions: {
    // URL for customer to complete recovery
    recoveryUrl?: string;
    
    // If immediate retry on alternative processor
    immediateRetry?: {
      status: 'success' | 'failed' | 'pending';
      transactionId?: string;
      processor?: string;
    };
    
    // Customer notification sent
    notificationSent?: {
      email?: boolean;
      sms?: boolean;
    };
  };
  
  // Recovery timeline
  timeline?: {
    createdAt: string;
    estimatedCompletionAt?: string;
    nextAttemptAt?: string;
  };
  
  // Webhook configuration
  webhooks: {
    url: string;
    events: string[];
  };
  
  metadata?: object;
}

type RecoveryStrategy = 
  | { type: 'alternative_processor'; processor: string; }
  | { type: 'alternative_payment_method'; methods: string[]; }
  | { type: 'delayed_retry'; retryAt: string; }
  | { type: 'installments'; plan: InstallmentPlan; }
  | { type: 'customer_contact'; channel: string; }
  | { type: 'not_recoverable'; reason: string; };

interface InstallmentPlan {
  numberOfPayments: number;
  paymentAmount: number;
  frequency: 'weekly' | 'biweekly' | 'monthly';
}
```

#### Example Request
```json
{
  "idempotencyKey": "merchant_order_12345_recovery_1",
  "merchantId": "merch_abc123",
  "merchantOrderId": "order_12345",
  "customer": {
    "id": "cust_xyz789",
    "email": "customer@example.com",
    "phone": "+1234567890",
    "name": {
      "firstName": "Jane",
      "lastName": "Smith"
    },
    "billingAddress": {
      "line1": "123 Main St",
      "city": "San Francisco",
      "state": "CA",
      "postalCode": "94102",
      "country": "US"
    },
    "customerMetrics": {
      "totalOrders": 5,
      "totalSpent": 50000,
      "accountAge": 365,
      "previousFailedPayments": 0
    }
  },
  "payment": {
    "amount": {
      "value": 15000,
      "currency": "USD"
    },
    "paymentMethod": {
      "type": "card",
      "card": {
        "last4": "4242",
        "brand": "visa",
        "expiryMonth": "12",
        "expiryYear": "2026",
        "bin": "424242"
      }
    },
    "processor": {
      "name": "stripe",
      "processorTransactionId": "pi_1234567890"
    }
  },
  "failure": {
    "timestamp": "2025-10-06T14:30:00Z",
    "code": "insufficient_funds",
    "message": "Your card has insufficient funds.",
    "category": "insufficient_funds",
    "previousAttempts": 0
  },
  "recoveryOptions": {
    "allowedStrategies": [
      "alternative_processor",
      "alternative_payment_method",
      "delayed_retry",
      "installments"
    ],
    "customization": {
      "brandColor": "#FF5722",
      "logoUrl": "https://merchant.com/logo.png",
      "returnUrl": "https://merchant.com/checkout/success",
      "cancelUrl": "https://merchant.com/checkout/cancel"
    },
    "notifications": {
      "email": true,
      "sms": true
    },
    "recoveryWindow": 72
  },
  "metadata": {
    "orderItems": [
      {
        "name": "Premium Subscription",
        "quantity": 1,
        "price": 15000
      }
    ],
    "isSubscription": true,
    "subscriptionPeriod": "monthly",
    "urgency": "high"
  }
}
```

#### Example Response
```json
{
  "recoveryId": "rec_9876543210",
  "status": "recovery_initiated",
  "strategy": {
    "primary": {
      "type": "delayed_retry",
      "retryAt": "2025-10-09T14:30:00Z"
    },
    "fallback": [
      {
        "type": "alternative_payment_method",
        "methods": ["bank_transfer", "digital_wallet"]
      },
      {
        "type": "installments",
        "plan": {
          "numberOfPayments": 3,
          "paymentAmount": 5000,
          "frequency": "monthly"
        }
      }
    ],
    "confidence": 0.75,
    "estimatedTimeToRecovery": 4320
  },
  "actions": {
    "recoveryUrl": "https://pay.handsin.com/recovery/rec_9876543210",
    "notificationSent": {
      "email": true,
      "sms": true
    }
  },
  "timeline": {
    "createdAt": "2025-10-06T14:30:05Z",
    "estimatedCompletionAt": "2025-10-09T14:30:00Z",
    "nextAttemptAt": "2025-10-09T14:30:00Z"
  },
  "webhooks": {
    "url": "https://merchant.com/webhooks/handsin",
    "events": [
      "RECOVERY_INITIATED",
      "RECOVERY_COMPLETED",
      "RECOVERY_FAILED",
      "CUSTOMER_ACTION_REQUIRED"
    ]
  }
}
```

### 2.2 Recovery Status API

#### Endpoint
```
GET https://api.handsin.com/v1/payment-recovery/{recoveryId}
```

#### Response Schema
```typescript
interface RecoveryStatusResponse {
  recoveryId: string;
  merchantId: string;
  merchantOrderId: string;
  status: RecoveryStatus;
  currentStrategy?: RecoveryStrategy;
  attempts: RecoveryAttempt[];
  customer: {
    id: string;
    email: string;
    lastInteraction?: string;
  };
  payment: {
    amount: {
      value: number;
      currency: string;
    };
    recoveredAmount?: {
      value: number;
      currency: string;
    };
  };
  timeline: {
    createdAt: string;
    lastUpdatedAt: string;
    completedAt?: string;
    expiresAt?: string;
  };
  result?: {
    success: boolean;
    finalStrategy?: string;
    transactionId?: string;
    processor?: string;
    completedAt?: string;
  };
}

type RecoveryStatus = 
  | 'analyzing'
  | 'recovery_initiated'
  | 'retry_scheduled'
  | 'customer_action_required'
  | 'in_progress'
  | 'recovered'
  | 'partially_recovered'
  | 'not_recoverable'
  | 'expired'
  | 'cancelled';

interface RecoveryAttempt {
  attemptId: string;
  strategy: RecoveryStrategy;
  status: 'pending' | 'processing' | 'success' | 'failed';
  timestamp: string;
  processor?: string;
  failureReason?: string;
  transactionId?: string;
}
```

### 2.3 Cancel Recovery API

#### Endpoint
```
POST https://api.handsin.com/v1/payment-recovery/{recoveryId}/cancel
```

#### Request Schema
```typescript
interface CancelRecoveryRequest {
  reason?: string;
  notifyCustomer?: boolean;
}
```

---

## 3. SDK Specifications

### 3.1 JavaScript SDK

#### Installation
```bash
npm install @handsin/payment-recovery
```

#### Initialization
```javascript
import { PaymentRecovery } from '@handsin/payment-recovery';

const recoveryClient = new PaymentRecovery({
  apiKey: 'your_api_key',
  environment: 'production', // or 'sandbox'
  merchantId: 'merch_abc123'
});
```

#### Core Methods

##### 3.1.1 Submit Failed Payment
```javascript
async submitFailedPayment(request: PaymentRecoveryRequest): Promise<PaymentRecoveryResponse>

// Example usage
try {
  const response = await recoveryClient.submitFailedPayment({
    idempotencyKey: `order_${orderId}_recovery_${Date.now()}`,
    merchantOrderId: orderId,
    customer: customerData,
    payment: paymentData,
    failure: failureData,
    recoveryOptions: {
      allowedStrategies: ['alternative_processor', 'delayed_retry'],
      customization: {
        brandColor: '#FF5722',
        logoUrl: 'https://example.com/logo.png'
      }
    }
  });
  
  console.log('Recovery ID:', response.recoveryId);
  console.log('Recovery URL:', response.actions.recoveryUrl);
  
  // Redirect customer to recovery flow if needed
  if (response.status === 'customer_action_required') {
    window.location.href = response.actions.recoveryUrl;
  }
} catch (error) {
  console.error('Recovery submission failed:', error);
}
```

##### 3.1.2 Get Recovery Status
```javascript
async getRecoveryStatus(recoveryId: string): Promise<RecoveryStatusResponse>

// Example usage
const status = await recoveryClient.getRecoveryStatus('rec_9876543210');
console.log('Current status:', status.status);
console.log('Attempts:', status.attempts.length);
```

##### 3.1.3 Render Recovery Widget
```javascript
renderRecoveryWidget(containerId: string, recoveryId: string, options?: WidgetOptions): void

// Example usage
recoveryClient.renderRecoveryWidget('recovery-container', 'rec_9876543210', {
  onComplete: (result) => {
    console.log('Payment recovered!', result);
    window.location.href = '/success';
  },
  onCancel: () => {
    console.log('Customer cancelled recovery');
    window.location.href = '/checkout';
  },
  theme: {
    primaryColor: '#FF5722',
    borderRadius: '8px'
  }
});
```

##### 3.1.4 Handle Recovery Events
```javascript
onRecoveryEvent(recoveryId: string, callback: (event: RecoveryEvent) => void): void

// Example usage
recoveryClient.onRecoveryEvent('rec_9876543210', (event) => {
  switch (event.type) {
    case 'recovery_completed':
      console.log('Payment recovered successfully!');
      updateOrderStatus(event.data.transactionId);
      break;
    case 'recovery_failed':
      console.log('Recovery failed:', event.data.reason);
      notifySupport(event.data);
      break;
    case 'customer_action_required':
      console.log('Customer needs to take action');
      sendNotification(event.data);
      break;
  }
});
```

### 3.2 Server-Side SDK (Node.js)

#### Installation
```bash
npm install @handsin/payment-recovery-node
```

#### Usage Example
```javascript
const { PaymentRecoveryClient } = require('@handsin/payment-recovery-node');

const client = new PaymentRecoveryClient({
  apiKey: process.env.HANDSIN_API_KEY,
  webhookSecret: process.env.HANDSIN_WEBHOOK_SECRET
});

// Submit failed payment
app.post('/payment/failed', async (req, res) => {
  try {
    const recovery = await client.submitFailedPayment({
      idempotencyKey: `order_${req.body.orderId}_${Date.now()}`,
      merchantOrderId: req.body.orderId,
      customer: req.body.customer,
      payment: req.body.payment,
      failure: req.body.failure
    });
    
    res.json({
      success: true,
      recoveryId: recovery.recoveryId,
      recoveryUrl: recovery.actions.recoveryUrl
    });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Handle webhooks
app.post('/webhooks/handsin/recovery', express.raw({ type: 'application/json' }), (req, res) => {
  const signature = req.headers['x-handsin-signature'];
  
  try {
    const event = client.verifyWebhook(req.body, signature);
    
    switch (event.type) {
      case 'RECOVERY_COMPLETED':
        handleRecoveryCompleted(event.data);
        break;
      case 'RECOVERY_FAILED':
        handleRecoveryFailed(event.data);
        break;
    }
    
    res.status(200).send('OK');
  } catch (error) {
    res.status(400).send('Invalid signature');
  }
});
```

---

## 4. Recovery Intelligence Engine

### 4.1 Failure Analysis

#### 4.1.1 Failure Code Classification
```typescript
enum FailureCategory {
  INSUFFICIENT_FUNDS = 'insufficient_funds',
  CARD_DECLINED = 'card_declined',
  FRAUD_SUSPECTED = 'fraud_suspected',
  EXPIRED_CARD = 'expired_card',
  INVALID_CARD = 'invalid_card',
  DO_NOT_HONOR = 'do_not_honor',
  PROCESSING_ERROR = 'processing_error',
  AUTHENTICATION_FAILED = 'authentication_failed',
  OTHER = 'other'
}

interface FailureAnalysis {
  category: FailureCategory;
  recoverability: 'high' | 'medium' | 'low' | 'none';
  suggestedStrategies: RecoveryStrategy[];
  confidence: number;
  reasoning: string;
}
```

#### 4.1.2 Recoverability Matrix Example (real recovery intelligence is dynamic)

| Failure Category | Recoverability | Primary Strategy | Secondary Strategy |
|-----------------|----------------|------------------|-------------------|
| Insufficient Funds | High (70-80%) | Split payments | Installments |
| Card Declined | Medium (40-60%) | Alternative Processor | Alternative Payment Method |
| Expired Card | High (60-70%) | Alternative Payment Method | Customer Contact |
| Invalid Card | Low (20-30%) | Alternative Payment Method | Customer Contact |
| Fraud Suspected | Low (10-20%) | Manual Review | Customer Contact |
| Processing Error | High (80-90%) | Alternative Processor | Immediate Retry |
| Authentication Failed | Medium (50-60%) | Customer Contact | Alternative Payment Method |
| Do Not Honor | Medium (30-50%) | Alternative Processor | Delayed Retry |

### 4.2 ML-Based Routing Decision

The Recovery Intelligence Engine uses machine learning to predict the optimal recovery strategy and processor routing. The system analyzes multiple factors to determine the best approach for each failed payment.

#### 4.2.1 How It Works (Merchant Perspective)

When you submit a failed payment through our API, our ML system automatically:

1. **Analyzes the failure** - Examines the decline code, processor response, and failure category
2. **Evaluates customer context** - Reviews payment history, success rate, and customer value
3. **Checks historical performance** - Looks at similar transactions and processor success rates
4. **Predicts optimal strategy** - Determines which recovery approach has the highest success probability
5. **Returns recommendation** - Provides you with the best strategy and confidence score

The system considers 100+ factors including:
- **Failure characteristics**: Category, processor, decline code
- **Customer characteristics**: LTV, payment history, success rate
- **Payment characteristics**: Amount, method, card BIN reputation
- **Contextual factors**: Time, location, merchant category
- **Historical performance**: Processor and strategy success rates

#### 4.2.2 What You Receive

For each submitted failed payment, you receive:

1. **Recommended Strategy** - The best recovery approach (e.g., "Alternative Processor", "Delayed Retry", "Installment Plan")
2. **Confidence Score** - Probability of success (0-100%)
3. **Processor Recommendation** - If applicable, which processor to route to
4. **Optimal Timing** - When to attempt recovery (immediate, delayed by hours/days)
5. **Expected Success Rate** - Based on similar historical transactions
6. **Alternative Strategies** - Ranked list of backup options if primary strategy fails

**Example Response:**
```json
{
  "recommendedStrategy": {
    "type": "alternative_processor",
    "processor": "stripe",
    "confidence": 0.78,
    "expectedSuccessRate": 0.65
  },
  "timing": {
    "recommended": "immediate",
    "alternativeWindows": ["24h", "72h"]
  },
  "alternatives": [
    {
      "type": "delayed_retry",
      "delay": "24h",
      "confidence": 0.62
    },
    {
      "type": "installment_plan",
      "confidence": 0.55
    }
  ]
}
```

---

## 5. Customer-Facing Recovery Experience

### 5.1 Recovery Page Flow
```
1. Landing Page
   ├─ Show order summary
   ├─ Explain payment issue
   └─ Display recovery options

2. Option Selection
   ├─ Retry with same method
   │  └─ Confirm retry → Process
   │
   ├─ Alternative payment method
   │  ├─ Select method (card/bank/wallet)
   │  ├─ Enter payment details
   │  └─ Submit → Process
   │
   └─ Installment plan
      ├─ Select plan
      ├─ Review schedule
      └─ Confirm → Setup recurring

3. Processing
   ├─ Show loading state
   └─ Real-time status updates

4. Completion
   ├─ Success: Show confirmation
   │  ├─ Transaction details
   │  ├─ Receipt email sent
   │  └─ Redirect to merchant
   │
   └─ Failure: Show error
      ├─ Explain what happened
      ├─ Offer alternative options
      └─ Contact support link
```

---

## 7. Webhook Events

### 7.1 Event Types

```typescript
enum RecoveryEventType {
  RECOVERY_INITIATED = 'RECOVERY_INITIATED',
  RECOVERY_STRATEGY_SELECTED = 'RECOVERY_STRATEGY_SELECTED',
  RECOVERY_ATTEMPT_STARTED = 'RECOVERY_ATTEMPT_STARTED',
  RECOVERY_ATTEMPT_FAILED = 'RECOVERY_ATTEMPT_FAILED',
  CUSTOMER_ACTION_REQUIRED = 'CUSTOMER_ACTION_REQUIRED',
  CUSTOMER_VIEWED_RECOVERY_PAGE = 'CUSTOMER_VIEWED_RECOVERY_PAGE',
  CUSTOMER_UPDATED_PAYMENT_METHOD = 'CUSTOMER_UPDATED_PAYMENT_METHOD',
  RECOVERY_COMPLETED = 'RECOVERY_COMPLETED',
  RECOVERY_FAILED = 'RECOVERY_FAILED',
  RECOVERY_EXPIRED = 'RECOVERY_EXPIRED',
  RECOVERY_CANCELLED = 'RECOVERY_CANCELLED'
}
```

### 7.2 Webhook Payload Structure

```typescript
interface RecoveryWebhookEvent {
  id: string;
  eventType: RecoveryEventType;
  merchantId: string;
  createdAt: Date;
  recoveryId: string;
  
  data: {
    recoveryId: string;
    merchantOrderId: string;
    
    // Strategy information
    strategy?: {
      type: string;
      details: object;
    };
    
    // Attempt information (for attempt events)
    attempt?: {
      attemptId: string;
      processor?: string;
      status: string;
      failureReason?: string;
    };
    
    // Result information (for completion events)
    result?: {
      success: boolean;
      transactionId?: string;
      processor?: string;
      amount?: Money;
      completedAt?: Date;
      finalStrategy?: string;
    };
    
    // Customer interaction (for customer events)
    customerInteraction?: {
      action: string;
      timestamp: Date;
      device?: string;
      location?: string;
    };
  };
  
  metadata?: object;
}
```

### 7.3 Webhook Examples

#### Recovery Completed
```json
{
  "id": "evt_rec_complete_123",
  "eventType": "RECOVERY_COMPLETED",
  "merchantId": "merch_abc123",
  "createdAt": "2025-10-09T14:35:00Z",
  "recoveryId": "rec_9876543210",
  "data": {
    "recoveryId": "rec_9876543210",
    "merchantOrderId": "order_12345",
    "result": {
      "success": true,
      "transactionId": "txn_recovered_456",
      "processor": "stripe_backup",
      "amount": {
        "value": 15000,
        "currency": "USD"
      },
      "completedAt": "2025-10-09T14:35:00Z",
      "finalStrategy": "delayed_retry"
    }
  }
}
```

#### Customer Action Required
```json
{
  "id": "evt_customer_action_456",
  "eventType": "CUSTOMER_ACTION_REQUIRED",
  "merchantId": "merch_abc123",
  "createdAt": "2025-10-06T14:32:00Z",
  "recoveryId": "rec_9876543210",
  "data": {
    "recoveryId": "rec_9876543210",
    "merchantOrderId": "order_12345",
    "strategy": {
      "type": "alternative_payment_method",
      "details": {
        "availableMethods": ["bank_account", "digital_wallet"],
        "recoveryUrl": "https://pay.handsin.com/recovery/rec_9876543210"
      }
    }
  }
}
```


## Appendices

### Appendix A: Failure Code Mapping

Complete mapping of common processor decline codes to failure categories and recovery strategies.

### Appendix B: Processor Integration Specifications

Detailed technical specifications for integrating with various payment processors.

### Appendix C: ML Intelligence Engine

**Note for Partners:** Our Recovery Intelligence Engine uses advanced machine learning to optimize payment recovery. The system continuously learns from millions of transactions to improve success rates.

**Key Capabilities:**
- Real-time strategy prediction (< 100ms response time)
- Continuous learning from transaction outcomes
- Merchant-specific optimization
- Multi-processor routing intelligence
- Fraud and compliance risk assessment

**What This Means for You:**
- Higher recovery rates through intelligent routing
- Reduced manual intervention
- Optimized customer experience
- Transparent confidence scoring
- Actionable insights and recommendations

### Appendix D: Compliance Documentation

PCI DSS, GDPR, PSD2 compliance certifications and audit reports.

### Appendix E: API Rate Limits & Quotas

Detailed rate limiting policies by tier and endpoint.

---

**End of Technical Specification**

*This document is subject to updates as the product evolves. Last updated: October 6, 2025*
