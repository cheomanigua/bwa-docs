---
title: "Stripe Integration"
weight: 60
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
# bookHref: ''
# bookIcon: ''
---

# Stripe Integration

## Stripe Dashboard

> [!NOTE]
> You can limit to one subscription per customer in the Stripe Dashboard -> Settings -> Payments

> [!NOTE]
> You can set all options a customer can make in Customer Portal by editing Dashboard -> Settings -> Billing -> Customer Portal

## API

These are the main fields involved when developing the application:

#### 1. Customers - [API](https://docs.stripe.com/api/customers)
It is the user in the application. Main fields are:
- id
- name
- email

#### 2. Products - [API](https://docs.stripe.com/api/products)
It is the plan name in the application. Main fields are:
- id
- default_price (Prices id)
- name

#### 3. Prices - [API](https://docs.stripe.com/api/prices)
It is the price value in the application. Main fields are:
- id
- product (Products id)

#### 4. Subscriptions - [API](https://docs.stripe.com/api/subscriptions) | [docs](https://docs.stripe.com/subscriptions)
Subscriptions allow customers to make recurring payments for access to a product. It is the user's current recurring subscription in the application. For building the correct subscription integration, visit [here](https://docs.stripe.com/billing/subscriptions/design-an-integration#build-your-subscriptions-integration). Main fields are:
- id
- customer (Customers id)

#### 5. Invoices - [API](https://docs.stripe.com/api/invoices)
It is the monthly transaction in the application. Main fields are:
- id

## Concepts

When creating a product in the Stripe dashboard, you are actually inputing:

- Products:
    - name
- Prices:
    - currency
    - unit_amount
    - type
    - recurring/interval

## Summary

The key challenge for your project is ensuring the Caddy proxy is configured to route all Stripe-related server calls to your Go backend, including the crucial asynchronous **Webhooks**.

### 1\. Backend: Go Implementation

Your Go backend will handle the core Stripe logic using the official Go SDK.

| Task | Go Implementation |
| :--- | :--- |
| **API Key Setup** | Store your **Secret Key** (`sk_test_...`) as an environment variable (e.g., `STRIPE_SECRET_KEY`) for secure access within your Go service. |
| **Create Session** | Implement a Go handler (e.g., `POST /api/checkout`) that calls the Stripe API to create a `checkout.Session` and responds with the session ID or URL. |
| **Webhook Handler** | Implement a dedicated Go handler (e.g., `POST /webhooks/stripe`) to: 1. Verify the signature using the Webhook Signing Secret (`whsec_...`). 2. Deserialize the event. 3. Update your database based on the event type (e.g., fulfill user access upon `checkout.session.completed`). |

### 2\. Caddy/Podman: Environment and Routing

Your Caddyfile and Podman setup need to be updated to expose the new API and webhook routes.

#### **A. Caddyfile Update (Priority Routing)**

You must ensure the new webhook endpoint is caught by the proxy and sent to the Go backend, just like `/api/*`.

Assuming you define your webhook path as `/webhooks/*`, you need to update your `@backend` matcher and the `route` directive in the Caddyfile:

```caddy
// CRITICAL Caddyfile Update:
// (Inside the :5000 block, in the ROUTE: BACKEND section)

@backend {
    path /api/*
    path /webhooks/* // <-- ADD THIS LINE
    path_regexp post_content ^/posts/.+$
}

// ... rest of the 'route @backend' block ...
route @backend {
    reverse_proxy backend:8081 {
        // ... transport details ...
    }
}
```

This ensures `POST /webhooks/stripe` hits your Go service (`backend:8081`) and doesn't get a `405` from the static file server.

#### **B. Podman Compose/Environment Variables**

You need to pass the Stripe keys to your Go backend container using environment variables in your `podman-compose.yml`.

1. **.env**: Create the `.env` file in the root of your project with this content:
    ```bash
    STRIPE_SECRET_KEY="sk_test_yoursecretkeystring"
    STRIPE_WEBHOOK_SECRET="whsec_yourwebhooksecretstring"
    ```

2. **compose.dev.yaml**: Edit the file and add the Stripe and Webhook secrets:

    ```yaml
    # podman-compose.yml snippet
    services:
      backend:
        image: your-go-backend-image
        environment:
          # Use the test secret key
          - STRIPE_SECRET_KEY=${STRIPE_SECRET_KEY}
          # This is the signing secret from the Stripe CLI (see Phase 3)
          - STRIPE_WEBHOOK_SECRET=${STRIPE_WEBHOOK_SECRET}
        # ... rest of the backend config ...
    ```

