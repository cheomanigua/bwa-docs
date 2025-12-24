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
> All Products and Prices are created manually in the Stripe Dashboard

> [!NOTE]
> You can limit to one subscription per customer in the Stripe Dashboard -> Settings -> Payments

> [!NOTE]
> You can set all options a customer can perform in Customer Portal by editing Stripe Dashboard -> Settings -> Billing -> Customer Portal

> [!NOTE]
> You can create prices **Lookup Keys** in Stripe Dashboard -> Product catalog -> click on a product -> click on the price settings button -> Edit Price -> Lookup key

> [!NOTE]
> You can configure your webhooks in Stripe Dashboard -> Settings -> Payments -> Checkout and Payments Links tab -> Configure webhooks

## Summary

The key challenge for your project is ensuring the Caddy proxy is configured to route all Stripe-related server calls to your Go backend, including the crucial asynchronous **Webhooks**.

### 1\. Backend: Go Implementation

Your Go backend will handle the core Stripe logic using the official Go SDK.

| Task | Go Implementation |
| :--- | :--- |
| **API Key Setup** | Store your **Secret Key** (`sk_test_...`) as an environment variable (e.g., `STRIPE_SECRET_KEY`) for secure access within your Go service. |
| **Create Session** | Implement a Go handler (e.g., `POST /api/checkout`) that calls the Stripe API to create a `checkout.Session` and responds with the session ID or URL. |
| **Webhook Handler** | Implement a dedicated Go handler (e.g., `POST /webhooks/stripe`) to: 1. Verify the signature using the Webhook Signing Secret (`whsec_...`). 2. Deserialize the event. 3. Update your database based on the event type (e.g., fulfill user access upon `checkout.session.completed`). |

### 2\. Podman: Environment and Bridging

Your local machine and Podman setup need to be updated to expose the new API and webhook bridge.

#### **A. Brigde Stripe with local containers**

You have to bridge the gap between the live Stripe API and your application running on your local machine.

```bash
stripe listen --forward-to localhost:5000/api/stripe-webhook
```

You will get this output:

```bash
> Ready! You are using Stripe API Version [2022-11-15].
Your webhook signing secret is whsec_longalphanumericstring (^C to quit)
```
> [!WARNING]
> Keep the bridge open, don't stop it with **Ctrl** + **C**. Open a new terminal to run your containers.

You must use the `whsec_` secret it outputs to set the `StripeWebhookSecret` in your local Go environment before running your server. This is necessary for the integrity check performed by `webhook.ConstructEventWithOptions(...)` in `main.go`:
```go
StripeWebhookSecret = getEnv("STRIPE_WEBHOOK_SECRET", "whsec_...")

//..

    event, err := webhook.ConstructEventWithOptions(
        payload,
        r.Header.Get("Stripe-Signature"),
        StripeWebhookSecret,
        webhook.ConstructEventOptions{
            // Avoid conflict between locally installed Stripe and container installed Stripe
            IgnoreAPIVersionMismatch: true,
        },
    )
```



#### **B. Podman Compose/Environment Variables**

You need to pass the Stripe keys to your Go backend container using environment variables in your `podman-compose.yml`.

1. **.env**: Create the `.env` file in the root of your project in your local machine with this content:
    ```bash
    STRIPE_SECRET_KEY="sk_test_yoursecretkeystring"
    STRIPE_WEBHOOK_SECRET="whsec_longalphanumericstring"
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
    stripe listen --forward-to localhost:5000/api/stripe-webhook
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

### 2\. Go Handler: `handleCreateCheckoutSession`

This example assumes you are using Price Lookup Key (which you set up in your Stripe Dashboard) to your Go service.

```go
package main

import (
    // ..
    "github.com/stripe/stripe-go/v84"
    checkoutsession "github.com/stripe/stripe-go/v84/checkout/session"
)

var (
    // ..
	StripeSecretKey     = getEnv("STRIPE_SECRET_KEY", "sk_test_...")
	StripeWebhookSecret = getEnv("STRIPE_WEBHOOK_SECRET", "whsec_...")
	Domain              = getEnv("DOMAIN", "http://localhost:5000")
)

const (
	PlanLookupBasic = "basic_plan"
	PlanLookupPro   = "pro_plan"
	PlanLookupElite = "elite_plan"
)


