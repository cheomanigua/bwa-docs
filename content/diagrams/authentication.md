+++
date = '2025-11-28T21:11:22+01:00'
draft = false
title = 'Authentication'
section = 'diagrams'
weight = 530
+++

## Signup workflow


#### Signup workflow

```mermaid
flowchart LR
    subgraph FIREBASE
        direction TB
        hosting
        auth
        firestore
    end
    subgraph Cloud Run
        goapi(Go API)
    end

    visitor <-- "7 / 1" --> hosting
    hosting <-- "6 / 2" --> goapi
    goapi <-- "4 / 3" --> auth
    goapi -- "5" --> firestore

```

- **1**: User opens app and fills registration form
- **2**: Frontend calls Go API
- **3**: Go API create user in Firebase Auth
- **4**: Auth return UID
- **5**: Go API saves profile in Firestore
- **6**: Go API returns success
- **7**: Frontend shows "registered - now log in"

```mermaid
sequenceDiagram
    participant User as User (Browser)
    participant Frontend as Frontend<br/>(Firebase Hosting)
    participant GoAPI as Go API<br/>(Cloud Run)
    participant Auth as Firebase Auth
    participant FS as Firestore

    User->>Frontend: 1
    Frontend->>GoAPI: 2
    GoAPI->>Auth: 3
    Auth-->>GoAPI: 4
    GoAPI->>FS: 5
    GoAPI-->>Frontend: 6
    Frontend-->>User: 7
```


## Login workflow

```mermaid
flowchart LR
    subgraph FIREBASE
        direction TB
        hosting
        auth
        firestore
    end
    subgraph Cloud Run
        goapi(Go API)
    end

    user -- "1" --> hosting
    hosting <-- "3 / 2" --> auth
    hosting <-- "7 / 4" --> goapi
    goapi <-- "6 / 5" --> auth
    goapi <-- "8" --> firestore
```

- **1**: User opens website and fills email + password
- **2**: Frontend calls Firebase JS SDK → signInWithEmailAndPassword()
- **3**: Firebase Auth returns ID token to browser
- **4**: Frontend calls your Go API with Authorization: Bearer <token>
- **5**: Go API calls authClient.VerifyIDToken()
- **6**: Firebase Auth confirms token is valid → returns UID
- **7**: Go API returns protected data
- **8**: (optional) Go API reads/writes Firestore as admin

```mermaid
sequenceDiagram
    participant User as User (Browser)
    participant Frontend as Frontend<br/>(Firebase Hosting)
    participant Auth as Firebase Auth
    participant GoAPI as Go API<br/>(Cloud Run)
    participant FS as Firestore

    User->>Frontend: 1
    Frontend->>Auth: 2
    Auth-->>Frontend: 3
    Frontend->>GoAPI: 4
    GoAPI->>Auth: 5
    Auth-->>GoAPI: 6
    opt
        GoAPI->>FS: 8
        FS-->>GoAPI: 8
    end
    GoAPI-->>Frontend: 7
```