#### **C. Brigde Stripe with local containers**

You have to bridge the gap between the live Stripe API and your application running on your local machine.

```bash
stripe listen --forward-to 127.0.0.1:8081/api/stripe-webhook
```
You must use the new `whsec_...` secret it outputs to set the `StripeWebhookSecret` in your local Go environment before running your server. This is necessary for the integrity check performed by `webhook.ConstructEventWithOptions(...)`

> [!WARNING]
> Keep the bridge open, don't stop it with **Ctrl** + **C**. Open a new terminal to run your containers.

-----

## Local Development with the Stripe CLI

For local testing, you need the Stripe CLI to tunnel webhooks to your local Podman network.

### Step 1: Install and Log In

If you haven't already:

```bash
stripe login
```

### Step 2: Forward Webhooks to the Go Container

When running Podman, services communicate over a virtual network. To get Stripe webhooks into your Go container (`backend`), you need to listen on the **host machine's** port and forward it to your container's internal IP or port.

Since the Caddy container (which is externally exposed) is already routing traffic to your Go backend, the simplest approach is to forward the webhook traffic to the **Caddy container's exposed port (5000)** on your host machine, hitting the `/webhooks/stripe` endpoint that Caddy proxies to Go.

1.  Run the Stripe CLI listener in a dedicated terminal:
    ```bash
    # This command listens for Stripe events and forwards them to your local Caddy port (5000)
    stripe listen --forward-to localhost:5000/webhooks/stripe
    ```
2.  The CLI will output the **Webhook Signing Secret (`whsec_...`)**.
3.  **Critical:** Copy this `whsec_...` and update the `STRIPE_WEBHOOK_SECRET` environment variable in your `podman-compose.yml` file, then restart your backend service.

Now, whenever Stripe sends a test event, the CLI captures it and sends it through `localhost:5000`, Caddy routes it to your Go service, and your Go webhook handler processes it.

That's a perfect request. Let's start with the **Go backend** implementation for creating the Stripe Checkout Session, which is the most common way to initiate a payment.

## Go Backend: Creating the Checkout Session

This code will be part of your `backend` service. When your frontend calls an API endpoint like `/api/create-checkout-session`, this handler executes, communicates with Stripe, and responds with the URL needed to redirect the user to Stripe's payment page.

### 1\. The Dependencies

Make sure your Go module is configured and you have the Stripe Go SDK installed.

```bash
go get github.com/stripe/stripe-go/v84
```

### 2\. Go Handler: `CreateCheckoutSession`

This example assumes you pass a `priceID` (which you set up in your Stripe Dashboard) to your Go service.

```go
package main

import (
    "net/http"
    "os"
    "log"

    "github.com/stripe/stripe-go/v84"
    "github.com/stripe/stripe-go/v84/checkout/session"
)

// Initialize Stripe with your Secret Key, typically done once at application startup.
// The key should be read from the environment variable (e.g., in main() or init()).
func init() {
    // stripe.Key = os.Getenv("STRIPE_SECRET_KEY")
    // NOTE: For local dev, ensure this env var is set in your podman-compose.yml
    if stripe.Key == "" {
        log.Fatal("STRIPE_SECRET_KEY environment variable not set.")
    }
}

// CreateCheckoutSession handles the request to initiate a Stripe checkout.
// This should be mapped to the /api/create-checkout-session POST endpoint.
func CreateCheckoutSession(w http.ResponseWriter, r *http.Request) {
    // --- 1. Get necessary parameters (e.g., Price ID, User ID) ---
    // In a real application, you would parse the request body for the desired Price ID
    // and retrieve the current user's details (email, customer ID) from the request context.
    
    // For this example, we will hardcode the Price ID and use dummy URLs.
    priceID := "price_1N1xG82eZvKYlo2Cc1LpQ3dF" // Replace with your actual Price ID
    userEmail := "user@example.com"
    
    // --- 2. Define the Session Parameters ---
    params := &stripe.CheckoutSessionParams{
        // Line Items: The product/subscription the user is buying
        LineItems: []*stripe.CheckoutSessionLineItemParams{
            {
                Price:    stripe.String(priceID),
                Quantity: stripe.Int64(1),
            },
        },
        
        // Mode: Set to 'subscription' for recurring payments, or 'payment' for one-time purchases
        Mode: stripe.String(string(stripe.CheckoutSessionModeSubscription)),
        
        // Customer Info: Prefill email to match existing Stripe customers or create a new one.
        CustomerEmail: stripe.String(userEmail),
        
        // URLs: Where to send the user after completion or cancellation.
        // NOTE: Use your public Caddy address (http://localhost:5000) for these.
        SuccessURL: stripe.String("http://localhost:5000/dashboard?session_id={CHECKOUT_SESSION_ID}"),
        CancelURL:  stripe.String("http://localhost:5000/pricing"),
    }

    // --- 3. Create the Session via Stripe API ---
    s, err := session.New(params)
    if err != nil {
        log.Printf("session.New failed: %v", err)
        http.Error(w, "Failed to create checkout session", http.StatusInternalServerError)
        return
    }

    // --- 4. Respond with the Redirect URL ---
    // The frontend will receive this URL and redirect the user.
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusOK)
    // You can respond with a JSON object containing the URL:
    // {"url": "https://checkout.stripe.com/..."}
    w.Write([]byte(`{"url": "` + s.URL + `"}`)) 
}
```

