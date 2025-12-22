---
title: "Deploy"
weight: 100
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
# bookHref: ''
# bookIcon: ''
---

> [!WARNING]
> WORK IN PROGRESS. This is only at the planification stage. It is not approved yet.

You can selectively deploy different parts of your Hugo-generated site to **different targets** (GCS for restricted posts, Firebase Hosting for the rest). GitHub Actions gives you the flexibility to run arbitrary scripts, so you can do exactly what you described. Let’s break it down.

---

## 0. Manually

In a production environment, you shift from using local scripts to using the **Google Cloud CLI (`gcloud`)** and your **Go backend** to handle the heavy lifting.

To upload a new post from your computer and ensure it is protected by your "Content Guard," follow these steps:

### 1. Upload the Content to GCS

Instead of using a local emulator URL, you will use the `gsutil` command (part of the Google Cloud SDK) to push files to your production bucket.

```bash
# Upload the file
gcloud storage cp ./index.html gs://your-production-bucket-name/posts/week0004/index.html

```

### 2. Set the Access Metadata

This is the most critical step. In production, your Go API looks at the **GCS Custom Metadata** to decide who can see the file. You must tag the file with the required plan.

```bash
# Set metadata so only 'pro' and 'elite' users can see it
gcloud storage objects update gs://your-production-bucket-name/posts/week0004/index.html \
    --metadata=required-plans=pro

```

### 3. Verification Flow

Once the file is in the bucket with the metadata:

1. **Public User:** Tries to visit `yoursite.com/posts/week0004/`. Your Go API (Cloud Run) checks the GCS metadata, sees `pro` is required, and returns a **401 Unauthorized** or redirects to login.
2. **Basic User:** Logs in. Go API sees they are `basic`, checks metadata (`pro`), and returns a **403 Forbidden**.
3. **Pro/Elite User:** Logs in. Go API sees their plan satisfies the `pro` requirement and streams the file from GCS to their browser.

---

### Comparison: Local vs. Production

| Feature | Local Emulator | Production |
| --- | --- | --- |
| **Storage** | GCS Emulator (Port 9000) | Google Cloud Storage Bucket |
| **Logic** | Go API (Localhost) | Go API (Cloud Run) |
| **Auth** | Firebase Auth Emulator | Firebase Auth (Live) |
| **Upload Tool** | `curl` / `upload_posts.sh` | `gcloud storage` or CI/CD Pipeline |

### Recommended: Automate with a "Publisher" Script

In a real-world scenario, you shouldn't do this manually every time. You can create a simple `publish.sh` on your computer:

```bash
#!/bin/bash
# Usage: ./publish.sh [folder_name] [required_plan]
# Example: ./publish.sh week0004 pro

FOLDER=$1
PLAN=$2
BUCKET="gs://your-production-bucket-name"

echo "Publishing $FOLDER with plan $PLAN..."

# 1. Upload
gcloud storage cp -r ./$FOLDER "$BUCKET/posts/"

# 2. Apply Metadata to all files in that folder
gcloud storage objects update "$BUCKET/posts/$FOLDER/*" --metadata=required-plans=$PLAN

echo "Done! Post is now live and protected."

```

Perfect. Since you are using the **TOML** format (the `+++` delimiters) and your plans are listed inside the `categories` array, we can make the GitHub Action even smarter.

It will dynamically look at that `categories` list, find the "highest" plan mentioned, and use that to tag your files in Google Cloud Storage.

### How the "Zero-Touch" Workflow looks now:

1. **You edit:** `vi content/posts/week0004.md`
2. **You set the category:** Just add `'basic'`, `'pro'`, or `'elite'` to that array.
3. **You push:** `git push`
4. **GitHub Action:**
* Reads the `categories` line.
* Finds the most restrictive plan (e.g., if both `basic` and `pro` are there, it chooses `pro`).
* Builds the HTML with Hugo.
* Uploads and tags the GCS object with `required-plans=pro`.



### The Updated GitHub Action Logic

We will use a **service account key** stored as a "Secret" in GitHub. This is the simplest way to give the GitHub Action permission to manage your GCS bucket without manual setup on your machine.

Here is the specific part of the script that handles your `+++` front matter dynamically. It uses a bit of "text magic" (`sed` and `awk`) to grab the categories without needing any extra tools.

Create `.github/workflows/deploy.yml` in your repository. This script handles the Hugo build, extracts the plan from your `+++` front matter, and updates Google Cloud.