// Initialize Stripe with your Secret Key, typically done once at application startup.
// The key should be read from the environment variable (e.g., in main() or init()).
func main() {
    stripe.Key = StripeSecretKey // Stripe functions use stripe.Key under the hood, no need to call it
}

// handleCreateCheckoutSession resolves the human-readable plan name to a Stripe Price ID via Lookup Keys.
func handleCreateCheckoutSession(w http.ResponseWriter, r *http.Request) {
	if r.Method != http.MethodPost {
		http.Error(w, "Method not allowed", http.StatusMethodNotAllowed)
		return
	}

	var req struct {
		PlanName string `json:"planName"` // Frontend sends "basic_plan", "pro_plan", etc.
	}
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		http.Error(w, "Invalid request", http.StatusBadRequest)
		return
	}

	// 1. Resolve Lookup Key to actual Price ID
	priceParams := &stripe.PriceListParams{}
	priceParams.LookupKeys = []*string{stripe.String(req.PlanName)}
	i := price.List(priceParams)

	var targetPrice *stripe.Price
	for i.Next() {
		targetPrice = i.Price()
	}

	if targetPrice == nil {
		log.Printf("Could not find Price for Lookup Key: %s", req.PlanName)
		http.Error(w, "Invalid plan selected", http.StatusBadRequest)
		return
	}

	// 2. Create Checkout Session using the retrieved ID
	params := &stripe.CheckoutSessionParams{
		Mode: stripe.String(string(stripe.CheckoutSessionModeSubscription)),
		LineItems: []*stripe.CheckoutSessionLineItemParams{
			{
				Price:    stripe.String(targetPrice.ID),
				Quantity: stripe.Int64(1),
			},
		},
		BillingAddressCollection: stripe.String(string(stripe.CheckoutSessionBillingAddressCollectionRequired)),
		SuccessURL:               stripe.String(Domain + "/dashboard?session_id={CHECKOUT_SESSION_ID}"),
		CancelURL:                stripe.String(Domain + "/"),
	}

	s, err := checkoutsession.New(params)
	if err != nil {
		log.Printf("Checkout creation failed: %v", err)
		http.Error(w, "Could not initiate checkout.", http.StatusInternalServerError)
		return
	}

	w.WriteHeader(http.StatusOK)
	json.NewEncoder(w).Encode(map[string]string{"sessionId": s.ID})
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

1.  **Local Testing (Stripe CLI):** Run the Stripe CLI to tunnel webhooks to your local Caddy endpoint, which will print the necessary `whsec_...` secret:

    ```bash
    stripe listen --forward-to localhost:5000/api/stripe-webhook
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


You now have the full boilerplate for the client-side initiation (`CreateCheckoutSession`) and the server-side fulfillment (`StripeWebhookHandler`).

## Lookup Keys

The **Lookup Key** is a specific Stripe feature designed exactly for what you are trying to achieve: **decoupling your code from Price IDs.**


### How to use Lookup Keys to avoid Price IDs

#### 1. Set the Lookup Key in Stripe

In the same "Edit Price" area where you saw the field:

* For your Basic price, set the Lookup Key to `basic_plan`.
* For your Pro price, set it to `pro_plan`.
* For your Elite price, set it to `elite_plan`.

#### 2. Update your Go code to use Lookup Keys

Instead of passing a `price_...` ID from your frontend or env, you tell Stripe to find the price based on that key.

**In `handleCreateCheckoutSession`:**
You can now accept a "slug" like `pro` from your frontend instead of a Price ID.

```go
// main.go

func handleCreateCheckoutSession(w http.ResponseWriter, r *http.Request) {
    // ... parse request ...
    // Assume req.PlanName is "pro_plan", "basic_plan", etc.

    params := &stripe.CheckoutSessionParams{
        SuccessURL: stripe.String(Domain + "/dashboard?session_id={CHECKOUT_SESSION_ID}"),
        LineItems: []*stripe.CheckoutSessionLineItemParams{
            {
                // Use PriceData with a LookupKey instead of a direct Price ID
                PriceData: &stripe.CheckoutSessionLineItemPriceDataParams{
                    LookupKey: stripe.String(req.PlanName),
                },
                Quantity: stripe.Int64(1),
            },
        },
        Mode: stripe.String(string(stripe.CheckoutSessionModeSubscription)),
    }
    // ...
}

