---
title: "Troubleshoot"
weight: 75
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
# bookHref: ''
# bookIcon: ''
---

### Restart gcs-emulator and content-uploader
```
podman compose up -d --force-recreate gcs-emulator content-uploader
podman compose logs content-uploader

[upload] Starting content upload...
[upload] Uploading: posts/week0003/index.html (plans: visitor)
ServiceException: 401 Anonymous caller does not have storage.objects.list access to the Google Cloud Storage bucket. Permission 'storage.objects.list' denied on resource (or it may not exist).
```
-------

## x-goo-meta-required-plans

Troubleshooting to check if `upload_posts.sh` is fetching `x-goog-meta-required-plans`:

## Step 1. Dynamic upload and tagging

This command runs a containerized Google Cloud SDK environment to execute a script that dynamically uploads and tags Hugo posts to a local GCS emulator.

```
podman run --rm --network testing_custom_app_network \
  -v "${PWD}:/app" \
  -w /app \
  -e GCS_BUCKET=content \
  -e GCS_EMULATOR_HOST=gcs-emulator:9000 \
  -e STORAGE_EMULATOR_HOST=http://gcs-emulator:9000 \
  google/cloud-sdk:slim \
  /bin/bash ./upload_posts.sh
  
  
[upload] Ensuring bucket exists...
[upload] Starting content upload...
[upload] Uploading gated: week0003 (plans: elite)
[upload] Uploading gated: week0001 (plans: basic,pro)
[upload] Uploading gated: week0002 (plans: basic)
[upload] Warning: HTML file not found for _index, skipping...
[upload] Upload complete.
```

## Step 2. Check content-uploader logs

```
podman logs content-uploader

[upload] Ensuring bucket exists...
[upload] Starting content upload...
[upload] Uploading gated: week0003 (plans: elite)
[upload] Uploading gated: week0001 (plans: basic,pro)
[upload] Uploading gated: week0002 (plans: basic)
[upload] Uploading public: test
[upload] Warning: HTML file not found for _index, skipping...
[upload] Upload complete.
```

## Step 3. Check bucket status

Check name of the bucket. If it's empty or doesn't show `content`, your `content-uploader` failed to create the bucket.

```
curl http://localhost:9000/storage/v1/b | jq

% Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   374  100   374    0     0   359k      0 --:--:-- --:--:-- --:--:--  365k
{
  "kind": "storage#buckets",
  "items": [
    {
      "kind": "storage#bucket",
      "id": "content",
      "defaultEventBasedHold": false,
      "name": "content",
      "versioning": {
        "enabled": false
      },
      "timeCreated": "2025-12-22T14:49:25.766049Z",
      "updated": "2025-12-22T14:49:25.766049Z",
      "location": "US-CENTRAL1",
      "storageClass": "STANDARD",
      "projectNumber": "0",
      "metageneration": "1",
      "etag": "RVRhZw==",
      "locationType": "region"
    }
  ]
}
```

## Step 4. Check if a required plan is set

```
curl -s "http://localhost:9000/storage/v1/b/content/o/posts%2Fweek0003%2Findex.html" | jq

{
  "kind": "storage#object",
  "name": "posts/week0003/index.html",
  "id": "content/posts/week0003/index.html",
  "bucket": "content",
  "size": "6401",
  "contentType": "text/html",
  "crc32c": "gzIlcA==",
  "md5Hash": "p5U9enHQHtTZ6smj9AKPtQ==",
  "etag": "p5U9enHQHtTZ6smj9AKPtQ==",
  "storageClass": "STANDARD",
  "timeCreated": "2025-12-22T14:49:26.007169Z",
  "timeStorageClassUpdated": "2025-12-22T14:49:26.007172Z",
  "updated": "2025-12-22T14:49:26.007172Z",
  "generation": "1766414966007174",
  "metadata": {
    "required-plans": "elite"
  },
  "selfLink": "http://0.0.0.0:9000/storage/v1/b/content/o/posts%2Fweek0003%2Findex.html",
  "mediaLink": "http://0.0.0.0:9000/download/storage/v1/b/content/o/posts%2Fweek0003%2Findex.html?alt=media",
  "metageneration": "1"
}
```


## Step 5. Check gated content implementation

```
curl -I http://localhost:8081/posts/week0003/

cheo@hal:~/code/hugo/project/testing$ curl -I http://localhost:8081/posts/week0003/
HTTP/1.1 403 Forbidden
Content-Type: text/html
Date: Mon, 22 Dec 2025 20:29:59 GMT
Content-Length: 317


podman logs go-backend

2025/12/22 20:28:21 GCS Client configured for emulator at: http://gcs-emulator:9000/storage/v1/ (Project ID strictness disabled)
2025/12/22 20:28:21 Server starting on :8081
2025/12/22 20:29:59 === handleContentGuard START === Request: HEAD /posts/week0003/ (from 10.89.0.6:50646)
2025/12/22 20:29:59 No authenticated user → treating as visitor
2025/12/22 20:29:59 Raw objectPath after TrimPrefix: "posts/week0003/"
2025/12/22 20:29:59 No file extension detected → appending index.html → "posts/week0003/index.html"
2025/12/22 20:29:59 Final GCS object key: "posts/week0003/index.html"
2025/12/22 20:29:59 DEBUG: All Metadata for posts/week0003/index.html: map[required-plans:elite]
2025/12/22 20:29:59 Object "posts/week0003/index.html" exists → required-plans metadata: [elite]
2025/12/22 20:29:59 ACCESS DENIED → User plan "visitor" does not satisfy required plans [elite]
```

## Step 6.

```
hugo -D --minify
podman run --rm --network testing_custom_app_network \
  -v "${PWD}:/app" \
  -w /app \
  google/cloud-sdk:slim \
  /bin/bash ./upload_posts.sh

curl -v http://localhost:8081/posts/week0003
curl -v http://localhost:8081/posts/test
```