```yaml
name: Deploy Protected Posts
on:
  push:
    paths:
      - 'content/posts/**'

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4
        with:
          fetch-depth: 2

      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v2

      - name: Build Site
        run: hugo --minify

      - name: Authenticate to Google Cloud
        uses: google-github-actions/auth@v2
        with:
          credentials_json: '${{ secrets.GCP_SA_KEY }}' # This is the "Key" we will add to GitHub

      - name: Set up Cloud SDK
        uses: google-github-actions/setup-gcloud@v2

      - name: Detect and Deploy
        run: |
          # 1. Identify which Markdown file changed
          CHANGED_FILES=$(git diff --name-only HEAD^ HEAD | grep '^content/posts/')

          for file in $CHANGED_FILES; do
            slug=$(basename "$file" .md)
            
            # 2. Extract the 'categories' array from your +++ front matter
            # This logic looks for the highest tier in your list
            CATEGORIES=$(grep "categories =" "$file")
            
            if [[ $CATEGORIES == *"elite"* ]]; then
              PLAN="elite"
            elif [[ $CATEGORIES == *"pro"* ]]; then
              PLAN="pro"
            elif [[ $CATEGORIES == *"basic"* ]]; then
              PLAN="basic"
            else
              PLAN="free"
            fi

            echo "🚀 Deploying $slug with Plan: $PLAN"

            # 3. Upload the generated HTML folder to your Production Bucket
            gcloud storage cp -r "public/posts/$slug" gs://your-production-bucket/posts/

            # 4. Apply the Metadata tag so your Go API can protect it
            gcloud storage objects update "gs://your-production-bucket/posts/$slug/**" \
              --metadata=required-plans=$PLAN
          done
```

### Why this works for you:

* **Matches your format:** It specifically looks for the `categories = [...]` line you already use.
* **Smart Selection:** If you have `categories = ['basic', 'pro']`, the script is smart enough to see `pro` and set the security to that level.
* **No new files:** You don't have to change how you use Hugo at all.

---

## 1. Your Workflow Concept

1. **Author writes posts locally using Hugo**

   * Restricted content → `content/posts`
   * Unrestricted content → other pages (Home, Login, Contact, etc.)
2. **Push to GitHub**
3. **GitHub Actions workflow runs**

   * Builds Hugo site → generates `public/` folder
   * Splits the output:

     * `/posts` → **GCS Bucket**
     * Everything else → **Firebase Hosting**
   * Uploads each to the right platform
4. Users visit site:

   * Unrestricted pages served directly via Firebase Hosting
   * Restricted pages served via GCS with **signed URLs** (your Go server)

---

## 2. Feasibility