```

#### 3. Handling the Webhook with Lookup Keys

When the webhook fires, the `Price` object will contain the `LookupKey`. You no longer need a map or environment variables to identify the plan.

```go
// Inside handleStripeWebhook
if event.Type == "checkout.session.completed" {
    // ... get session ...
    
    // Expand the price to see the LookupKey
    params := &stripe.SubscriptionParams{}
    params.AddExpand("items.data.price")
    sub, _ := subscription.Get(checkoutSession.Subscription.ID, params)

    // Identify the plan purely by the key you set in the dashboard
    lookupKey := sub.Items.Data[0].Price.LookupKey
    
    var regPlan string
    switch lookupKey {
    case "basic_plan":
        regPlan = "basic"
    case "pro_plan":
        regPlan = "pro"
    case "elite_plan":
        regPlan = "elite"
    default:
        regPlan = "visitor"
    }
}

```

### Why Lookup Keys are the "Cleanest" Solution:

1. **No IDs in Code:** You never have to touch a `price_1Se...` string in your Go files again.
2. **Environment Agnostic:** You can use the same lookup key `pro_plan` in both **Test Mode** and **Live Mode**. Your code doesn't care that the actual IDs are different between environments.
3. **Instant Updates:** If you want to change which Price ID represents the "Pro" plan (e.g., a new price for 2025), you just move the `pro_plan` Lookup Key to the new Price in the dashboard. Your code stays exactly the same.

Would you like me to help you refactor your `dashboard.html` to send these string keys (like `basic_plan`) instead of the actual Price IDs?

YES

Using **Lookup Keys** is the most professional way to handle this. It allows you to move away from the `price_` keys entirely in your logic while making your code compatible with both **Stripe Test Mode** and **Live Mode** without changes.

### 1. Update `main.go` Configuration

Since we are no longer using the `price_` IDs, you can simplify your configuration. You only need to define the **Lookup Keys** (the string names) you chose in the Stripe Dashboard.

```go
// main.go

var (
    // These match the "Lookup key" field you filled in the Stripe Dashboard
    PlanLookupBasic = "basic_plan"
    PlanLookupPro   = "pro_plan"
    PlanLookupElite = "elite_plan"
)

```

### 2. Update Checkout Session Creation

When a user selects a plan, your frontend should send the **Lookup Key** (e.g., `"pro_plan"`). Your backend then tells Stripe to find the price associated with that key.

```go
func handleCreateCheckoutSession(w http.ResponseWriter, r *http.Request) {
    // ... parse request body for req.LookupKey ...

    params := &stripe.CheckoutSessionParams{
        SuccessURL: stripe.String(Domain + "/dashboard?session_id={CHECKOUT_SESSION_ID}"),
        CancelURL:  stripe.String(Domain + "/pricing"),
        Mode:       stripe.String(string(stripe.CheckoutSessionModeSubscription)),
        LineItems: []*stripe.CheckoutSessionLineItemParams{
            {
                // IMPORTANT: Use PriceData with LookupKey instead of Price ID
                PriceData: &stripe.CheckoutSessionLineItemPriceDataParams{
                    LookupKey: stripe.String(req.LookupKey),
                },
                Quantity: stripe.Int64(1),
            },
        },
    }

    s, err := checkoutsession.New(params)
    // ... handle redirect ...
}

```

### 3. Update the Webhook to Identify the Plan

In your webhook, you can now check the `LookupKey` attribute of the price to determine which level the user just purchased. This removes the need for any ID-to-Name mapping tables.

```go
// Inside handleStripeWebhook for "checkout.session.completed"
subID := checkoutSession.Subscription.ID

// Expand the price object so we can see the LookupKey
params := &stripe.SubscriptionParams{}
params.AddExpand("items.data.price")
sub, _ := subscription.Get(subID, params)

// Get the key you set in the Dashboard (e.g., "pro_plan")
activeLookupKey := sub.Items.Data[0].Price.LookupKey

var regPlan string
switch activeLookupKey {
case PlanLookupBasic:
    regPlan = "basic"
case PlanLookupPro:
    regPlan = "pro"
case PlanLookupElite:
    regPlan = "elite"
default:
    regPlan = "free"
}

