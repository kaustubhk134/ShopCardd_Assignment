**System Design & Implementation Plan for Global Microservices**

**Topic: Global Notification Service**

**Submitted By: Kaustubh Kulurkar**

## **Why This Microservice?**

Modern applications need to communicate with users constantly - sending order updates, login OTPs, payment alerts, delivery notifications, promotional messages, and more. Each of these messages may go out through different channels like Email, SMS, Push Notifications, or WhatsApp.

Without a centralized system, every team or service ends up building its own notification logic, integrating with different providers, handling failures, retries, templates, and logs separately. This leads to duplicated work, inconsistent user experience, and makes debugging extremely difficult.

The Global Notification Service solves this problem by offering one unified API that any application can use to send notifications. It hides all the complexity of provider integrations, ensures reliable delivery through queues and retries, maintains complete logs, and keeps project-specific data isolated. This makes communication simpler, faster, and more consistent across the entire organization.

## **1\. High-Level Architecture Diagram**

## **API Gateway (Express)**

This is the front door for all incoming requests.

Responsibility: accept requests, do quick checks, and send them to the notification system.

- Verifies the x-api-key so only authorized projects can send messages.
- Validates the request body (required fields, correct format).
- It enforces rate limits (so one project can't spam everyone).
- Returns a quick acknowledgement (for e.g., 202 Accepted + notificationId) so clients don't wait for delivery.

## **Auth Service / API Key Store**

A small service or DB collection that knows which API keys belong to which project.

Responsibility: map incoming API keys to projectId and tenant rules. It also stores metadata like allowed channels and rate-limit settings. In short: "who is calling and what are they allowed to do."

## **Notification Service (Orchestrator)**

The brain that coordinates everything.  
Responsibility:

- Receives validated requests from the gateway.
- Performs idempotency check (i.e. did the same request already run?).
- Persists a notifications record (so every message has a trace).
- Enqueues jobs into the queue for actual sending.
- Returns the notificationId to the caller and tracks overall status.

## **Idempotency Store**

Responsibility: remember requests identified by idempotency-key. If a client resends the same key, the system returns the previous result instead of sending duplicates.

## **Queue (Redis + Bull or similar)**

The temporary holding area for work that needs to be done.  
Responsibility:

- Decouples the API from slow provider calls - API can acknowledge quickly while the queue handles delivery later.
- Manages retry schedules and delays (exponential backoff).
- Keeps jobs durable and visible to workers even if the server restarts.
- Helps control throughput so providers aren't overwhelmed.

## **Worker Pool (Channel-specific workers)**

Background processes that actually send messages. You can run many workers in parallel.  
Responsibility:

- Pull a job from the queue, read the notificationId, and process channels one by one (email, sms, push...).
- Use the right adapter for each channel.
- Update notification_logs and the notifications record with attempt results.
- Throw or mark failures in a way the queue understands so retries happen automatically.

## **Provider Adapters (Email / SMS / Push / WhatsApp modules)**

Small modules that know how to talk to a specific external service (e.g., Nodemailer for SMTP, Twilio for SMS).  
Responsibility:

- Translate the generic notification payload into provider-specific API calls.
- Handle provider responses and normalize success/failure info.
- Verify signatures for incoming webhooks (when providers call us back).
- Keep provider logic isolated so you can swap or add providers easily.

## **Storage (MongoDB)**

The system's memory: stores notifications, projects, idempotency keys, and logs.  
Responsibility:

- Persist notifications (what was requested), notification_logs (every attempt), projects (api keys & limits), and idempotency_keys.
- Provide indexes for quick lookups (by notificationId and projectId) so dashboard and APIs are responsive.
- Serve as the audit trail for troubleshooting and compliance.

## **Dead Letter Queue (DLQ) / Failed Jobs Store**

A place for jobs that failed too many times.  
Responsibility:

- Capture jobs that retried and kept failing (e.g., bad phone numbers or persistent provider errors).
- Let operators inspect, fix, or manually retry those jobs without losing data.

## **Webhook Receiver**

Endpoint to accept asynchronous delivery reports from providers (e.g., SMS delivered, email bounced).  
Responsibility:

- Verify provider signatures to ensure authenticity.
- Update notification_logs and notifications status based on delivery events (DELIVERED, BOUNCED, READ).
- Provide visibility into final delivery states beyond initial SENT.

## **Rate Limiter (per-tenant)**

Controls how many requests a project can make over time.  
Responsibility:

- Prevents abuse and accidental floods from one tenant.
- Uses config per projectId (e.g., 100 req/min).
- Works at the gateway and can also inform worker concurrency decisions.

## **Monitoring & Logging (Prometheus / Grafana / ELK)**

The observability stack.  
Responsibility:

- Track metrics: requests/sec, queue length, worker errors, delivery failure rates.
- Store logs in structured format for fast search.
- Trigger alerts if error rate or queue length crosses thresholds so humans can act quickly.

## **Admin Dashboard (EJS + Bootstrap)**

A simple UI for operators or non-technical reviewers.  
Responsibility:

- Show recent notifications, their statuses, and per-channel logs.
- Allow filtering by projectId, status, or date.
- Provide actions such as re-queueing a DLQ item or retrying a failed notification.

## **Secret & Config Store (ENV or Secrets Manager)**

Where provider API keys and sensitive configs live.  
Responsibility:

- Keep provider credentials out of code.
- Allow rotation and scoped access; for production prefer AWS Secrets Manager or similar.

## **Health & Recovery Utilities**

Small endpoints and scripts to keep the system healthy.  
Responsibility:

- Liveness/readiness endpoints for orchestrators and workers.
- Scripts to requeue stuck jobs, clean expired idempotency keys, and inspect DLQ.

**Data Flow:**

Client calls API → Gateway validates and forwards → Notification Service saves record and enqueues job → Worker picks job and uses adapter to call provider → Worker logs result and updates notification → Webhook may later update delivery status → Monitoring alerts if something goes wrong.

**2\. Database Schema**

**1. projects:**
```bash
{
  \_id: ObjectId,
  projectId: String, // human readable unique id e.g. "shopcard-123"
  name: String,
  apiKeyHash: String, // hashed API key
  allowedChannels: \[String\],// \["email","sms","push","whatsapp"\]
  rateLimit: Number, // requests per minute
  createdAt: Date,
  status: String // 'active' | 'disabled'
}
```
**Indexes: { projectId: 1 } (unique), { apiKeyHash: 1 }**

**2. Notifications:**
```bash
{
  \_id: ObjectId,
  notificationId: String, // UUID v4
  projectId: String,
  templateId: String|null,
  channels: \[ "email", "sms" \],
  payload: Object, // template vars, recipient info
  status: String, // PENDING | PROCESSING | SENT | PARTIAL | FAILED
  attempts: Number,
  meta: { ip, userAgent },
  createdAt: Date,
  updatedAt: Date
}
```
**Indexes: { notificationId: 1 } (unique), { projectId: 1, createdAt: -1 }**

**3. idempotency_keys**
```bash
{
  \_id: ObjectId,
  key: String, // idempotency-key header
  projectId: String,
  notificationId: String,
  createdAt: Date,
  expiresAt: Date
}
```
**Indexes: { key: 1, projectId: 1 } (unique), TTL index on expiresAt**

**4. templates (optional)**
```bash
{
  \_id: ObjectId,
  projectId: String,
  templateId: String,
  channels: \[String\],
  subject: String,
  body: String, // ejs style placeholders
  createdAt: Date
}
```
**3\. & Multi-Tenancy Strategy**

**In this system, many different projects (clients) will use the same Notification Service. This means they all share the same infrastructure - same API, same database, same queue - but their data must remain fully isolated. Project A should never see Project B's notifications, templates, logs, or API usage.**

**To ensure this isolation, we use a simple but very effective strategy:**

## **1\. Every project gets its own API key**

When a project is created, we give it a unique API key.  
This key acts like a secret password.

Whenever a project makes an API call, it must include:

x-api-key: &lt;secret-key&gt;

Only requests with a valid key are allowed in.

## **2\. Each API key is linked to a projectId**

In the database, we store:

apiKeyHash → projectId

So when a request comes in, the server immediately knows:

- Which project is calling
- What channels it is allowed to use
- What rate limits apply

This mapping is the foundation of multi-tenancy.

## **3\. Every record we store includes the projectId**

Any time we create something - a notification, a log entry, a template, or an idempotency key - we always save the projectId inside it.

Example notification record:

{

"notificationId": "12345",

"projectId": "project-A",

"status": "SENT",

"channels": \["email"\]

}

This means if Project B tries to fetch this notification, it will not be returned, because it belongs to Project A.

## **4\. Every database query is automatically filtered by projectId**

Inside the code we always query like this:

db.notifications.find({

notificationId,

projectId: req.projectId

})

This is extremely important.  
It ensures that:

- Even if Project B guesses Project A's notificationId,
- Or even if someone tries to bypass the frontend,

they still cannot access anything that doesn't match their own projectId.

The server will simply return "Not Found".

## **5\. Workers also respect project boundaries**

When workers pick jobs from the queue, each job contains its projectId. Workers use this to:

- Load only the correct templates for that project
- Log events under the right project
- Enforce project-specific rules

Even inside the background processes, tenants stay isolated.

Even though Project A and Project B use the same infrastructure (same API layer, same database, same queue), they can never access each other's data. Every request is authenticated using an API key that maps to a specific projectId. All stored records include this projectId, and every database query is automatically filtered by it. This ensures complete tenant isolation.

**3\. Integration & Implementation Plan**

# **API Design - Key Endpoints**

**Below are the primary APIs exposed by the Global Notification Service. All APIs require the tenant's x-api-key for authentication and automatically scope data to the corresponding projectId.**

## **1\. Send Notification**

### **POST /api/v1/notifications/send**

**Trigger a notification through one or multiple channels (Email, SMS, Push, WhatsApp).**

**Headers:**

- **x-api-key: &lt;project-api-key&gt;**
- **idempotency-key: &lt;optional-key&gt;**

**Request Body:**

**{**

**"templateId": "order_confirmation",**

**"channels": \["email", "sms"\],**

**"payload": {**

**"to": {**

**"email": "<user@example.com>",**

**"phone": "+9199xxxxxxx"**

**},**

**"vars": {**

**"userName": "Amit",**

**"orderId": "ORD-1234"**

**}**

**},**

**"metadata": {**

**"source": "checkout-page"**

**}**

**}**

**Response (202):**

**{**

**"notificationId": "uuid-v4",**

**"status": "PENDING"**

**}**

## **2\. Get Notification Status**

### **GET /api/v1/notifications/{notificationId}**

**Fetch the current status of a notification.**

**Headers:**

- **x-api-key: &lt;project-api-key&gt;**

**Response:**

**{**

**"notificationId": "uuid-v4",**

**"projectId": "project-A",**

**"channels": \["email", "sms"\],**

**"status": "SENT",**

**"attempts": 1,**

**"logs": \[**

**{ "channel": "email", "event": "SENT", "timestamp": "..." },**

**{ "channel": "sms", "event": "DELIVERED", "timestamp": "..." }**

**\]**

**}**

## **3\. List Notifications (Paginated)**

### **GET /api/v1/notifications?limit=20&page=1**

**Returns notifications that belong to the tenant, sorted by newest first.**

**Headers:**

- **x-api-key: &lt;project-api-key&gt;**

## **4\. Retry Failed Notification**

### **POST /api/v1/notifications/{notificationId}/retry**

**Manually retry a failed notification (useful for dashboard or admin tools).**

**Headers:**

- **x-api-key: &lt;project-api-key&gt;**

## **5\. Webhook Endpoint (Provider Callbacks)**

### **POST /api/v1/hooks/{providerName}**

**Receives delivery receipts from external providers (e.g., Twilio, SMTP, FCM).**

**Headers:**

- **Provider-specific signature headers**

**Body Example:**

**{**

**"providerMessageId": "abc123",**

**"event": "DELIVERED",**

**"timestamp": 1234567890**

**}**

## **6\. Admin - Create Project (Internal Only)**

### **POST /api/v1/admin/projects**

**Used internally to create new tenants and generate API keys.**

**Body:**

**{**

**"name": "ShopCard Application",**

**"allowedChannels": \["email", "sms", "push"\]**

**}**

**Response:**

**{**

**"projectId": "shopcard-001",**

**"apiKey": "generated-secret-key"**

**}**

**_Note: This endpoint is protected using an internal admin token. Not available to normal clients._**

## **7\. Admin - Create or Update Template**

### **POST /api/v1/admin/templates**

**Uploads notification templates per tenant.**

**Body:**

**{**

**"projectId": "shopcard-001",**

**"templateId": "order_confirmation",**

**"channels": \["email"\],**

**"subject": "Order Confirmed: {{orderId}}",**

**"body": "Hello {{userName}}, your order {{orderId}} has been placed."**

**}**

**Tech stack choices:**

## **Backend framework - Node.js + Express**

**Why:** You already know JavaScript/Node and Express is lightweight and fast to prototype with. It's excellent for building REST APIs and has a huge ecosystem (middleware, validation, authentication).  
**Alternative:** Java + Spring Boot (more "enterprisey" but heavier).

## **Database (primary) - MongoDB**

**Why:** Flexible document model fits notifications (variable payloads, logs, templates). Easy to add fields and store per-tenant documents like notifications, logs, projects. Mongoose makes schemas straightforward.  
**Alternative:** PostgreSQL if you prefer strong relational guarantees or want ACID for billing/ledger features.

## **Queue / Job Processing - Redis + Bull (or BullMQ)**

**Why:** Simple to run locally (Docker), easy API for delayed/retry jobs and DLQ patterns, good visibility of job states. Perfect for sending notifications asynchronously and implementing backoff/retry logic.  
**Alternative:** RabbitMQ (more feature-rich) or AWS SQS (managed).

## **Email adapter - Nodemailer (SMTP)**

**Why:** Works with any SMTP provider, simple to configure for demos or real SMTP like SendGrid/Mailgun. Lets you show working email sends without complex SDKs.  
**Alternative:** Provider SDKs (SendGrid SDK) for richer features.

## **SMS / WhatsApp adapter - Twilio (or pluggable stub)**

**Why:** Twilio is industry-standard, exposes SMS and WhatsApp APIs, and has good docs. For assignment/demo, you can implement a small stub adapter and swap in Twilio later.  
**Alternative:** Nexmo (Vonage), Plivo, or provider-specific APIs.

## **Push notifications - Firebase Cloud Messaging (FCM) (or stub)**

**Why:** FCM is free and standard for Android/iOS/web push. Easy to mock for a design document and optional to wire in later.  
**Alternative:** APNs directly for iOS (more setup).

## **Background workers - Node.js worker processes (same codebase or separate service)**

**Why:** Keeps business logic in JavaScript, easy to share adapters/models between API and worker; spawn multiple worker processes for scaling.  
**Alternative:** Separate microservice in another language if needed later.

## **Caching / Rate-limiting store - Redis**

**Why:** Fast for token buckets and rate limiting; also used by Bull so minimal infra duplication.  
**Alternative:** In-memory (bad for multiple instances) or external rate-limiting services.

## **Local development & reproducible env**

**Docker Compose** (MongoDB, Redis, app)  
**Why:** Easy to run everything locally and demonstrate the flow on your laptop.

**Failure Handling:**

**Quick detection & restart** - health checks (liveness/readiness) let orchestration (Docker Compose/systemd/Kubernetes) detect failures and restart the service automatically.  

**Graceful recovery** - when the service comes back, workers continue processing jobs from the queue and update the database with latest statuses.  

**Human alerting** - monitoring tools (Prometheus/Sentry) trigger alerts (email/Slack/pager) if the service is down or error rates spike, so an engineer can investigate.

**If a 3rd-party API fails:**

If the third party API fails we can handle it by using optional chaining from preventing

**4\. Roadmap (Micro-Steps)**

**Phase 1: Setup & Database  
**

- Start by setting up the basic Node.js + Express project and creating a clean folder structure.  

- Configure environment variables and connect the service to MongoDB using Mongoose.  

- Create a small seed script to insert a sample project with an API key so I can test authentication later.  

- Add a simple /health route to confirm the service is running properly.  

**Phase 2: Core Notification APIs  
**

- Build the main endpoint: POST /api/v1/notifications/send which creates a notification entry in the database and returns a notificationId.
- Implement GET /api/v1/notifications/:id for viewing the status of a specific notification, and a paginated listing API for all notifications under a project.
- Add the idempotency key logic so repeated requests with the same key don't create duplicate notifications.  

**Phase 3: Security & Tenant Isolation  
**

- Implement middleware to validate the x-api-key and map it to the correct projectId.
- Ensure that every database query automatically filters by projectId, so one project never accesses another's data.
- Add basic rate limiting per project (e.g., requests/minute) to avoid overload or misuse.  

**Phase 4: Queue & External Provider Integration  
**

- Integrate Redis + Bull to push each notification into a background queue instead of sending it directly.
- Create a worker process that pulls jobs from the queue and calls channel adapters (email, SMS, etc.). Initially, these will be simple stubs to simulate sends.
- Add retry logic with exponential backoff and a dead-letter queue for jobs that repeatedly fail.  

**Phase 5: Polishing & Documentation**

- Store logs of every attempt in a notification_logs collection to keep a full audit trail.
- Write a clear README with setup steps, API usage examples, and environment variables.
- Prepare the final assignment document with architecture diagrams, DB schema, API list, and this roadmap.
