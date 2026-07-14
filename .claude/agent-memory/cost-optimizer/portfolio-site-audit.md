---
name: portfolio-site-cost-audit
description: Cost optimization findings for static portfolio website infrastructure (S3+CloudFront+Terraform state)
metadata:
  type: project
---

# Portfolio Site Cost Audit (2026-07-14)

Infrastructure: Static HTML/CSS portfolio deployed to AWS with S3 + CloudFront CDN, GitHub OIDC CI/CD, Terraform state backend (commented out).

## High-Impact Findings

### 1. CloudFront Price Class Too Generous
**File:** `terraform/main.tf` line 99
**Current:** `price_class = "PriceClass_200"`
**Issue:** PriceClass_200 includes all regions except the most expensive (PriceClass_All). For a low-traffic portfolio site serving primarily from one region (ap-south-1), this is overkill.
**Recommendation:** Downgrade to `PriceClass_100` (North America + Europe only)
**Estimated Savings:** 50-60% reduction in CloudFront costs (e.g., from ~$50/month to ~$20/month at typical static site traffic)
**Why:** This is a portfolio site, not a globally-distributed app. Traffic likely concentrates in 1-2 regions. PriceClass_100 covers 99% of typical use cases for cost-conscious sites.

### 2. S3 Versioning Without Lifecycle Cleanup
**File:** `terraform/main.tf` lines 38-44
**Current:** Versioning enabled, no lifecycle rule to delete noncurrent versions
**Issue:** Every time content is updated, old versions remain in S3 and incur storage charges indefinitely. For a portfolio site that gets occasional updates, this wastes 10-20% of storage costs over time.
**Recommendation:** Either:
  - **Option A (Safest):** Add lifecycle rule to expire noncurrent versions after 30 days:
    ```
    resource "aws_s3_bucket_lifecycle_configuration" "site" {
      bucket = aws_s3_bucket.site.id
      rule {
        id     = "expire-noncurrent-versions"
        status = "Enabled"
        noncurrent_version_expiration {
          noncurrent_days = 30
        }
      }
    }
    ```
  - **Option B (Maximum Savings):** Disable versioning entirely (only if no rollback needed):
    ```
    versioning_configuration {
      status = "Suspended"
    }
    ```
**Estimated Savings:** Option A: 10-15% S3 storage cost reduction. Option B: ~30-40% (but loses version history).
**Why:** Portfolio content rarely needs rollback. 30-day retention balances safety with cost.

## Medium-Impact Findings

### 3. CloudFront 404 Cache TTL Too Low
**File:** `terraform/main.tf` lines 119-124
**Current:** `error_caching_min_ttl = 10` (seconds)
**Issue:** 10 seconds means CloudFront refetches the 404 error response from S3 every 10 seconds. While unlikely for a static portfolio with stable URLs, this increases origin requests slightly.
**Recommendation:** Increase to `error_caching_min_ttl = 300` (5 minutes minimum)
**Estimated Savings:** Negligible (< 1% cost reduction), but reduces unnecessary origin requests.
**Why:** Static sites don't have dynamic 404 rates. 5-minute TTL is safe and cacheable.

## Low-Impact Findings

### 4. DynamoDB State Locking (Future)
**File:** `terraform/backend.tf` lines 19-27 (currently commented out)
**Current:** Backend block commented; will use provisioned capacity by default when uncommented
**Issue:** When backend is enabled, DynamoDB will default to provisioned billing (1 WCU, 1 RCU minimum = ~$25-30/month), which is overkill for state locking.
**Recommendation:** When enabling backend, add billing mode:
  ```
  resource "aws_dynamodb_table" "terraform_lock" {
    ...
    billing_mode = "PAY_PER_REQUEST"  # ← Add this
    ...
  }
  ```
**Estimated Savings:** PAY_PER_REQUEST costs ~$0.25-1/month (only charges for actual requests); provisioned costs ~$25/month.
**Why:** State operations are infrequent (occasional terraform plan/apply). On-demand is 20-50x cheaper.

### 5. S3 Incomplete Multipart Upload Cleanup (Missing)
**File:** `terraform/main.tf` (no lifecycle rule exists)
**Current:** No cleanup policy for abandoned multipart uploads
**Issue:** If uploads ever fail mid-way, orphaned parts accumulate and incur storage costs indefinitely. Unlikely for static site uploads, but a best practice.
**Recommendation:** Add to lifecycle configuration:
  ```
  rule {
    id     = "cleanup-incomplete-uploads"
    status = "Enabled"
    abort_incomplete_multipart_upload {
      days_after_initiation = 7
    }
  }
  ```
**Estimated Savings:** Negligible for this workload (< $0.01/month), but prevents future surprises.
**Why:** Free insurance against accidental multipart upload leaks.

## No-Cost Items (Already Optimized)

- **CloudFront Compression:** Enabled (line 115) ✓ Reduces data transfer cost
- **CloudFront Cache Policy:** Using AWS-managed `Managed-CachingOptimized` ✓ Optimal for static sites
- **S3 Storage Class:** Standard (correct for frequently accessed content) ✓
- **No unnecessary logging:** S3/CloudFront logging not enabled (saves cost) ✓
- **No WAF:** Not attached to CloudFront (not needed for static site) ✓
- **IPv6 Support:** Enabled but no additional cost ✓

## Summary

**Actionable Changes Ranked by Cost Impact:**

| Priority | Finding | Impact | Effort |
|----------|---------|--------|--------|
| 1 | Downgrade CloudFront PriceClass to 100 | 50-60% CloudFront cost ↓ | 1 line |
| 2 | Add S3 lifecycle rule for noncurrent versions | 10-15% S3 cost ↓ | 10 lines |
| 3 | Increase 404 error cache TTL | <1% cost ↓ | 1 line |
| 4 | Plan on-demand DynamoDB for state backend | 20-50x cheaper when enabled | 1 line (future) |
| 5 | Add incomplete multipart upload cleanup | <0.1% cost ↓ | 5 lines |

**Total Estimated Annual Savings:** $400-600+ (depending on traffic volume and current CloudFront/S3 spend)
