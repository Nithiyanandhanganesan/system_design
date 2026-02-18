# Webhooks - Complete Guide

## What are Webhooks?

Webhooks are **HTTP callbacks** that automatically send data to a specified URL when specific events occur. Think of them as "reverse APIs" - instead of you polling for data, the service notifies you when something happens.

### Traditional Polling vs Webhooks

```
Polling (Inefficient):
Your App  ──GET /events──>  Service
Your App  <──"No new data"── Service
Your App  ──GET /events──>  Service  
Your App  <──"No new data"── Service
Your App  ──GET /events──>  Service
Your App  <──"New data!"─── Service

Webhooks (Efficient):
Your App  <──POST /webhook── Service (when event occurs)
```

## How Webhooks Work

### 1. **Registration**
You register a webhook URL with a service:
```http
POST /api/webhooks
{
  "url": "https://myapp.com/webhook/payment-completed",
  "events": ["payment.completed", "payment.failed"]
}
```

### 2. **Event Occurs**
Something happens in the service (payment completed, new user registered, etc.)

### 3. **HTTP POST Request**
The service sends data to your webhook URL:
```http
POST /webhook/payment-completed
Content-Type: application/json
X-Webhook-Event: payment.completed
X-Webhook-Signature: sha256=abc123...

{
  "event": "payment.completed",
  "timestamp": "2026-02-15T10:30:00Z",
  "data": {
    "payment_id": "pay_123456789",
    "amount": 2999,
    "currency": "USD",
    "customer_id": "cust_987654321"
  }
}
```

## Real-World Webhook Examples

### 1. **Payment Processing (Stripe)**
```javascript
// Webhook endpoint to handle Stripe events
app.post('/webhook/stripe', express.raw({type: 'application/json'}), (req, res) => {
    const sig = req.headers['stripe-signature'];
    let event;
    
    try {
        // Verify webhook signature
        event = stripe.webhooks.constructEvent(req.body, sig, process.env.STRIPE_WEBHOOK_SECRET);
    } catch (err) {
        console.log('Webhook signature verification failed:', err.message);
        return res.status(400).send(`Webhook Error: ${err.message}`);
    }
    
    // Handle the event
    switch (event.type) {
        case 'payment_intent.succeeded':
            const paymentIntent = event.data.object;
            await handlePaymentSuccess(paymentIntent);
            break;
            
        case 'payment_intent.payment_failed':
            const failedPayment = event.data.object;
            await handlePaymentFailure(failedPayment);
            break;
            
        case 'customer.subscription.updated':
            const subscription = event.data.object;
            await updateSubscription(subscription);
            break;
            
        default:
            console.log(`Unhandled event type ${event.type}`);
    }
    
    res.json({received: true});
});

async function handlePaymentSuccess(paymentIntent) {
    // Update order status in database
    await Order.update(
        { status: 'paid', payment_id: paymentIntent.id },
        { where: { id: paymentIntent.metadata.order_id } }
    );
    
    // Send confirmation email
    await sendPaymentConfirmationEmail(paymentIntent.metadata.customer_email);
    
    // Update inventory
    await updateInventory(paymentIntent.metadata.product_id);
}
```

### 2. **GitHub Integration**
```javascript
// Handle GitHub webhook events
app.post('/webhook/github', (req, res) => {
    const event = req.headers['x-github-event'];
    const payload = req.body;
    
    switch (event) {
        case 'push':
            handlePushEvent(payload);
            break;
            
        case 'pull_request':
            handlePullRequestEvent(payload);
            break;
            
        case 'issues':
            handleIssueEvent(payload);
            break;
            
        default:
            console.log(`Unhandled GitHub event: ${event}`);
    }
    
    res.status(200).send('OK');
});

async function handlePushEvent(payload) {
    const { repository, commits, pusher } = payload;
    
    // Trigger CI/CD pipeline
    if (payload.ref === 'refs/heads/main') {
        await triggerDeployment({
            repo: repository.full_name,
            commit: payload.after,
            branch: 'main'
        });
    }
    
    // Send Slack notification
    await sendSlackMessage(`New push to ${repository.name} by ${pusher.name}: ${commits.length} commits`);
}

async function handlePullRequestEvent(payload) {
    const { action, pull_request } = payload;
    
    if (action === 'opened') {
        // Run automated tests
        await triggerPRTests(pull_request);
        
        // Add PR to review queue
        await addToReviewQueue(pull_request);
    }
}
```

