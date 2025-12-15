+++
date = '2025-12-14T00:10:47+01:00'
draft = false
title = 'Stripe'
section = 'code'
weight = 725
+++

# API

```
var (
	// ... existing configuration ...

	// Actual Key, Secret and IDs from Stripe Test Mode
	StripeSecretKey = getEnv("TEST_SECRET_KEY", "sk_test_51MqjBWASlOB9XtLrMroE76gYUlZHh24ZBwB846GK6emMa9xYtmNaV9hC9GHVvOSEv1blS85wusuZN8EUQ7gk7MU400Vs1SWZ66")
	StripeWebhookSecret = getEnv("TEST_WEBHOOK_SECRET", "whsec_411696825124da63c647bbb56b8584183a39c1a65345af1e9e71a0f3be41bb71")
	StripePriceIDBasic = getEnv("TEST_PRICE_ID_BASIC", "price_1SeJiGASlOB9XtLr4Eh1sFKQ")
	StripePriceIDPro = getEnv("TEST_PRICE_ID_PRO", "price_1SeJhZASlOB9XtLrwhqZJIOz")
	StripePriceIDElite = getEnv("TEST_PRICE_ID_ELITE", "price_1SeJYfASlOB9XtLrAATjV41F")

    // Frontend URL on Caddy container's port 5000
	StripeSuccessURLBase = getEnv("TEST_SUCCESS_URL_BASE", "http://localhost:5000/success") 
	StripeCancelURLBase  = getEnv("TEST_CANCEL_URL_BASE", "http://localhost:5000/cancel")
)
```

> [!NOTE]
> In order to obtain the [webhook signing secret](https://docs.stripe.com/webhooks) `TEST_WEBHOOK_SECRET` you have to run:
> `stripe listen --forward-to localhost:8081/api/stripe/webhook`

# General

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

# Miscelanea

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

