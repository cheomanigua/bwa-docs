---
title: "Infrastructure"
weight: 15
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
# bookHref: ''
# bookIcon: ''
---

## General

### Services

- **Firebase Hosting**: Frontend.
- **Cloud Run**: Go API backend.
- **Firebase Authentication**: User login/register.
- **Firestore**: User account database.
- **Stripe**: Payment and billing platform.

### Development Infraestructure

- Development is in local machine and two local containers.
- **Hugo** static web generator is installed in computer.
- **Firebase Emulator** is running in one container.
- **Go API server** is running in another container.

### Production Infraestructure

- Development in in local machine and two local containers.
- Changes are pushed when needed to GitHub, which triggers GitHub actions.
- Github action runs Hugo static site generator and deploys public directory to **Firebase**.
- Github actions runs Cloud Build, pushes image to Artifact Registry and deploys to **Cloud Run**

## Development Infraestructure

### Containers

##### `compose.dev.yaml`

```yaml
version: "3.9"

services:
  firebase:
    build:
      context: .
      dockerfile: firebase/Containerfile
    image: localhost/firebase-emulator:latest
    container_name: firebase-emulator

    working_dir: /workspace/firebase

    # Required for Fedora/RedHat/SELinux!
    security_opt:
      - label=disable

    volumes:
      - ./firebase:/workspace/firebase:Z
      - ./public:/workspace/public:Z
      - ./.firebase_data:/workspace/.firebase_data:Z

    ports:
      - "4000:4000"   # Emulator UI
      - "5000:5000"   # Hosting
      - "8080:8080"   # Firestore
      - "9099:9099"   # Auth

    command:
      [
        "emulators:start",
        "--project=my-test-project",
        "--only=auth,firestore,hosting,ui",
      ]
      
  # -----------------------------------------------------
  # NEW SERVICE: Go API Backend
  # -----------------------------------------------------
  go-backend:
    image: localhost/test_go-backend:latest
    build:
      context: ./backend
      dockerfile: Containerfile
    container_name: go-api-backend
    ports:
      - "8081:8081" # Expose the API port
    # Ensure it starts after the firebase emulator is up (optional but good practice)
    depends_on:
      - firebase
    environment:
      # Pass the internal network address of the emulator (service name) to the API
      FIREBASE_AUTH_HOST: firebase:9099
      FIREBASE_FIRESTORE_HOST: firebase:8080
    security_opt:
      - label=disable # Required for Fedora/RedHat/SELinux!
```

##### `firebase/Containerfile`

```yaml
FROM node:20-slim

# Install only what firebase-tools actually needs to run the emulators
RUN apt-get update && apt-get install -y --no-install-recommends \
    openjdk-17-jre-headless \
    ca-certificates \
    && rm -rf /var/lib/apt/lists/* \
    && apt-get clean

# Install exact firebase-tools version globally
RUN npm install -g firebase-tools@13.25.0

# Copy only the files the emulator actually needs
COPY firebase/ ./firebase/
COPY public/ ./public/

EXPOSE 4000 5000 8080 9099

ENTRYPOINT ["firebase"]
CMD ["emulators:start", "--project=my-test-project", "--only=auth,firestore,hosting,ui"]
```

##### `backend/Containerfile`

```yaml
# Stage 1: Build the Go application
FROM golang:1.25-alpine AS builder

WORKDIR /app

COPY . .

RUN go mod tidy

# Build the application
RUN CGO_ENABLED=0 GOOS=linux go build -o /go-api ./main.go

# Stage 2: Create the final lean image
FROM alpine:latest

# Install dependencies needed by the Go application
RUN apk add --no-cache ca-certificates

# Set the working directory
WORKDIR /root/

# Copy the built binary from the builder stage
COPY --from=builder /go-api .

# Expose the port the application runs on
EXPOSE 8081

# Command to run the executable
CMD ["./go-api"]
```

## Dev vs Prod

There are three cycles:

{{< tabs >}}

{{% tab "Development" %}} 

- Specific Containerfiles and compose files for development.

{{% /tab %}}

{{% tab "Testing" %}} 

