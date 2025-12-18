---
title: "CLI"
weight: 80
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
# bookHref: ''
# bookIcon: ''
---

# CLI

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