That's the most critical part of a Stripe integration. The Webhook Handler is what securely tells your backend that a payment was successful, allowing you to fulfill the user's order (e.g., granting them premium access, sending an email).

This handler must be very robust, as it receives direct asynchronous calls from Stripe.

## Go Backend: Stripe Webhook Handler

This code handles the `POST /webhooks/stripe` endpoint and includes the necessary **signature verification** to ensure the request truly came from Stripe.

### 1\. The Go Handler: `StripeWebhookHandler`

```go
package main

import (
    "io"
    "log"
    "net/http"
    "os"

    "github.com/stripe/stripe-go/v84"
    "github.com/stripe/stripe-go/v84/webhook"
)

// StripeWebhookHandler processes incoming Stripe events.
// This should be mapped to the /webhooks/stripe POST endpoint.
func StripeWebhookHandler(w http.ResponseWriter, r *http.Request) {
    // 1. Read the Webhook Signing Secret from the environment
    const MaxBodyBytes = int64(65536) // Limit body size for security (64KB)
    
    // CRITICAL: Ensure this is the 'whsec_' secret from the Stripe CLI (for local testing)
    webhookSecret := os.Getenv("STRIPE_WEBHOOK_SECRET") 
    
    // --- 2. Read the Raw Request Body ---
    // The body must be read raw, as Stripe verification fails if it's been parsed/modified.
    r.Body = http.MaxBytesReader(w, r.Body, MaxBodyBytes)
    body, err := io.ReadAll(r.Body)
    if err != nil {
        log.Printf("Error reading request body: %v\n", err)
        http.Error(w, "Request body error", http.StatusServiceUnavailable)
        return
    }

    // --- 3. Verify Signature and Construct Event ---
    signature := r.Header.Get("Stripe-Signature")
    
    event, err := webhook.ConstructEvent(body, signature, webhookSecret)
    if err != nil {
        log.Printf("Error verifying Stripe webhook signature: %v\n", err)
        http.Error(w, "Invalid signature", http.StatusBadRequest)
        return
    }

    // --- 4. Handle the Event Type ---
    switch event.Type {
    case "checkout.session.completed":
        // FULFILLMENT LOGIC
        
        var session stripe.CheckoutSession
        err := json.Unmarshal(event.Data.Raw, &session)
        if err != nil {
            log.Printf("Error unmarshaling event: %v", err)
            // Still return 200, but log the failure
            w.WriteHeader(http.StatusOK) 
            return
        }

        // --- Use session.ID or session.ClientReferenceID to identify the user/order ---
        log.Printf("Checkout session completed for Customer: %s", session.Customer.ID)
        // TODO: Call a function here to update your database, grant access, etc.
        // FulfillOrder(session.ClientReferenceID, session.Customer.ID)
        
    case "customer.subscription.deleted":
        // REVOCATION LOGIC (e.g., subscription expired/cancelled)
        log.Println("Received customer.subscription.deleted event. Revoking access.")
        // TODO: Implement logic to revoke user access
        
    // Add other events you care about (e.g., "invoice.payment_failed")
    default:
        log.Printf("Unhandled event type: %s", event.Type)
    }

    // --- 5. Acknowledge Receipt Immediately (CRITICAL) ---
    // Stripe requires a 200 OK response quickly. If your fulfillment logic is complex,
    // defer it to a background worker (e.g., Go routine, message queue).
    w.WriteHeader(http.StatusOK)
}
```

