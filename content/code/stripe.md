+++
date = '2025-12-14T00:10:47+01:00'
draft = false
title = 'Stripe'
section = 'code'
weight = 725
+++

## Plans and Prices Creation

Subscription plans and prices can be created in the Stripe dashboard.

## Environment Variables

```
var (
	// ... existing configuration ...

	// Actual Key, Secret and IDs from Stripe Test Mode
	StripeSecretKey = getEnv("TEST_SECRET_KEY", "sk_test_***")
	StripeWebhookSecret = getEnv("TEST_WEBHOOK_SECRET", "whsec_***")
	StripePriceIDBasic = getEnv("TEST_PRICE_ID_BASIC", "price_***")
	StripePriceIDPro = getEnv("TEST_PRICE_ID_PRO", "price_***")
	StripePriceIDElite = getEnv("TEST_PRICE_ID_ELITE", "price_***")

    // Frontend URL on Caddy container's port 5000
	StripeSuccessURLBase = getEnv("TEST_SUCCESS_URL_BASE", "http://localhost:5000/success") 
	StripeCancelURLBase  = getEnv("TEST_CANCEL_URL_BASE", "http://localhost:5000/cancel")
)
```

> [!NOTE]
> In order to obtain the [webhook signing secret](https://docs.stripe.com/webhooks) `TEST_WEBHOOK_SECRET` you have to run:
> `stripe listen --forward-to localhost:8081/api/stripe/webhook`

## User Creation

The example below create a user with a subscription:

```go
package main

import (
	"fmt"
	"log"

	"github.com/stripe/stripe-go/v83"
	"github.com/stripe/stripe-go/v83/customer"
	"github.com/stripe/stripe-go/v83/paymentmethod"
	"github.com/stripe/stripe-go/v83/subscription"
)

func main() {
	// Set your Stripe secret key
	stripe.Key = "sk_test_yourtestkey" // Replace with your actual test key

	// Step 1: Create a new customer
	customerParams := &stripe.CustomerParams{
		Name:  stripe.String("John Doe"),
		Email: stripe.String("johndoe@mail.com"),
	}
	newCustomer, err := customer.New(customerParams)
	if err != nil {
		log.Fatalf("Error creating customer: %v", err)
	}

	fmt.Printf("Step 1. Customer created with ID: %s\n", newCustomer.ID)

	// Step 2: Attach a test payment method to the new customer
	testPaymentMethodID := "pm_card_visa" // This is a test card provided by Stripe

	attachParams := &stripe.PaymentMethodAttachParams{
		Customer: stripe.String(newCustomer.ID), // Attach the payment method to the newly created customer
	}

	attachedPM, err := paymentmethod.Attach(testPaymentMethodID, attachParams)
	if err != nil {
		log.Fatalf("Error attaching payment method: %v", err)
	}
	fmt.Printf("Step 2a. Payment method %s successfully attached to customer\n", attachedPM.ID)

	// Optionally: Set the attached payment method as the default for the customer
	updateParams := &stripe.CustomerParams{
		InvoiceSettings: &stripe.CustomerInvoiceSettingsParams{
			DefaultPaymentMethod: stripe.String(attachedPM.ID),
		},
	}

	_, err = customer.Update(newCustomer.ID, updateParams)
	if err != nil {
		log.Fatalf("Error updating customer with default payment method: %v", err)
	}
	fmt.Println("Step 2b. Default payment method set for customer")

	// Step 3: Subscribe the customer to a plan (using a price ID)
	subscriptionParams := &stripe.SubscriptionParams{
		Customer: stripe.String(newCustomer.ID),
		Items: []*stripe.SubscriptionItemsParams{
			&stripe.SubscriptionItemsParams{
				Price: stripe.String("price_yourpriceid"), // Replace with your actual Price ID
			},
		},
	}

	newSubscription, err := subscription.New(subscriptionParams)
	if err != nil {
		log.Fatalf("Error creating subscription: %v", err)
	}
	fmt.Printf("Step 3. Subscription created with ID: %s\n", newSubscription.ID)
}
```

## User Management

### Delete user

{{< tabs >}}
{{% tab "CLI" %}}

```
stripe customers list --email johndoe@mail.com | grep id
"id": "cus_TbrQMZ2H4uWbe5"
stripe subscriptions list --customer cus_TbrQMZ2H4uWbe5 | grep id
"id": "sub_1SedhGASlOB9XtLrO9AtG8xF"
stripe subscriptions cancel sub_1SedhGASlOB9XtLrO9AtG8xF
stripe customers delete cus_TbrQMZ2H4uWbe5
```

{{% /tab %}}
{{% tab "Go" %}}

```go
// Add the Stripe Client to your App struct
type App struct {
	// ... existing fields ...
	StripeClient *stripe.Client // New field
}

// In NewApp:
// app.StripeClient = stripe.NewClient(os.Getenv("STRIPE_SECRET_KEY"))

func (a *App) handleDeleteAccount(w http.ResponseWriter, r *http.Request) {
	user := a.getAuthenticatedUserFromCookie(r)
	if user == nil {
		http.Error(w, "Unauthorized", http.StatusUnauthorized)
		return
	}

	// 1. Get stripeID from Firestore (assuming it's already loaded or fetched)
    // NOTE: For full safety, you should re-fetch the user data from Firestore 
    // to ensure you have the correct, current stripeID.
	dsnap, err := a.FirestoreClient.Collection("users").Doc(user.UID).Get(r.Context())
	if err != nil || !dsnap.Exists() {
        // ... error handling
	}
    stripeID, _ := dsnap.Data()["stripeID"].(string)

	if stripeID != "" {
		// 2. Cancel active subscriptions (Optional but recommended before deletion)
		subID, err := findActiveSubscriptionID(stripeID) // Using a helper function (defined above)
		if err != nil {
			log.Printf("Error checking subscriptions for %s: %v", stripeID, err)
			// Continue to deletion, as this is non-critical for the delete operation
		}
		if subID != "" {
			if err := cancelSubscription(subID); err != nil { // Using a helper function
				log.Printf("Error canceling subscription %s: %v", subID, err)
				// Continue to customer deletion
			}
		}

		// 3. Delete Stripe Customer
		if err := deleteStripeCustomer(stripeID); err != nil { // Using a helper function
			log.Printf("Error deleting Stripe customer %s: %v", stripeID, err)
			http.Error(w, "Failed to delete Stripe account, please try again.", http.StatusInternalServerError)
			return
		}
	}
	
	// 4. Delete Firestore Doc
	_, err = a.FirestoreClient.Collection("users").Doc(user.UID).Delete(r.Context())
	if err != nil {
		log.Printf("Error deleting Firestore user %s: %v", user.UID, err)
		// NOTE: You might need manual cleanup if this fails, but continue to Firebase delete
	}
	
	// 5. Delete Firebase User
	if err := a.AuthClient.DeleteUser(r.Context(), user.UID); err != nil {
		log.Printf("Error deleting Firebase user %s: %v", user.UID, err)
		http.Error(w, "Account deleted from Stripe/Firestore but failed in Firebase.", http.StatusInternalServerError)
		return
	}
	
	// 6. Log out by clearing session cookie
	http.SetCookie(w, &http.Cookie{Name: "__session", Value: "", MaxAge: -1, Path: "/"})

	w.WriteHeader(http.StatusOK)
	json.NewEncoder(w).Encode(map[string]string{"message": "Account successfully deleted."})
}
```

{{% /tab %}}
{{< /tabs >}}

### List customers

{{< tabs >}}
{{% tab "CLI" %}}
```
stripe customers list
```
{{% /tab %}}
{{% tab "Go" %}}
```
stripe.Key = "{{TEST_SECRET_KEY}}"

params := &stripe.CustomerListParams{}
result := customer.List(params)
```
{{% /tab %}}
{{< /tabs >}}

### Retrieve customer

{{< tabs >}}
{{% tab "CLI" %}}
```
stripe customers retrieve customerid
```
{{% /tab %}}
{{% tab "Go" %}}
```
stripe.Key = "{{TEST_SECRET_KEY}}"

params := &stripe.CustomerParams{}
result, err := customer.Get("customerid", params)
```
{{% /tab %}}
{{< /tabs >}}

### Search customer (Option 1)

{{< tabs >}}
{{% tab "CLI" %}}
```
stripe customers list --email="customer@mail.com"
```
{{% /tab %}}
{{% tab "Go" %}}
```
stripe.Key = "{{TEST_SECRET_KEY}}"

params := &stripe.CustomerListParams{Email: stripe.String("customer@mail.com")}
result := customer.List(params)
```
{{% /tab %}}
{{< /tabs >}}

### Search customer (Option 2)

{{< tabs >}}
{{% tab "CLI" %}}
```
stripe customers search --query="email:'customer@mail.com'"
```
{{% /tab %}}
{{% tab "Go" %}}
```
stripe.Key = "{{TEST_SECRET_KEY}}"

params := &stripe.CustomerSearchParams{
  SearchParams: stripe.SearchParams{Query: "email:'customer@mail.com'"},
}
result := customer.Search(params)
```
{{% /tab %}}
{{< /tabs >}}

### Delete customer

{{< tabs >}}
{{% tab "CLI" %}}
```
stripe customers delete customerid
```
{{% /tab %}}
{{% tab "Go" %}}
```
stripe.Key = "{{TEST_SECRET_KEY}}"

params := &stripe.CustomerParams{}
result, err := customer.Del("customerid", params)
```
{{% /tab %}}
{{< /tabs >}}

### List subscriptions

{{< tabs >}}
{{% tab "CLI" %}}
```
stripe subscriptions list --customer="customerid"
```
{{% /tab %}}
{{% tab "Go" %}}
```
stripe.Key = "{{TEST_SECRET_KEY}}"

params := &stripe.SubscriptionListParams{Customer: stripe.String("customerid")}
result := subscription.List(params)
```
{{% /tab %}}
{{< /tabs >}}

### Cancel subscription

{{< tabs >}}
{{% tab "CLI" %}}
```
stripe subscriptions cancel subscriptionid
```
{{% /tab %}}
{{% tab "Go" %}}
```
stripe.Key = "{{TEST_SECRET_KEY}}"

params := &stripe.SubscriptionCancelParams{}
result, err := subscription.Cancel("subscriptionid", params)
```
{{% /tab %}}
{{< /tabs >}}

### List invoices

{{< tabs >}}
{{% tab "CLI" %}}
```
stripe invoices list --customer="customerid"
```
{{% /tab %}}
{{% tab "Go" %}}
```
stripe.Key = "{{TEST_SECRET_KEY}}"

params := &stripe.InvoiceListParams{Customer: stripe.String("customerid")}
result := invoice.List(params)
```
{{% /tab %}}
{{< /tabs >}}

### Delete invoice

{{< tabs >}}
{{% tab "CLI" %}}
```
stripe invoices delete invoicedid
```
{{% /tab %}}
{{% tab "Go" %}}
```
stripe.Key = "{{TEST_SECRET_KEY}}"

params := &stripe.InvoiceParams{}
result, err := invoice.Del("invoicedid", params)
```
{{% /tab %}}
{{< /tabs >}}

## Miscelanea

> [!NOTE]
> For listing/counting in live mode, add the `--live` flag after the `--limit 200` flag.

### List all customers email addresses

```
stripe customers list --limit 200 | grep email | awk -F': "' '{print $2}' | sed 's/[",]//g'
```


### Count all customers email addresses

```
stripe customers list --limit 200 | grep email | awk -F': "' '{print $2}' | sed 's/[",]//g' | wc -l
```
