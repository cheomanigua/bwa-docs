+++
date = '2025-11-27T20:16:24+01:00'
draft = false
title = 'SRS'
section = 'diagrams'
weight = 510
+++


### Registration + Payment Flow (Textual)

```mermaid
flowchart TD
    A[Pricing Page] --> B[Click Plan Card]
    B --> C[Registration Form<br/>Plan pre-selected]
    C --> D[Submit]
    D --> E[Payment Provider]
    E -->|Success| F[Create Account + Activate Plan]
    E -->|Failed| G[Show Error + Retry]
```
### Content Access Flow

```mermaid
flowchart TD
    I1[User opens post] --> J1[Check subscription plan]
    J1 -->|Access Allowed| K1[Show post]
    J1 -->|Access Denied| L1[Show Upgrade Plan message]
```
---

## Use Case Diagram

```mermaid
flowchart LR
    V1[Visitor] --> A2[View Plans]
    V1 --> B2[Open Registration - Plan Preselected]
    V1 --> C2[Register with Plan]

    C2 --> Pay1[Payment Provider]
    Pay1 --> H2[Create Account]
    Pay1 --> I2[Activate Subscription]

    U1[User] --> D2[Access Allowed Posts]
    U1 --> E2[Manage Subscription]

    Admin1[Admin] --> F2[Create/Edit Plans]
    Admin1 --> G2[Create/Edit Posts]
```