### 3. **E-commerce Order Processing**
```javascript
// Handle order events from multiple sources
app.post('/webhook/orders', async (req, res) => {
    try {
        const { source, event_type, order_data } = req.body;
        
        // Verify webhook authenticity
        if (!verifyWebhookSignature(req)) {
            return res.status(401).json({ error: 'Invalid signature' });
        }
        
        switch (event_type) {
            case 'order.created':
                await processNewOrder(order_data);
                break;
                
            case 'order.paid':
                await fulfillOrder(order_data);
                break;
                
            case 'order.cancelled':
                await cancelOrder(order_data);
                break;
                
            case 'order.refunded':
                await processRefund(order_data);
                break;
        }
        
        res.json({ status: 'processed' });
    } catch (error) {
        console.error('Webhook processing error:', error);
        res.status(500).json({ error: 'Internal server error' });
    }
});

async function processNewOrder(orderData) {
    // Create order in database
    const order = await Order.create({
        external_id: orderData.id,
        customer_email: orderData.customer.email,
        total_amount: orderData.total,
        items: orderData.line_items,
        status: 'pending'
    });
    
    // Check inventory
    for (const item of orderData.line_items) {
        const available = await checkInventory(item.product_id, item.quantity);
        if (!available) {
            await sendInventoryAlert(item.product_id);
        }
    }
    
    // Send order confirmation
    await sendOrderConfirmation(orderData.customer.email, order);
}
```

## Webhook Security

### 1. **Signature Verification**
```javascript
const crypto = require('crypto');

function verifyWebhookSignature(req, secret) {
    const signature = req.headers['x-webhook-signature'];
    const body = JSON.stringify(req.body);
    
    // Create expected signature
    const expectedSignature = crypto
        .createHmac('sha256', secret)
        .update(body)
        .digest('hex');
    
    const expectedHeader = `sha256=${expectedSignature}`;
    
    // Use timingSafeEqual to prevent timing attacks
    return crypto.timingSafeEqual(
        Buffer.from(signature),
        Buffer.from(expectedHeader)
    );
}

// Usage in webhook endpoint
app.post('/webhook', (req, res) => {
    if (!verifyWebhookSignature(req, process.env.WEBHOOK_SECRET)) {
        return res.status(401).json({ error: 'Invalid signature' });
    }
    
    // Process webhook
    processWebhook(req.body);
    res.json({ received: true });
});
```

### 2. **IP Whitelist**
```javascript
const allowedIPs = [
    '192.168.1.100',    // Service provider IP
    '10.0.0.50',        // Another allowed IP
    '203.0.113.0/24'    // CIDR range
];

function isIPAllowed(ip) {
    return allowedIPs.some(allowedIP => {
        if (allowedIP.includes('/')) {
            // Handle CIDR notation
            return isIPInCIDR(ip, allowedIP);
        }
        return ip === allowedIP;
    });
}

app.post('/webhook', (req, res) => {
    const clientIP = req.ip || req.connection.remoteAddress;
    
    if (!isIPAllowed(clientIP)) {
        console.log(`Rejected webhook from unauthorized IP: ${clientIP}`);
        return res.status(403).json({ error: 'Forbidden' });
    }
    
    // Process webhook
});
```

### 3. **Rate Limiting**
```javascript
const rateLimit = require('express-rate-limit');

// Rate limit webhook endpoints
const webhookRateLimit = rateLimit({
    windowMs: 15 * 60 * 1000, // 15 minutes
    max: 100, // limit each IP to 100 requests per windowMs
    message: 'Too many webhook requests from this IP',
    standardHeaders: true,
    legacyHeaders: false,
});

app.use('/webhook', webhookRateLimit);
```

### 4. **Request Validation**
```javascript
const Joi = require('joi');

const webhookSchema = Joi.object({
    event_type: Joi.string().required(),
    timestamp: Joi.string().isoDate().required(),
    data: Joi.object().required(),
    source: Joi.string().valid('stripe', 'github', 'shopify').required()
});

app.post('/webhook', (req, res) => {
    // Validate request body
    const { error, value } = webhookSchema.validate(req.body);
    
    if (error) {
        console.log('Invalid webhook payload:', error.details);
        return res.status(400).json({ 
            error: 'Invalid payload', 
            details: error.details 
        });
    }
    
    // Process validated webhook
    processWebhook(value);
    res.json({ received: true });
});
```

## Webhook Reliability Patterns