// Proceed to update Firestore with regPlan...

```

### Summary of the Workflow

1. **Dashboard**: You assign the key `pro_plan` to your price.
2. **Code**: You refer only to the string `"pro_plan"`.
3. **Deployment**: When you move from Test to Live mode, you simply assign the same lookup key `pro_plan` to your Live price in Stripe. **You don't have to change a single line of code.**

**Would you like me to update the JavaScript in your `dashboard.html` to handle these lookup keys when a user wants to change their plan?**

YES

To implement this, you’ll need to update your frontend to send the **Lookup Key** string instead of a specific Price ID. This makes your dashboard much cleaner and completely removes those long `price_` strings from your HTML.

### 1. Update the HTML Buttons

In your `dashboard.html` (or wherever your pricing table is), update your buttons or selection logic to use the **Lookup Keys** you created in the Stripe Dashboard (e.g., `basic_plan`, `pro_plan`, `elite_plan`).

```html
<div class="plan-card">
  <h3>Pro Plan</h3>
  <button onclick="createCheckoutSession('pro_plan')">
    Upgrade to Pro
  </button.
</div>

```

### 2. Update the JavaScript `createCheckoutSession`

Refactor your JavaScript function to send this key to your Go backend.

```javascript
async function createCheckoutSession(lookupKey) {
  try {
    const response = await fetch("/api/create-checkout-session", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ lookupKey: lookupKey }), // Send lookupKey
    });

    const data = await response.json();
    if (data.url) {
      window.location.href = data.url;
    }
  } catch (error) {
    console.error("Checkout error:", error);
  }
}

```

### 3. Update the Go Backend Handler

Finally, update `handleCreateCheckoutSession` in `main.go` to use the `LookupKey` field. Stripe will automatically resolve `"pro_plan"` to the correct `price_1Se...` ID for you.

```go
func handleCreateCheckoutSession(w http.ResponseWriter, r *http.Request) {
    var req struct {
        LookupKey string `json:"lookupKey"`
    }
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, "Invalid request", http.StatusBadRequest)
        return
    }

    params := &stripe.CheckoutSessionParams{
        SuccessURL: stripe.String(Domain + "/dashboard?session_id={CHECKOUT_SESSION_ID}"),
        CancelURL:  stripe.String(Domain + "/pricing"),
        Mode:       stripe.String(string(stripe.CheckoutSessionModeSubscription)),
        LineItems: []*stripe.CheckoutSessionLineItemParams{
            {
                // Stripe looks up the actual Price ID based on this string
                PriceData: &stripe.CheckoutSessionLineItemPriceDataParams{
                    LookupKey: stripe.String(req.LookupKey),
                },
                Quantity: stripe.Int64(1),
            },
        },
    }

    s, _ := checkoutsession.New(params)
    json.NewEncoder(w).Encode(map[string]string{"url": s.URL})
}

```

### Why this is the ultimate "No-Price-ID" solution:

* **Code Portability**: You can copy this exact code to a new project, and as long as you set up the same Lookup Keys in a new Stripe account, it will work instantly.
* **Frontend Safety**: If a user inspects your source code, they only see names like `pro_plan` rather than your internal Stripe Price IDs.
* **Version Control**: You no longer need to update your environment variables every time you change a price. Just reassign the "Lookup key" in Stripe to a new Price object.

**Would you like me to help you update the Webhook logic to correctly identify the plan using these Lookup Keys when a payment is successful?**

YES

Refactoring your **dashboard.html** to use **Lookup Keys** is a great move. It makes your frontend code much cleaner because you replace those long, cryptic `price_` IDs with human-readable names like `basic_plan` or `pro_plan`.

### 1. Update the Plan Buttons

In your `dashboard.html`, locate your pricing table or upgrade buttons. You will update the `onclick` handlers to pass the **Lookup Key** string instead of an environment variable or hardcoded ID.

```html
<div class="pricing-grid">
  <div class="plan-card">
    <h3>Basic</h3>
    <button onclick="createCheckoutSession('basic_plan')">Get Basic</button>
  </div>

  <div class="plan-card">
    <h3>Pro</h3>
    <button onclick="createCheckoutSession('pro_plan')">Upgrade to Pro</button>
  </div>
