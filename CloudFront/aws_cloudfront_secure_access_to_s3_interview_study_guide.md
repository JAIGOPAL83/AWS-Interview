# AWS CloudFront & Secure S3 Access — Interview Study Guide

---

## 1. Core Architecture & Setup Overview

To securely serve content from an Amazon S3 bucket using Amazon CloudFront while keeping the bucket private, the industry-standard approach is **Origin Access Control (OAC)**.

### Setup Checklist
1. **S3 Bucket Settings:**
   * Enable **Block *all* public access**.
   * Disable or keep S3 static website hosting off (use the S3 REST API endpoint).
2. **CloudFront Distribution:**
   * **Origin Domain:** Select the S3 REST API endpoint (`bucket-name.s3.amazonaws.com`).
   * **Origin Access:** Select **Origin access control settings (recommended)** and attach an OAC profile.
   * **Viewer Protocol Policy:** Enforce `Redirect HTTP to HTTPS` or `HTTPS Only`.
3. **S3 Bucket Policy:**
   * Restrict access strictly to the CloudFront Service Principal via `AWS:SourceArn`.

---

## 2. Reference Policies & Templates

### S3 Bucket Policy (OAC Read Access)
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowCloudFrontServicePrincipalReadOnly",
      "Effect": "Allow",
      "Principal": {
        "Service": "cloudfront.amazonaws.com"
      },
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::YOUR-BUCKET-NAME/*",
      "Condition": {
        "StringEquals": {
          "AWS:SourceArn": "arn:aws:iam::YOUR-ACCOUNT-ID:distribution/YOUR-DISTRIBUTION-ID"
        }
      }
    }
  ]
}
```

### KMS Key Policy (If using SSE-KMS / Customer Managed Keys)
```json
{
  "Sid": "AllowCloudFrontKMSDecrypt",
  "Effect": "Allow",
  "Principal": {
    "Service": "cloudfront.amazonaws.com"
  },
  "Action": [
    "kms:Decrypt",
    "kms:GenerateDataKey*"
  ],
  "Resource": "*",
  "Condition": {
    "StringEquals": {
      "AWS:SourceArn": "arn:aws:iam::YOUR-ACCOUNT-ID:distribution/YOUR-DISTRIBUTION-ID"
    }
  }
}
```

---

## 3. Top Interview Questions & Detailed Answers

### Q1. What is the fundamental difference between OAC (Origin Access Control) and legacy OAI (Origin Access Identity)?
* **Identity vs. Service Principal:** OAI acts as a legacy "special virtual user". OAC uses **AWS SigV4 (Signature Version 4)** request signing and leverages the standard IAM Service Principal (`cloudfront.amazonaws.com`) evaluated via `AWS:SourceArn` or `AWS:SourceAccount`.
* **KMS Encryption:** OAI does **not** support SSE-KMS-encrypted S3 buckets. OAC natively supports KMS-encrypted buckets because SigV4 carries the signature context required by KMS.
* **HTTP Methods:** OAI only supports `GET` and `HEAD`. OAC supports all HTTP verbs (`GET`, `HEAD`, `POST`, `PUT`, `DELETE`).
* **Global Availability:** OAC is supported across all AWS regions, including opt-in GovCloud and China regions.

---

### Q2. How would you systematically troubleshoot a `403 Forbidden` error when CloudFront attempts to access an S3 bucket via OAC?
1. **Bucket Policy Condition:** Ensure `AWS:SourceArn` in the S3 bucket policy exactly matches the CloudFront distribution ARN (`arn:aws:iam::<ACCOUNT_ID>:distribution/<DISTRIBUTION_ID>`).
2. **KMS Policy:** If SSE-KMS is enabled, verify the key policy permits `kms:Decrypt` to `cloudfront.amazonaws.com` with the correct `AWS:SourceArn`.
3. **S3 Endpoint Type:** Verify the CloudFront Origin uses the S3 **REST API domain** (`bucket-name.s3.amazonaws.com`), NOT the S3 static website hosting endpoint.
4. **Object Ownership:** Ensure uploaded objects belong to the bucket owner (or enforce *Bucket Owner Enforced* setting on the S3 bucket).
5. **Requested Object Path:** Verify the object key exists and that Default Root Object is correctly configured if requesting root paths (`/`).

---

### Q3. How do you decide between Signed URLs and Signed Cookies for private content distribution?
* **Signed URLs:**
  * Ideal for single files or media streams targeting client applications that do not maintain HTTP cookie state (e.g., custom media players, mobile native apps).
  * URL format includes signature parameters in query strings.
* **Signed Cookies:**
  * Ideal for granting access to multiple files, entire folders, or multi-segment streams (HLS/DASH) without requiring signature parameters on every chunk/segment URL.
  * Preserves clean application URLs.

---

### Q4. Compare Cache Invalidation vs. File Versioning for updating edge content.
* **Invalidation (`create-invalidation`):**
  * Manually purges cached objects at Edge Locations before TTL expiration.
  * **Trade-off:** First 1,000 paths/month are free; additional paths incur charges. Propagation takes time and does not clear client-side browser cache if cache headers allowed local storage.
* **File Versioning / Cache Busting:**
  * Modifies file paths or query parameters (e.g., `app.v2.js` or `style.css?v=2`).
  * **Trade-off:** Recommended for CI/CD pipelines. Zero invalidation costs, instantaneous cache updates for clients, and simple rollback paths.

---

### Q5. How do you prevent users from bypassing CloudFront and hitting origins directly?
* **For S3 Origins:** Block public access on S3 and enforce **OAC** in the bucket policy.
* **For Application Load Balancers (ALB):**
  1. Configure CloudFront to inject a custom secret header (e.g., `X-Custom-Header: secret-value`).
  2. Configure ALB Listener Rules to inspect headers and reject traffic without the token.
  3. Alternatively, restrict ALB ingress rules to CloudFront IP ranges (via AWS IP Prefix Lists) or use AWS WAF.

---

### Q6. Explain the roles of Edge Locations, Regional Edge Caches (REC), and Origin Shield.
* **Edge Location:** Points of Presence (PoPs) closest to end-users running caching, AWS WAF, and edge compute (CloudFront Functions, Lambda@Edge).
* **Regional Edge Cache (REC):** Sits between Edge Locations and origins. Has larger cache capacities to absorb cache misses from smaller Edge Locations.
* **Origin Shield:** An optional, centralized caching layer placed in a chosen AWS Region. Consolidates requests across all global RECs into a single request stream to lower origin load and maximize cache hit ratios.

---

## 4. Quick Comparison Matrix

| Feature | CloudFront OAC | CloudFront OAI (Legacy) |
| :--- | :--- | :--- |
| **Signing Protocol** | AWS SigV4 | Legacy Signing |
| **KMS Support (SSE-KMS)** | Yes | No |
| **Allowed Methods** | GET, HEAD, POST, PUT, DELETE | GET, HEAD |
| **IAM Principal Type** | Service Principal (`cloudfront.amazonaws.com`) | Legacy Canonical User ID |
| **Condition Keys** | `AWS:SourceArn`, `AWS:SourceAccount` | N/A |