### 1. **Retry Logic (Server-Side)**
```javascript
class WebhookDelivery {
    constructor() {
        this.maxRetries = 5;
        this.retryDelays = [1000, 2000, 5000, 10000, 30000]; // Exponential backoff
    }
    
    async sendWebhook(url, payload, attempt = 0) {
        try {
            const response = await axios.post(url, payload, {
                timeout: 10000,
                headers: {
                    'Content-Type': 'application/json',
                    'X-Webhook-Event': payload.event,
                    'X-Webhook-Delivery': uuidv4(),
                    'X-Webhook-Attempt': attempt + 1
                }
            });
            
            if (response.status >= 200 && response.status < 300) {
                console.log(`Webhook delivered successfully to ${url}`);
                return { success: true, attempt: attempt + 1 };
            } else {
                throw new Error(`HTTP ${response.status}: ${response.statusText}`);
            }
            
        } catch (error) {
            console.log(`Webhook delivery failed (attempt ${attempt + 1}): ${error.message}`);
            
            if (attempt < this.maxRetries - 1) {
                // Schedule retry
                const delay = this.retryDelays[attempt];
                console.log(`Retrying in ${delay}ms...`);
                
                setTimeout(() => {
                    this.sendWebhook(url, payload, attempt + 1);
                }, delay);
            } else {
                console.error(`Webhook delivery failed after ${this.maxRetries} attempts`);
                await this.handleFinalFailure(url, payload);
            }
        }
    }
    
    async handleFinalFailure(url, payload) {
        // Store failed webhook for manual retry
        await FailedWebhook.create({
            url,
            payload,
            failed_at: new Date(),
            attempts: this.maxRetries
        });
        
        // Alert administrators
        await sendAlert(`Webhook delivery failed: ${url}`);
    }
}
```

### 2. **Idempotency (Client-Side)**
```javascript
// Handle duplicate webhooks gracefully
const processedWebhooks = new Set();

app.post('/webhook', async (req, res) => {
    const webhookId = req.headers['x-webhook-delivery'] || 
                     `${req.body.event_type}_${req.body.timestamp}`;
    
    // Check if already processed
    if (processedWebhooks.has(webhookId)) {
        console.log(`Duplicate webhook ignored: ${webhookId}`);
        return res.json({ received: true, duplicate: true });
    }
    
    try {
        // Process webhook
        await processWebhook(req.body);
        
        // Mark as processed
        processedWebhooks.add(webhookId);
        
        // Clean up old entries (prevent memory leak)
        if (processedWebhooks.size > 10000) {
            const entries = Array.from(processedWebhooks);
            const toKeep = entries.slice(-5000); // Keep last 5000
            processedWebhooks.clear();
            toKeep.forEach(id => processedWebhooks.add(id));
        }
        
        res.json({ received: true });
    } catch (error) {
        console.error('Webhook processing error:', error);
        res.status(500).json({ error: 'Processing failed' });
    }
});
```

### 3. **Dead Letter Queue**
```javascript
const Queue = require('bull');
const webhookQueue = new Queue('webhook processing');
const deadLetterQueue = new Queue('failed webhooks');

// Main webhook processor
webhookQueue.process(async (job) => {
    const { url, payload } = job.data;
    
    try {
        await processWebhookPayload(payload);
        console.log(`Webhook processed: ${payload.event_type}`);
    } catch (error) {
        console.error(`Webhook processing failed: ${error.message}`);
        throw error; // This will trigger retry
    }
});

// Handle failed jobs
webhookQueue.on('failed', async (job, error) => {
    console.log(`Job ${job.id} failed after ${job.attemptsMade} attempts`);
    
    if (job.attemptsMade >= job.opts.attempts) {
        // Move to dead letter queue
        await deadLetterQueue.add('failed webhook', {
            originalJob: job.data,
            error: error.message,
            attempts: job.attemptsMade,
            failedAt: new Date()
        });
    }
});

// Add webhook to queue
app.post('/webhook', (req, res) => {
    webhookQueue.add('process webhook', {
        payload: req.body,
        receivedAt: new Date()
    }, {
        attempts: 3,
        backoff: {
            type: 'exponential',
            delay: 2000
        }
    });
    
    res.json({ received: true });
});
```

## Testing Webhooks

### 1. **Local Development with ngrok**
```bash
# Install ngrok
npm install -g ngrok

# Start your local server
node server.js

# In another terminal, expose local server
ngrok http 3000

# Use the ngrok URL for webhook registration
# https://abc123.ngrok.io/webhook
```

### 2. **Webhook Testing Tool**
```javascript
// Simple webhook testing server
const express = require('express');
const app = express();

app.use(express.json());

// Log all incoming webhooks
app.post('/test-webhook', (req, res) => {
    console.log('=== Incoming Webhook ===');
    console.log('Headers:', req.headers);
    console.log('Body:', JSON.stringify(req.body, null, 2));
    console.log('========================');
    
    // Always respond with success
    res.json({ 
        received: true, 
        timestamp: new Date().toISOString() 
    });
});

app.listen(3000, () => {
    console.log('Webhook test server running on http://localhost:3000');
    console.log('Test endpoint: http://localhost:3000/test-webhook');
});
```