-----

## Summary of Required Configuration Steps

To make both the Checkout Session and Webhook handlers work in your Caddy/Podman setup, ensure you've completed these steps:

1.  **Caddyfile Update:** Make sure the `/webhooks/*` path is correctly routed to your Go backend service:

    ```caddy
    @backend {
        path /api/*
        path /webhooks/* # <--- MUST INCLUDE THIS
        # ... other paths ...
    }
    // ... rest of the 'route @backend' block ...
    ```

2. **.env file**: Create the `.env` file in the root of your project with this content:
    ```bash
    STRIPE_SECRET_KEY="sk_test_yoursecretkeystring"
    STRIPE_WEBHOOK_SECRET="whsec_yourwebhooksecretstring"
    ```

3.  **Podman Environment:** Update your `podman-compose.yml` to pass the necessary keys to your `backend` container:

    ```yaml
    # podman-compose.yml snippet
    services:
      backend:
        image: your-go-backend-image
        environment:
          - STRIPE_SECRET_KEY=${STRIPE_SECRET_KEY}
          - STRIPE_WEBHOOK_SECRET=${STRIPE_WEBHOOK_SECRET}
        # ...
    ```

4.  **Local Testing (Stripe CLI):** Run the Stripe CLI to tunnel webhooks to your local Caddy endpoint, which will print the necessary `whsec_...` secret:

    ```bash
    stripe listen --forward-to localhost:5000/webhooks/stripe
    ```

You now have the full boilerplate for the client-side initiation (`CreateCheckoutSession`) and the server-side fulfillment (`StripeWebhookHandler`).

## Testing

1. In terminal A, run the `stripe listen` command for tunneling your localhost. Keep it running, don't stop it by pressing **Ctrl** + **C**:
```
stripe listen --forward-to 127.0.0.1:8081/api/stripe-webhook
```
2. Open terminal B and launch the containers.
3. In a browser, go to `localhost:5000` and click on **Register** on any of the pricing cards.
4. Fill out the Stripe form. When finished, you will be redirected to your account dashboard, but because you have not yet checked your "email address", you are redirected to login.
5. In terminal B, check the Go API backend logs:
```
podman logs go-backend
```
You will get something like this:
```
2025/12/16 13:34:14 Initializing Content Guard...
2025/12/16 13:34:14 ContentGuard: /posts/week0001 requires plans: [basic pro]
2025/12/16 13:34:14 ContentGuard: /posts/week0002 requires plans: [basic]
2025/12/16 13:34:14 ContentGuard: /posts/week0003 requires plans: [elite]
2025/12/16 13:34:14 Starting Go server on :8081
2025/12/16 13:35:35 Checkout session completed for Email: juan@mail.com, Plan: elite, Customer ID: cus_TcCpZkxD0hFQdr
2025/12/16 13:35:35 Created new Firebase user with UID: XueVUBP6KfWp18UupZc2wS5BLSQY
2025/12/16 13:35:36 SIMULATING SENDGRID: To johndoe@mail.com, Setup Link: http://127.0.0.1:9099/emulator/action?mode=resetPassword&lang=en&oobCode=lezz8pW7350dqq67PT-XZAKuMikvF6HfCEp7jXu8WM5TAxeZfwbYkr&apiKey=fake-api-key
```
The last logs line is what we are looking for. Now run these command:
```
curl "http://127.0.0.1:9099/emulator/action?mode=resetPassword&lang=en&oobCode=lezz8pW7350dqq67PT-XZAKuMikvF6HfCEp7jXu8WM5TAxeZfwbYkr&apiKey=fake-api-key&newPassword=fakepassword"
```
If everything goes well, you'll get:
```
{
  "authEmulator": {
    "success": "The password has been successfully updated.",
    "email": "johndoe@mail.com"
  }
}
```
Basically we just copied the url of the last line Go API backend log file, added the suffix `&newPassword=fakepassword"` and wrap it in a `curl` command. The `fakepassword` is made up, you can type whatever.

Now we can log in with `johndoe@mail.com` and `fakepassword`