</div>

```

### 2. Update the JavaScript Function

Next, update the `createCheckoutSession` function in the `<script>` section of your `dashboard.html`. This function will now send the `lookupKey` to your Go backend.

```javascript
async function createCheckoutSession(lookupKey) {
  try {
    const response = await fetch("/api/create-checkout-session", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      // We send the readable lookupKey string here
      body: JSON.stringify({ lookupKey: lookupKey }), 
    });

    if (!response.ok) {
      throw new Error("Failed to create checkout session");
    }

    const data = await response.json();
    if (data.url) {
      // Redirect the user to the Stripe Checkout page
      window.location.href = data.url;
    }
  } catch (error) {
    console.error("Checkout Error:", error);
    alert("Could not start checkout. Please try again.");
  }
}

```

### 3. Update the Go Backend Handler

In your `main.go`, update the `handleCreateCheckoutSession` function to read this `lookupKey` and pass it to Stripe using the `PriceData.LookupKey` parameter.

```go
func handleCreateCheckoutSession(w http.ResponseWriter, r *http.Request) {
    var req struct {
        LookupKey string `json:"lookupKey"`
    }
    
    // Decode the JSON request from the frontend
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, "Invalid request", http.StatusBadRequest)
        return
    }

    params := &stripe.CheckoutSessionParams{
        SuccessURL: stripe.String(Domain + "/dashboard?session_id={CHECKOUT_SESSION_ID}"),
        CancelURL:  stripe.String(Domain + "/pricing"),
        Mode:       stripe.String(string(stripe.CheckoutSessionModeSubscription)),
        LineItems: []*stripe.CheckoutSessionLineItemParams{
            {
                // Stripe uses this key to find the actual Price ID for you
                PriceData: &stripe.CheckoutSessionLineItemPriceDataParams{
                    LookupKey: stripe.String(req.LookupKey),
                },
                Quantity: stripe.Int64(1),
            },
        },
    }

    s, err := checkoutsession.New(params)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }

    json.NewEncoder(w).Encode(map[string]string{"url": s.URL})
}

```

### Why This Works Better

* **Security**: No `price_` IDs are visible in your HTML "View Source".
* **Maintenance**: If you want to change the price of the "Pro" plan, you create a new price in Stripe, give it the `pro_plan` lookup key, and **you don't have to touch your code or redeploy**.
* **Consistency**: You use the same lookup keys (`basic_plan`, etc.) in both **Test Mode** and **Live Mode**.

Would you like me to show you how to update your **Stripe Webhook** logic to use these same Lookup Keys to grant the correct user permissions?

YES

To implement the **Stripe Webhook** logic using **Lookup Keys**, you need to ensure your backend can identify which plan a customer bought without ever seeing a `price_` ID.

By using the **Lookup Key**, your code becomes "future-proof." If you ever change your prices in Stripe, you simply move the `basic_plan` or `pro_plan` label to the new price, and your webhook logic remains exactly the same.

### 1. Update the Webhook Handler

In your `handleStripeWebhook` function in `main.go`, you need to **expand** the price object. By default, Stripe only sends the ID. Expanding it allows you to see the `LookupKey` attribute.

```go
// main.go

func handleStripeWebhook(w http.ResponseWriter, r *http.Request) {
    // ... (existing code to read body and verify webhook signature) ...

    switch event.Type {
    case "checkout.session.completed":
        var session stripe.CheckoutSession
        err := json.Unmarshal(event.Data.Raw, &session)
        if err != nil {
            w.WriteHeader(http.StatusBadRequest)
            return
        }

        // 1. Fetch the subscription and EXPAND the price data
        params := &stripe.SubscriptionParams{}
        params.AddExpand("items.data.price")
        
        sub, err := subscription.Get(session.Subscription.ID, params)
        if err != nil {
            log.Printf("Error retrieving subscription: %v", err)
            w.WriteHeader(http.StatusInternalServerError)
            return
        }

        // 2. Identify the plan using the Lookup Key
        // We look at the first item in the subscription
        price := sub.Items.Data[0].Price
        activeLookupKey := price.LookupKey

        var planLevel string
        switch activeLookupKey {
        case "basic_plan":
            planLevel = "basic"
        case "pro_plan":
            planLevel = "pro"
        case "elite_plan":
            planLevel = "elite"
        default:
            planLevel = "free" // Fallback
        }

        // 3. Update your database (e.g., Firestore) with the planLevel
        log.Printf("User %s subscribed to plan: %s", session.Customer.ID, planLevel)
        // updateFirestorePlan(session.Customer.ID, planLevel)
    }

    w.WriteHeader(http.StatusOK)
}

