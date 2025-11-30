+++
date = '2025-11-28T19:43:49+01:00'
draft = false
title = 'Components'
section = 'diagrams'
weight = 520
+++

## System Components Overview


```mermaid
flowchart TB
    subgraph Client ["Visitor / Member (Browser)"]
        Browser[Web Browser]
    end

    subgraph Firebase ["Firebase"]
        Hosting[Firebase Hosting]
        Auth[Firebase Auth]
        Firestore[Cloud Firestore]
    end

    subgraph CloudRun ["Cloud Run – Go API"]
        API[Go Backend]
    end

    subgraph External ["External Services"]
        Stripe[Stripe Billing]
        Hugo[Hugo Content\npublic/posts/*.md]
    end

    %% Numbered connections only
    Browser -->|1| Hosting
    Hosting -->|2| Browser
    Browser -->|3| Auth
    Auth -->|4| Browser

    Browser -->|5| API
    API -->|6| Auth

    API <-->|7| Firestore

    API -->|8| Stripe
    Stripe -->|9| API
    API -->|10| Firestore

    Browser -->|11| API
    API -->|12| Hugo
    API -->|13| Firestore
    API -->|14| Browser

    %% Styling
    classDef client fill:#34A853,color:white
    classDef firebase fill:#FFA611,color:white
    classDef backend fill:#4285F4,color:white
    classDef external fill:#EA4335,color:white

    class Browser client
    class Hosting,Auth,Firestore firebase
    class API backend
    class Stripe,Hugo external
```

- **1**. Browser loads static site from Firebase Hosting
- **2**. Hosting serves HTML + JS + public posts
- **3**. User logs in / registers using Firebase Auth (client SDK)
- **4**. Firebase Auth returns ID token to browser
- **5**. Frontend calls any Go API endpoint with Authorization: Bearer <id-token>
- **6**. Go API verifies token using Firebase Admin SDK
- **7**. Go API reads/writes user profile, plan, Stripe IDs in Firestore
- **8**. Go API creates Checkout Session or Billing Portal session in Stripe
- **9**. Stripe sends webhooks (invoice.paid, subscription.deleted, etc.) to Go API
- **10**. Go API updates subscription status and plan in Firestore
- **11**. User requests a protected post (e.g. /post/premium-article)
- **12**. Go API reads the Markdown/HTML file and parses front-matter category
- **13**. Go API checks user’s current plan against post category
- **14**. Go API returns full post (200) or 403 upgrade message


## Protected Content (Posts)


```mermaid
flowchart LR
    subgraph FE["Frontend Browser"]
        U["User Requests Post"]
    end
    subgraph API["Go API Backend"]
        A1["Receive Request with Firebase Token"]
        A2["Get User Plan from Firestore"]
        A3["Read Hugo Content File"]
        A4["Parse Front Matter Category"]
        A5{"Does Plan Match Category?"}
    end
    subgraph FS["Firestore"]
        D1["User Profile and Plan"]
    end
    subgraph HG["Hugo Content"]
        C1["Markdown or HTML Post"]
    end
    U --> A1
    A1 --> A2
    A1 --> A3
    A2 --> D1
    A2 --> A5
    A3 --> C1
    A3 --> A4
    A4 --> A5
    A5 -->|Yes| R1["Return Full Post"]
    A5 -->|No| R2["Return 403 Upgrade Message"]
```