### 3. **Unit Testing Webhooks**
```javascript
const request = require('supertest');
const app = require('../server');

describe('Webhook Endpoints', () => {
    test('should process payment webhook correctly', async () => {
        const webhookPayload = {
            event_type: 'payment.completed',
            data: {
                payment_id: 'pay_test_123',
                amount: 2000,
                currency: 'USD'
            },
            timestamp: new Date().toISOString()
        };
        
        const response = await request(app)
            .post('/webhook/payments')
            .send(webhookPayload)
            .expect(200);
            
        expect(response.body.received).toBe(true);
        
        // Verify side effects
        const order = await Order.findOne({ 
            where: { payment_id: 'pay_test_123' } 
        });
        expect(order.status).toBe('paid');
    });
    
    test('should reject webhook with invalid signature', async () => {
        const response = await request(app)
            .post('/webhook/payments')
            .set('X-Webhook-Signature', 'invalid')
            .send({ event_type: 'test' })
            .expect(401);
            
        expect(response.body.error).toBe('Invalid signature');
    });
});
```

## Webhook Best Practices

### 1. **Always Respond Quickly**
```javascript
// Good: Process asynchronously
app.post('/webhook', (req, res) => {
    // Respond immediately
    res.json({ received: true });
    
    // Process asynchronously
    setImmediate(() => {
        processWebhookAsync(req.body);
    });
});

// Or use a queue
app.post('/webhook', (req, res) => {
    webhookQueue.add('process', req.body);
    res.json({ received: true });
});
```

### 2. **Log Everything**
```javascript
const winston = require('winston');

const logger = winston.createLogger({
    level: 'info',
    format: winston.format.json(),
    transports: [
        new winston.transports.File({ filename: 'webhooks.log' })
    ]
});

app.post('/webhook', (req, res) => {
    const webhookId = uuidv4();
    
    logger.info('Webhook received', {
        webhookId,
        event: req.body.event_type,
        timestamp: new Date().toISOString(),
        headers: req.headers,
        body: req.body
    });
    
    try {
        processWebhook(req.body);
        
        logger.info('Webhook processed successfully', {
            webhookId,
            event: req.body.event_type
        });
    } catch (error) {
        logger.error('Webhook processing failed', {
            webhookId,
            event: req.body.event_type,
            error: error.message,
            stack: error.stack
        });
    }
    
    res.json({ received: true });
});
```

### 3. **Monitor Webhook Health**
```javascript
// Webhook monitoring
class WebhookMonitor {
    constructor() {
        this.metrics = {
            received: 0,
            processed: 0,
            failed: 0,
            lastReceived: null
        };
        
        // Alert if no webhooks received in 1 hour
        setInterval(() => {
            this.checkHealth();
        }, 60 * 60 * 1000);
    }
    
    recordReceived() {
        this.metrics.received++;
        this.metrics.lastReceived = new Date();
    }
    
    recordProcessed() {
        this.metrics.processed++;
    }
    
    recordFailed() {
        this.metrics.failed++;
    }
    
    checkHealth() {
        const hourAgo = new Date(Date.now() - 60 * 60 * 1000);
        
        if (!this.metrics.lastReceived || this.metrics.lastReceived < hourAgo) {
            this.sendAlert('No webhooks received in the last hour');
        }
        
        const errorRate = this.metrics.failed / this.metrics.received;
        if (errorRate > 0.1) { // More than 10% failure rate
            this.sendAlert(`High webhook failure rate: ${(errorRate * 100).toFixed(1)}%`);
        }
    }
    
    async sendAlert(message) {
        // Send to monitoring service
        console.error(`WEBHOOK ALERT: ${message}`);
    }
}

const monitor = new WebhookMonitor();
```

## Webhooks vs Alternatives

### Webhooks vs Polling
```javascript
// Polling - Client pulls data
setInterval(async () => {
    const response = await fetch('/api/orders/status');
    const orders = await response.json();
    checkForUpdates(orders);
}, 30000); // Every 30 seconds

// Webhooks - Server pushes data
app.post('/webhook/order-updated', (req, res) => {
    const { order_id, new_status } = req.body;
    updateOrderStatus(order_id, new_status); // Immediate update
    res.json({ received: true });
});
```

### Webhooks vs WebSockets
- **Webhooks**: Server-to-server communication, one-way
- **WebSockets**: Real-time bidirectional communication, client-server

### Webhooks vs Message Queues
- **Webhooks**: HTTP-based, simple integration
- **Message Queues**: More reliable, better for high-volume internal communication

## Summary

**Webhooks are ideal for:**
✅ Event-driven integrations
✅ Real-time notifications  
✅ Automation triggers
✅ Third-party service integration
✅ Microservices communication

**Key Benefits:**
- Real-time data delivery
- Reduces unnecessary API calls
- Event-driven architecture
- Simple HTTP-based integration

**Important Considerations:**
- Security (signature verification)
- Reliability (retry logic, idempotency)
- Monitoring and logging
- Error handling and fallbacks