```

### 2. Handle Plan Changes (Upgrades/Downgrades)

When a user changes their plan through the **Stripe Customer Portal**, Stripe sends a `customer.subscription.updated` event. You should use the same logic there to keep your database in sync.

```go
    case "customer.subscription.updated":
        var sub stripe.Subscription
        json.Unmarshal(event.Data.Raw, &sub)

        // Again, expand the price to see the lookup key
        params := &stripe.SubscriptionParams{}
        params.AddExpand("items.data.price")
        fullSub, _ := subscription.Get(sub.ID, params)

        newLookupKey := fullSub.Items.Data[0].Price.LookupKey
        // ... (use the same switch logic as above to update the database) ...

```

### Key Benefits of This Approach

* **Decoupled IDs**: Your database updates are based on the **name** of the plan (`pro_plan`), not the random Stripe ID (`price_1Se...`).
* **Environment Sync**: You can use the same code for **Test** and **Live** environments. Just make sure the "Lookup key" in your Stripe Test dashboard matches the one in your Live dashboard.
* **No Re-deployments**: To change a price, you just swap the lookup key to a new price in the Stripe UI. Your server keeps running without needing an update.

Would you like me to help you create a **Go helper function** that maps these Lookup Keys to user permissions so you can easily restrict features in your app?

YES

To complete your transition to **Lookup Keys**, you need to update your **Stripe Webhook** logic. This ensures that when a payment is successful, your backend can identify the plan name (basic, pro, or elite) based on the human-readable key you set in the Stripe Dashboard, rather than a hardcoded ID.

### 1. Update the Webhook Handler in `main.go`

In your `handleStripeWebhook` function, you must **expand** the `price` object within the subscription. By default, Stripe only sends the ID; expanding it allows you to access the `LookupKey` attribute.

```go
// main.go

func handleStripeWebhook(w http.ResponseWriter, r *http.Request) {
    // ... (Verify webhook signature and get the event) ...

    switch event.Type {
    case "checkout.session.completed":
        var session stripe.CheckoutSession
        if err := json.Unmarshal(event.Data.Raw, &session); err != nil {
            w.WriteHeader(http.StatusBadRequest)
            return
        }

        // 1. Retrieve the subscription and EXPAND the price object
        params := &stripe.SubscriptionParams{}
        params.AddExpand("items.data.price")
        
        sub, err := subscription.Get(session.Subscription.ID, params)
        if err != nil {
            log.Printf("Error fetching subscription: %v", err)
            w.WriteHeader(http.StatusInternalServerError)
            return
        }

        // 2. Identify the plan using the LookupKey
        price := sub.Items.Data[0].Price
        activeLookupKey := price.LookupKey // This will be "pro_plan", etc.

        var planLevel string
        switch activeLookupKey {
        case "basic_plan":
            planLevel = "basic"
        case "pro_plan":
            planLevel = "pro"
        case "elite_plan":
            planLevel = "elite"
        default:
            planLevel = "free"
        }

        // 3. Update Firestore with the new planLevel
        log.Printf("User %s successfully subscribed to: %s", session.Customer.ID, planLevel)
        // updateFirestore(session.Customer.ID, planLevel)

    case "customer.subscription.updated":
        // Handle plan changes from the Stripe Customer Portal
        var sub stripe.Subscription
        json.Unmarshal(event.Data.Raw, &sub)

        params := &stripe.SubscriptionParams{}
        params.AddExpand("items.data.price")
        fullSub, _ := subscription.Get(sub.ID, params)

        newPlan := fullSub.Items.Data[0].Price.LookupKey
        log.Printf("Subscription updated. New plan: %s", newPlan)
        // Sync newPlan to your database...
    }

    w.WriteHeader(http.StatusOK)
}