* **Firebase Hosting:** GitHub Actions has a ready-made action [`FirebaseHostingDeploy`](https://github.com/marketplace/actions/deploy-to-firebase-hosting) to deploy your site.
* **GCS Upload:** You can use [`google-github-actions/upload-cloud-storage`](https://github.com/google-github-actions/upload-cloud-storage) or simply a `gsutil cp` command in your workflow to push files to a bucket.

So yes, GitHub can **send `public/posts` to GCS** and **everything else to Firebase Hosting** — it just needs the workflow to know which paths to deploy.

---

## 3. Suggested GitHub Actions Workflow

```yaml
name: Build and Deploy Hugo Site

on:
  push:
    branches:
      - main

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    steps:
      # 1. Checkout repository
      - name: Checkout repository
        uses: actions/checkout@v3

      # 2. Setup Hugo
      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v3
        with:
          hugo-version: '0.111.3'  # Your Hugo version

      # 3. Build Hugo site
      - name: Build Hugo
        run: hugo --minify

      # 4. Move restricted posts temporarily
      - name: Move restricted posts
        run: |
          mkdir -p ../restricted_posts_temp
          mv public/posts ../restricted_posts_temp/ || true

      # 5. Deploy unrestricted pages to Firebase Hosting
      - name: Deploy to Firebase Hosting
        uses: FirebaseExtended/action-hosting-deploy@v0
        with:
          repoToken: "${{ secrets.GITHUB_TOKEN }}"
          firebaseServiceAccount: "${{ secrets.FIREBASE_SERVICE_ACCOUNT }}"
          projectId: your-firebase-project-id
          channelId: live

      # 5. Set up Google Cloud SDK
      - name: Setup Google Cloud SDK
        uses: google-github-actions/setup-gcloud@v2
        with:
          service_account_key: "${{ secrets.GCP_SA_KEY }}"
          project_id: your-gcp-project-id

      # 7. Sync restricted posts to GCS using gsutil rsync (delete removed files)
      - name: Sync restricted posts to GCS
        run: |
          gsutil -m rsync -d -r ../restricted_posts_temp/posts gs://your-gcs-bucket/posts

      # 7. Restore posts folder
      - name: Restore posts folder
        run: mv ../restricted_posts_temp/posts public/ || true
```

**Notes:**

* Before deploying to Firebase Hosting, we temporarily **move the `public/posts` folder** out so that Firebase only sees unrestricted pages.
* Then we upload `public/posts` to **GCS Bucket** using the official GitHub Action.
* `GCP_SA_KEY` is your **service account JSON** stored as a GitHub secret.

1. Added `-d` to `gsutil rsync`:

```bash
gsutil -m rsync -d -r local_dir gs://bucket/path
```

* `-d` → deletes files in GCS that **don’t exist locally**.
* `-r` → recursive copy.
* `-m` → multi-threaded for speed.

2. Workflow ensures:

* Restricted posts are **always in sync** with your local Hugo content.
* Deleted posts in your repo automatically disappear from GCS.
* Firebase Hosting deployment only sees unrestricted pages.

---

## 4. Optional Enhancements

1. **Atomic deployment:** Use temporary folders for both Firebase and GCS so deployments don’t interfere.
2. **Signed URLs:** Your Go server already generates signed URLs for restricted content.
3. **Incremental deployment:** You could deploy only changed files to GCS with `gsutil rsync`.

---

## 5. Security Considerations

* Keep your Firebase and GCS service accounts as **GitHub secrets**.
* Only allow **signed URL access** for restricted files in GCS.
* Make sure Firebase Hosting never sees your restricted content.

---

### Bottom line

GitHub Actions **can handle both deployments**:

* `public/posts` → **GCS Bucket**
* `public/*` (everything else) → **Firebase Hosting**

It just requires:

* Splitting the generated `public/` folder
* Using the right deploy actions (`Firebase Hosting`, `GCS Upload`)
* Storing credentials securely in GitHub Secrets

Perfect! Adding the `-d` flag to `gsutil rsync` will **delete any files in GCS that no longer exist locally**, keeping your bucket perfectly in sync with your Hugo-generated `public/posts` folder. Here's the updated workflow:

---

### 🔹 Optional Enhancements

* Add a **commit hash or timestamp** to the logs so you know exactly which Hugo build was deployed.
* Add caching for Hugo themes (`~/.hugo`) to speed up builds.
* Optionally, split Firebase Hosting and GCS deploys into **parallel jobs** for faster CI.


## What to build

This is a critical question for setting up your Continuous Integration/Continuous Deployment (CI/CD) pipeline on GitHub Actions.

When you push a change to your repository, you should **only build and deploy the components that have changed or rely on the change.** Building the entire stack every time is inefficient and slow.

Here is a breakdown of what needs to be built or deployed, depending on the change:

### 1. New Post or Frontend Change (Hugo Source)

If you create a new post (`frontend/content/posts/new-post.md`), edit a layout (`frontend/layouts/`), or change an asset (`frontend/static/`):

| Component Affected | Action in CI/CD Pipeline | Notes |
| :--- | :--- | :--- |
| **Frontend Static Files (`frontend/public/`)** | **Hugo Build** | This is the primary action. Hugo must run to generate the new static HTML, CSS, and JS files (including your new post's HTML). |
| **Firebase Hosting** | **Deployment** | The generated `frontend/public/` directory must be deployed to Firebase Hosting. |
| **Cloud Storage** | **Deployment** | The protected posts (or the entire `public` directory content) must be uploaded to the Cloud Storage bucket (the production version of the GCS Emulator). |
| **Go API Permissions Map** | **Configuration Build** | **YES.** Since a new post with a new access requirement exists, the script that generates the `permissions.json` file **must** run. |
| **Go API (Cloud Run)** | **Re-deployment** | **YES.** The Go container needs to be rebuilt with the new `permissions.json` file packaged inside it, and then deployed to Cloud Run. The code itself didn't change, but its critical configuration file did. |

### 2. Backend Go Code Change

If you edit the Go API code (e.g., changing `backend/main.go` to fix a bug or change the authentication logic):

| Component Affected | Action in CI/CD Pipeline | Notes |
| :--- | :--- | :--- |
| **Go API (Cloud Run)** | **Container Build & Deployment** | The Go code needs to be recompiled, the new binary placed in a Docker image, and that image deployed to Cloud Run. |
| **Frontend** | **None** | The Hugo static files and permissions map are unchanged. |

### 3. CI/CD Pipeline Structure (GitHub Actions)

Your GitHub Actions workflow should use **path filtering** to trigger only the necessary jobs, saving time and resources.



| Triggering Path | Actions to Trigger |
| :--- | :--- |
| `frontend/content/**` | **Full Deployment Job:** 1. Build Hugo. 2. **Generate `permissions.json`**. 3. Build/Deploy Go API (with new config). 4. Deploy to Firebase Hosting and Cloud Storage. |
| `frontend/layouts/**` | **Frontend Deployment Job:** 1. Build Hugo. 2. Deploy to Firebase Hosting. (No permission map change needed). |
| `backend/**` | **Backend Deployment Job:** 1. Build Go API image. 2. Deploy to Cloud Run. (No Hugo build or config generation needed). |
| `firebase/**` | **Infrastructure/Rules Deployment:** Deploy Firestore rules, etc. (Less frequent). |

### Summary

> **If I create a new post and push to GitHub, will GitHub Actions build the entire stack, or only will trigger Hugo build?**

It should trigger more than just the Hugo build.

A new post requires:
1.  **Hugo Build:** To generate the static HTML file.
2.  **Permissions Configuration Build:** To update the `permissions.json` file with the new post's access rules.
3.  **Go API (Cloud Run) Re-deployment:** To load the new `permissions.json`.
4.  **Content Deployment:** To push the new post's static HTML to Cloud Storage and the updated static assets to Firebase Hosting.


## How building affects running services

That's a very practical question about managing application uptime during development and deployment. The answer is **it depends entirely on your process**, but the goal, especially in production, is to ensure minimal to zero service interruption.

In your specific **Development Environment** (using `podman-compose`):

### 1. Building the Go Backend Container

When you rebuild the `backend` container (e.g., after changing `main.go` or packaging a new `permissions.json`), the services are affected in this way:

* **During the Build Phase:** The existing `go-backend` container remains running and serving requests. The build process simply compiles the new code and creates a *new* image. **No services are affected.**
* **During the Restart/Redeploy Phase (`podman-compose up -d`):** When you run `up -d` to deploy the new image, `podman-compose` will:
    1.  Stop the *old* `go-backend` container.
    2.  Remove the old container.
    3.  Start the *new* `go-backend` container.

    During the tiny window between step 1 and step 3, the `go-backend` service is unavailable. Any requests for `/api/*` or `/posts/*` that hit Caddy during this time will receive a **502 Bad Gateway** error from the Caddy reverse proxy because its upstream target (`backend:8081`) is momentarily down.
    * **Impact:** **Minor interruption** for dynamic requests (API calls). Static files and GCS content will still be served by Caddy/GCS Emulator.

### 2. Building the Caddy Reverse Proxy Container

If you change the `reverse-proxy/Containerfile` (less common) or the `Caddyfile` and rebuild the `caddy` container:

* **During the Build Phase:** The existing `caddy-server` continues to run. **No services are affected.**
* **During the Restart/Redeploy Phase:** Similar to the Go backend, the old Caddy container is stopped and the new one is started.
    * **Impact:** **Brief complete interruption.** Since Caddy is the public entry point on `:5000`, the entire application will be inaccessible during the few seconds it takes to swap the container.

### 3. Hugo Build (Local, Baremetal)

If you create a new post and run `hugo` on your local machine:

* **Impact:** **None on the running containers.** The Hugo build process simply writes new files to the `frontend/public` directory, which is volume-mounted read-only into your `caddy` and `gcs-emulator` containers.
    * Caddy and the GCS Emulator will immediately begin reading the newly updated static files. No restart is required for the content to refresh.

---

### Comparison with Production (Cloud Run, Firebase)

In production, the goal is to achieve **Zero Downtime Deployment**, which requires more robust tooling provided by the cloud providers.

| Service | Production Deployment Method | Downtime Status |
| :--- | :--- | :--- |
| **Cloud Run (Go API)** | **Rolling Deployment.** When a new image is deployed, Cloud Run gradually brings up the new revision and shifts traffic to it **before** tearing down the old revision. This ensures at least one instance is always running to serve traffic. | **Zero Downtime.** |
| **Firebase Hosting** | **Atomic Deployment.** The new version of your static site (including all assets) is uploaded, and the traffic switch (the "atomic cutover") happens instantaneously only once all files are successfully verified. | **Zero Downtime.** |
| **Cloud Storage** | **Upload/Sync.** Uploading the new protected content to the bucket is generally an atomic operation for individual files. The service remains available throughout the process. | **Zero Downtime.** |

In summary, your local development process with basic `podman-compose` results in **minor downtime** for the containers you restart. In production, using Cloud Run and Firebase Hosting will virtually eliminate any downtime for your users.
