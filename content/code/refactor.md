+++
date = '2025-12-13T11:34:57+01:00'
draft = false
title = 'Refactor'
section = 'code'
weight = 750
+++

## Goals of the refactor

Based on your code, the refactor should:

1. **Remove global routing state** (custom mux)
2. **Separate concerns by file**
3. Turn `contentGuardHandler` into **real middleware**
4. Keep:

   * stdlib `net/http`
   * your Firebase / GCS logic
   * behavior exactly the same

---

## High-level structure (recommended)

```text
main.go
server.go
handlers_api.go
handlers_content.go
middleware.go
content_guard.go
auth.go
storage.go
models.go
config.go
```

Still **one package (`main`)**, no DI framework.

---

## 1. `main.go` (startup only)

```go
func main() {
	ctx := context.Background()

	initFirebase(ctx)
	initGCS(ctx)

	if err := contentGuard.Init(); err != nil {
		log.Fatal(err)
	}

	mux := newServer()

	port := getEnv("PORT", "8081")
	log.Printf("Starting Go server on :%s", port)
	log.Fatal(http.ListenAndServe(":"+port, mux))
}
```

---

## 2. `server.go` (routing)

```go
func newServer() http.Handler {
	mux := http.NewServeMux()

	// Content
	mux.Handle(
		"/posts/",
		contentGuardMiddleware(http.HandlerFunc(handleContent)),
	)

	// API
	mux.HandleFunc("/api/register", handleRegister)
	mux.HandleFunc("/api/sessionLogin", handleSessionLogin)
	mux.HandleFunc("/api/sessionLogout", handleSessionLogout)
	mux.HandleFunc("/api/session", handleSession)

	return mux
}
```

This is **idiomatic, explicit, testable**.

---

## 3. `middleware.go` (ContentGuard as middleware)

### Before

`contentGuardHandler` mixed:

* auth
* authorization
* storage
* proxying

### After

Authorization is middleware; content serving is a handler.

```go
func contentGuardMiddleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		user := getAuthenticatedUserFromCookie(r)
		plan := "visitor"
		if user != nil {
			plan = user.Plan
		}

		path := normalizePath(r.URL.Path)

		if !contentGuard.IsAuthorized(path, plan) {
			w.WriteHeader(http.StatusForbidden)
			w.Header().Set("Content-Type", "text/html")
			fmt.Fprint(w, accessDeniedHTML)
			return
		}

		// store normalized path for downstream handler
		ctx := context.WithValue(r.Context(), contentPathKey{}, path)
		next.ServeHTTP(w, r.WithContext(ctx))
	})
}
```

---

## 4. `handlers_content.go` (content serving only)

```go
func handleContent(w http.ResponseWriter, r *http.Request) {
	path := r.Context().Value(contentPathKey{}).(string)

	objectPath := strings.TrimPrefix(path, "/")
	objectPath = filepath.Join(objectPath, "index.html")

	signedURL, err := generateSignedURL(objectPath)
	if err != nil {
		http.Error(w, "Failed to generate secure link", http.StatusInternalServerError)
		return
	}

	resp, err := http.Get(signedURL)
	if err != nil {
		http.Error(w, "Content fetch error", http.StatusInternalServerError)
		return
	}
	defer resp.Body.Close()

	w.Header().Set("Content-Type", resp.Header.Get("Content-Type"))
	w.Header().Set("Cache-Control", "no-store")

	w.WriteHeader(resp.StatusCode)
	io.Copy(w, resp.Body)
}
```

---

## 5. `handlers_api.go` (unchanged logic, just grouped)

Your existing handlers stay nearly identical:

```go
func handleRegister(w http.ResponseWriter, r *http.Request) { ... }
func handleSessionLogin(w http.ResponseWriter, r *http.Request) { ... }
func handleSessionLogout(w http.ResponseWriter, r *http.Request) { ... }
func handleSession(w http.ResponseWriter, r *http.Request) { ... }
```

Only improvement: **no routing logic inside handlers**.

---

## 6. `content_guard.go` (unchanged, but clean boundary)

Your `ContentGuard` type is already well-designed.
It stays almost exactly as-is.

Only move helpers like regexes and `Init()` here.

---

## 7. `auth.go` (authentication helpers)

```go
func getAuthenticatedUserFromCookie(r *http.Request) *AuthUser {
	// unchanged logic
}
```

Keeps auth logic out of handlers.

---

## 8. `models.go`

```go
type UserRegistration struct { ... }
type AuthUser struct { ... }
```

Pure domain types → no dependencies.

---

## 9. `config.go`

```go
var (
	StaticRoot  = getEnv("STATIC_ROOT", "public")
	ContentRoot = getEnv("CONTENT_ROOT", "../frontend/content/posts")
	GCSBucket   = getEnv("GCS_BUCKET", "content")
	GCSAccessID = getEnv("GCS_ACCESS_ID", "localhost")
)
```

---

## What this refactor buys you

✅ Idiomatic Go structure
✅ Testable routing
✅ True middleware separation
✅ No global mux
✅ No behavior changes
✅ Easy future extraction into packages

---

## What I *didn’t* do (intentionally)

* No frameworks (`chi`, `echo`)
* No dependency injection libraries
* No context overengineering
* No config systems
* No new abstractions

This is still **“plain Go, done right.”**

---

If you want next, I can:

* Provide **one-file-at-a-time diffs**
* Add **httptest examples**
* Convert ContentGuard into composable middleware
* Show how this migrates cleanly to `chi` later