```

### 2. Benefits of the Final Architecture

By moving to this **Lookup Key** system, your setup is now fully optimized:

* **Code Blindness:** Your Go code and HTML never "see" a `price_` ID. They only deal with logical names like `pro_plan`.
* **Zero-Downtime Price Changes:** If you want to raise the price of "Pro," you create a new price in Stripe and move the `pro_plan` Lookup Key to it. Your website and server will automatically start using the new price without a restart.
* **Unified Testing:** You use the exact same strings (`basic_plan`, `pro_plan`) in both **Test** and **Live** modes. No more switching `.env` variables between environments.

### Next Steps

Now that your backend and frontend are synced via Lookup Keys, would you like me to help you create a **permission middleware** in Go that checks a user's `planLevel` from Firestore before allowing them to access specific features?

YES

To finalize your transition to **Lookup Keys**, we will now update your **Stripe Webhook** logic in `main.go`. This step ensures that when a payment is successful, your backend can automatically identify the plan level (basic, pro, or elite) based on the readable key you set in the Stripe Dashboard.

### 1. Update the Webhook Logic in `main.go`

In your `handleStripeWebhook` function, you must use the `AddExpand` parameter when retrieving the subscription. By default, Stripe only sends a Price ID; expanding the object allows your Go code to see the `LookupKey` string.

```go
// main.go

func handleStripeWebhook(w http.ResponseWriter, r *http.Request) {
    // ... (Verify webhook signature and get the event) ...

    switch event.Type {
    case "checkout.session.completed":
        var session stripe.CheckoutSession
        if err := json.Unmarshal(event.Data.Raw, &session); err != nil {
            w.WriteHeader(http.StatusBadRequest)
            return
        }

        // 1. Fetch the subscription and EXPAND the price object
        // This is crucial to see the 'lookup_key'
        params := &stripe.SubscriptionParams{}
        params.AddExpand("items.data.price")
        
        sub, err := subscription.Get(session.Subscription.ID, params)
        if err != nil {
            log.Printf("Error fetching subscription: %v", err)
            w.WriteHeader(http.StatusInternalServerError)
            return
        }

        // 2. Access the Lookup Key directly
        // No more price_ IDs or environment variable maps needed!
        price := sub.Items.Data[0].Price
        activeLookupKey := price.LookupKey 

        var planLevel string
        switch activeLookupKey {
        case "basic_plan":
            planLevel = "basic"
        case "pro_plan":
            planLevel = "pro"
        case "elite_plan":
            planLevel = "elite"
        default:
            planLevel = "free"
        }

        // 3. Update your database with the readable plan name
        log.Printf("User %s successfully subscribed to plan: %s", session.Customer.ID, planLevel)
        // updateFirestore(session.Customer.ID, planLevel)

    case "customer.subscription.updated":
        // This handles plan changes/upgrades from the Customer Portal
        var sub stripe.Subscription
        json.Unmarshal(event.Data.Raw, &sub)

        params := &stripe.SubscriptionParams{}
        params.AddExpand("items.data.price")
        fullSub, _ := subscription.Get(sub.ID, params)

        newPlan := fullSub.Items.Data[0].Price.LookupKey
        log.Printf("Subscription modified. New Lookup Key: %s", newPlan)
        // Sync the new plan name to your database here...
    }

    w.WriteHeader(http.StatusOK)
}

```

### 2. Benefits of this Final Architecture

By completing this setup, your integration is now using the most secure and maintainable pattern available in Stripe:

* **Human-Readable Code**: Your `switch` statements use words like `"pro_plan"` instead of random strings like `"price_1Se..."`.
* **Environment Agnostic**: You use the exact same strings in your **Test** and **Live** dashboards. You don't have to change your Go code or your `.env` file when you go live.
* **Instant Price Updates**: If you want to change the price of a plan, you simply create a new price in the Stripe Dashboard and move the **Lookup Key** label to that new price. Your server will instantly start granting the correct permissions for the new price without needing a restart.

Would you like me to show you how to create a simple **Go middleware** that checks this `planLevel` in Firestore to protect specific pages or API routes in your app?

To implement a **permission middleware** in Go, you'll create a function that wraps your existing handlers. This function will check the user's `planLevel` in **Firestore** and only allow them to proceed if they have the required subscription.

### 1. Create the Permission Middleware

Add this function to your `main.go`. It takes a `requiredPlan` string and returns an `http.HandlerFunc`.

```go
// main.go