- Specific Containerfiles and compose files for testing.

{{% /tab %}}

{{% tab "Production" %}} 

- Specific Containerfiles and compose files for production.

{{% /tab %}}

{{< /tabs >}}



| # | Area                        | Firebase Emulator (localhost)                                                                                 | Production Firebase + GCP                                                                                       | What breaks / changes in prod |
|---|-----------------------------|---------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|--------------------------------|
| 1 | Authentication              | `FIREBASE_AUTH_EMULATOR_HOST="localhost:9099"` → unlimited users, no email verification, no rate limits       | Real users, email verification, password policies, reCAPTCHA, blocked abusive IPs, rate limits                 | Email verification required, sign-in can be blocked |
| 2 | Firestore Data              | Data lives only in RAM or local files → disappears on shutdown/restart                                       | Data is permanent, global, eventually consistent, billed per read/write/delete                                   | You pay for every operation, data survives forever |
| 3 | Firestore Security Rules    | **Completely ignored** unless you explicitly run `firebase emulators:start --only firestore,auth` with rules | Enforced on every read/write → will reject requests even if your Go Admin SDK bypasses them for client SDKs     | Your frontend will 403 if rules are wrong |
| 4 | Firebase Admin SDK          | Works normally (you use `option.WithoutAuthentication()` or service account)                                   | In Cloud Run you **must** use Application Default Credentials or service account key                          | No `WithoutAuthentication()` in prod |
| 5 | Stripe Webhooks             | You cannot hit `localhost` from Stripe → you need ngrok/tunnel or `stripe listen --forward-to localhost:8081` | Stripe hits your real Cloud Run URL directly                                                                    | Webhook endpoint must be public + HTTPS |
| 6 | Cloud Run URLs              | `http://localhost:8081`                                                                                       | `https://your-service-abc123-uc.a.run.app`                                                                      | CORS, HTTPS, IAP if private |
| 7 | Billing & Quotas           | Unlimited everything for free                                                                                 | Hard quotas + costs: reads, writes, storage, Auth users, Hosting bandwidth, etc.                              | You will hit limits and get billed |
| 8 | Latency & Consistency       | Instant (<5 ms)                                                                                               | 50–400 ms depending on region + eventual consistency                                                            | Slower, occasional stale reads |
| 9 | Functions Triggers          | `functions`, `firestore`, `auth` triggers work locally                                                        | Real Cloud Functions or Cloud Run triggered by events                                                           | Different URLs, cold starts |
|10| Hosting + Custom Domains    | Only serves on `localhost:5000`                                                                               | Real custom domain + SSL + CDN                                                                                  | Real SEO, real performance |
|11| Identity Platform Features  | Not emulated (phone auth, SAML, OIDC, blocking functions)                                                     | Available in prod (extra cost)                                                                                  | Features missing locally |
|12| Service Account Permissions| You can use any service account JSON                                                                          | In GCP the Cloud Run service account must have exact roles (Firestore, Auth Admin, etc.)                      | IAM errors if roles missing |
|13| Logging & Monitoring        | `firebase emulators:start` prints to console                                                                  | Cloud Logging, Error Reporting, Tracing                                                                         | Different debugging experience |

### Quick checklist: “Will it work the same in production?”

| Feature                               | Works identically? | Notes |
|---------------------------------------|---------------------|-------|
| `authClient.VerifyIDToken()`          | Yes              | Same code |
| `firestoreClient.Get/Set` with Admin SDK | Yes              | Same code |
| Registration flow (your Go API)       | Yes              | Same code |
| Stripe Checkout Session creation      | Yes              | Same code |
| Stripe webhooks                       | No               | Must use real endpoint or CLI forwarding |
| Firestore Security Rules              | No               | Ignored locally |
| Email verification / password reset   | No               | Not emulated |
| Rate limiting / abuse protection      | No               | No limits locally |
| Costs                                 | No               | $0 vs real money |

### Bottom line you feel every day

| Local emulator → great for 95 % of development (fast, free, offline)  
| Production → you suddenly pay, hit quotas, deal with webhooks, security rules, IAM, and real users