func RequirePlan(requiredPlan string, next http.HandlerFunc) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		// 1. Get user ID from session (replace with your session logic)
		userID := getUserIDFromSession(r)
		if userID == "" {
			http.Redirect(w, r, "/login", http.StatusSeeOther)
			return
		}

		// 2. Fetch the user's current plan from Firestore
		doc, err := firestoreClient.Collection("users").Doc(userID).Get(r.Context())
		if err != nil {
			http.Error(w, "Error checking permissions", http.StatusInternalServerError)
			return
		}

		// 3. Compare current plan vs required plan
		userPlan, _ := doc.DataAt("planLevel")
		
		// Logic: "elite" can access everything, "pro" can access pro/basic, etc.
		if !hasPermission(userPlan.(string), requiredPlan) {
			http.Error(w, "Upgrade required to access this feature", http.StatusForbidden)
			return
		}

		// User has permission, proceed to the actual handler
		next.ServeHTTP(w, r)
	}
}

// Helper to determine plan hierarchy
func hasPermission(userPlan, required string) bool {
	hierarchy := map[string]int{"free": 0, "basic": 1, "pro": 2, "elite": 3}
	return hierarchy[userPlan] >= hierarchy[required]
}

```

### 2. Protect Your Routes

In your `main()` function, wrap the routes you want to secure. Instead of calling `handleEliteFeature` directly, you wrap it with the middleware.

```go
func main() {
    // ... initialization ...

    // Public features
    http.HandleFunc("/api/public-data", handlePublicData)

    // Protected features - only accessible to "pro" and "elite"
    http.HandleFunc("/api/pro-tools", RequirePlan("pro", handleProTools))

    // Elite only features
    http.HandleFunc("/api/elite-analytics", RequirePlan("elite", handleEliteAnalytics))

    log.Fatal(http.ListenAndServe(":8081", nil))
}

```

### 3. Update the UI for Forbidden Access

When a user hits a "Forbidden" (403) error because they don't have the right plan, it's best to redirect them to the pricing page. You can modify the middleware to redirect instead of throwing an error:

```go
if !hasPermission(userPlan.(string), requiredPlan) {
    // Redirect to pricing with a message
    http.Redirect(w, r, "/pricing?error=upgrade_required", http.StatusSeeOther)
    return
}

```

### Why this is the best setup:

* **Centralized Logic**: You define permissions in one place. If you change what "Pro" includes, you only edit the hierarchy map.
* **Database as Source of Truth**: Since your **Stripe Webhook** updates Firestore, this middleware always reflects the user's latest payment status.
* **Clean Handlers**: Your actual feature handlers (like `handleProTools`) don't need to check for subscriptions anymore; they can focus entirely on their specific business logic.

Would you like me to help you update your **Firestore schema** or provide a helper function to display "locked" icons in your HTML based on these plan levels?



* * *
* * *

## Testing

1. In terminal A, run the `stripe listen` command for tunneling your localhost. Keep it running, don't stop it by pressing **Ctrl** + **C**:
```
stripe listen --forward-to localhost:5000/api/stripe-webhook
```
2. Ensure that in `.env` file the `STRIPE_WEBHOOK_SECRET` variable is set to the output of the above command.
3. Open terminal B and launch the containers.
4. In a browser, go to `localhost:5000` and click on **Register** on any of the pricing cards.
5. Fill out the Stripe form. When finished, you will be redirected to your account dashboard, but because you have not yet checked your "email address", you are redirected to login.
6. In terminal B, check the Go API backend logs:
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
2025/12/16 13:35:36 SIMULATING SENDGRID: To johndoe@mail.com, Setup Link: http://localhost:9099/emulator/action?mode=resetPassword&lang=en&oobCode=lezz8pW7350dqq67PT-XZAKuMikvF6HfCEp7jXu8WM5TAxeZfwbYkr&apiKey=fake-api-key
```
The last logs line is what we are looking for. Now run these command:
```
curl "http://localhost:9099/emulator/action?mode=resetPassword&lang=en&oobCode=lezz8pW7350dqq67PT-XZAKuMikvF6HfCEp7jXu8WM5TAxeZfwbYkr&apiKey=fake-api-key&newPassword=fakepassword"
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