Keep developing 100 % against the emulator, but **always do a final test on a real staging project** before releasing — especially Stripe webhooks, security rules, and IAM are the three things that most commonly break when you go live.

### Files develop vs prod

Here’s the **exact list of Firebase-related files/configs** that behave differently or can bite you when you switch from **emulators** to **real production** — and what you must check/adapt in each one.

| File / Config                          | What it contains                                                                                 | Emulator vs Production difference                                                                                           | What you MUST change or double-check before going live |
|----------------------------------------|------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------|
| `firebase.json`                        | Hosting config, rewrites, emulator ports, functions triggers                            | In dev you often have `rewrites` to localhost or emulator ports                                                            | Remove or comment out any `rewrites` that point to `localhost` or emulator ports |
| `.firebaserc`                          | Mapping of local aliases → real Firebase project IDs (`default`, `staging`, `prod`)      | `firebase use dev` → points to your emulator project; `firebase use prod` → real project                                  | Make sure `firebase use prod` (or `--project your-prod-id`) is active before deploy |
| `firestore.rules`                      | Security rules                                                                                 | Completely ignored by emulator unless you start it with `--only firestore`; always enforced in prod                       | Test them locally with `firebase emulators:start --only firestore` + real client SDK (not Admin SDK) |
| `firestore.indexes.json`               | Composite indexes                                                                              | Works the same, but you forget to deploy them → queries fail in prod                                                       | Always run `firebase deploy --only firestore:indexes` after adding new ones |
| `storage.rules`                        | Storage security rules (if you use Firebase Storage)                                   | Same story as Firestore rules — ignored locally unless emulated                                                             | Deploy + test with real client SDK |
| `functions/.env` or `functions/.runtimeconfig` | Environment variables for Cloud Functions (if you have any)                            | Often contain `STRIPE_SECRET_KEY=test_...`, `IS_EMULATOR=true`, etc.                                                        | Switch to live Stripe keys, remove emulator flags |
| `src/firebaseConfig.ts` or `src/firebase.js` (web) | `firebase.initializeApp({ apiKey, authDomain, projectId, ... })`                       | Dev version usually points to `localhost` or dev project (`projectId: "demo-dev"`)                                          | Change `projectId`, `authDomain`, `apiKey` to production values |
| Service account JSON (for Go backend)  | `google-credentials.json` or ADC in Cloud Run                                            | In dev you often use `option.WithCredentialsFile("dev-service-account.json")` or `WithoutAuthentication()`                | In production Cloud Run → **delete** `WithoutAuthentication()` and rely on ADC (default service account) |
| Go environment variables               | `FIREBASE_AUTH_EMULATOR_HOST`, `FIRESTORE_EMULATOR_HOST`                                 | Set in dev → must be **absent** in production                                                                               | Remove these two env vars completely in prod |
| `stripe webhook secret`                | Hard-coded or in `.env`                                                                  | Dev: test webhook secret `whsec_test_...`; Prod: live secret `whsec_...`                                                    | Use different secrets per environment |
| `functions/index.js` or Cloud Run code | Any `process.env.IS_EMULATOR` checks                                                     | Often used to skip email verification, etc.                                                                                 | Remove or guard those blocks for production |

### Minimal checklist you run before every production deploy

```bash
# 1. Switch project
firebase use prod        # or --project your-real-project-id

# 2. Remove emulator env vars (Go backend)
unset FIREBASE_AUTH_EMULATOR_HOST
unset FIRESTORE_EMULATOR_HOST

# 3. Deploy everything that can be forgotten
firebase deploy --only hosting,firestore:rules,firestore:indexes,functions

# 4. Verify in browser console
#    firebase.app().name → should show your real project ID, not "demo-dev" or "__emulator__"

# 5. Test one real login + one protected API call
# 6. Test one Stripe test → live mode switch (if you have separate test/live projects)
```

Keep the dev versions of these files in a separate branch or `.env.dev` — never commit production secrets or `WithoutAuthentication()` to main.  
Do these steps and your switch from emulator → production will be painless every single time.
