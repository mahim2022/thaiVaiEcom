# Docker Build and Runtime Issues - Solution Guide

## Issue Summary
The Next.js storefront failed to build in Docker due to backend API unavailability during the build phase, and then failed at runtime with a `DYNAMIC_SERVER_USAGE` error.

## Root Causes

### 1. Build-Time API Dependency
- Next.js pages with `generateStaticParams()` were attempting to fetch from the Medusa backend during Docker build
- The backend wasn't available yet (container still building/starting)
- This caused `ECONNREFUSED` errors and build failures

### 2. .next Directory Overwritten by Volume Mount
- Docker successfully built the `.next` directory inside the image
- However, the volume mount `./my-medusa-storefront:/app` overwrote the entire `/app` directory
- The `.next` folder (build output) was lost, causing "production build not found" error at runtime

### 3. Environment Variable Mismatch
- `.env.local` had `MEDUSA_BACKEND_URL=http://localhost:9000`
- Inside Docker containers, the correct URL is `http://medusa:9000` (using Docker network service name)
- This could cause fetch failures at runtime

### 4. Incompatible Static/Dynamic Rendering
- Pages returned empty arrays from `generateStaticParams()` (due to build failures)
- Pages then tried to render dynamically but used static rendering logic
- This caused `DYNAMIC_SERVER_USAGE` errors

## Solution Steps

### Step 1: Add Healthcheck to Medusa Service
- Added TCP healthcheck to medusa service in `docker-compose.yml` that checks if port 9000 is accepting connections
- Configured healthcheck with 10 second intervals, 5 retries, and 30 second startup grace period
- Updated storefront's `depends_on` to use `condition: service_healthy` instead of just service_started
- This ensures storefront waits for medusa to fully start before attempting to build

### Step 2: Make Build-Time Data Fetching Resilient
- Wrapped all `generateStaticParams()` functions in try-catch blocks in three page files:
  - Categories page (`src/app/[countryCode]/(main)/categories/[...category]/page.tsx`)
  - Collections page (`src/app/[countryCode]/(main)/collections/[handle]/page.tsx`)
  - Products page (`src/app/[countryCode]/(main)/products/[handle]/page.tsx`)
- When fetch fails during build, catch the error, log a warning, and return empty array
- This allows the build to succeed and pages to render dynamically at runtime instead

### Step 3: Preserve .next Directory
- Added `/app/.next` to the volumes list in storefront service in `docker-compose.yml`
- This creates a named volume for the `.next` directory so it doesn't get overwritten by the source code mount
- The built assets from Docker image are now preserved when the container runs

### Step 4: Enable Dynamic Rendering
- Added `export const dynamic = "force-dynamic"` to all three page files (categories, collections, products)
- This directive tells Next.js to render these pages dynamically on every request
- Prevents static rendering conflicts when data isn't available during build time

## Key Takeaways

1. **Build vs Runtime Context**: Docker build happens in isolation without external dependencies - pages must gracefully handle missing backend data during build

2. **Volume Mounts**: Exclude build directories (`.next`, `.dist`, etc.) from being overwritten by volume mounts by creating separate volume entries

3. **Service Dependencies**: Use healthchecks with `condition: service_healthy` to ensure services are actually ready, not just started

4. **Dynamic Routes**: Pages that depend on runtime data should use `export const dynamic = "force-dynamic"` to avoid static/dynamic rendering conflicts

5. **Error Handling**: Always wrap external API calls in try-catch blocks during static generation functions like `generateStaticParams()`

## Common Errors and Fixes

| Error | Cause | Fix |
|-------|-------|-----|
| `ECONNREFUSED` during build | Backend not available at build time | Add try-catch to `generateStaticParams()` and return empty array |
| `production build not found` at runtime | `.next` directory overwritten by volume | Add `/app/.next` to volumes exclusion list |
| `DYNAMIC_SERVER_USAGE` error | Static page trying to use dynamic features | Add `export const dynamic = "force-dynamic"` |
| Cannot reach localhost:8000 | Backend URL uses localhost instead of service name | Use `http://medusa:9000` inside containers |
| Storefront starts before medusa ready | Simple `depends_on` doesn't check readiness | Use `condition: service_healthy` with healthcheck |





Some pages returning issues 

Issue Analysis and Resolution
What Caused the Issue
Root Cause: Stale Cache Problem

The application was experiencing two primary problems stemming from cached files that became outdated:

First Problem - Vite Module Cache:
When Medusa's admin panel builds, Vite (the build tool) creates optimized JavaScript chunks and stores references to these chunks in a cache directory inside the Docker container. When code changes happen or containers are rebuilt without clearing this cache, Vite continues to reference old chunk files that no longer exist at their expected locations. This causes 404 errors when the browser tries to load dynamically imported modules.

Second Problem - Browser Cache:
Web browsers aggressively cache JavaScript files to improve performance. When the application is rebuilt and generates new chunk files with different hashes or filenames, the browser continues serving old cached versions. These old files contain outdated import paths that point to non-existent resources, causing module loading failures.

Third Problem - Authentication State:
The 401 Unauthorized errors occurred because authentication session tokens were also cached by the browser, but the server had reset its session storage after the rebuild. The browser was sending expired or invalid session cookies.

What Solved It
Complete Solution - Multi-Layer Cache Clearing:

Docker Level:
Running the system-wide Docker prune removed all unused images, containers, networks, and volumes. This ensured no stale build artifacts persisted at the Docker layer.

Container Level:
Deleting the Vite cache directory inside the running container and restarting forced Vite to regenerate all module dependencies from scratch with fresh references. The .medusa directory, which stores compiled admin UI assets, was also cleared.

Browser Level:
Using incognito mode bypassed all browser caching mechanisms entirely, forcing the browser to fetch fresh copies of all JavaScript files, stylesheets, and other assets directly from the server. This broke the connection to any stale cached resources.

Why Incognito Mode Was the Final Key
Regular browser cache clearing or hard refresh wasn't sufficient because some browsers implement multiple cache layers including service workers, HTTP cache, and memory cache. Incognito mode starts with a completely clean slate with zero cached data, zero cookies, and zero stored authentication tokens. This proved that the server was serving correct files and the problem was purely client-side caching.

Prevention Going Forward
This type of issue typically occurs after development changes, Docker rebuilds, or aborted builds that leave the cache in an inconsistent state. The Vite build cache inside containers can become desynchronized from actual file locations when builds are interrupted or when switching between different code versions. Regular cache clearing during active development prevents accumulation of these inconsistencies.




What Went Wrong
The TypeScript compiler was unable to find the Medusa.js framework modules and their type declarations. The errors indicated that four specific module imports could not be resolved:

@medusajs/framework/types
@medusajs/framework/utils
@medusajs/framework/workflows-sdk
@medusajs/medusa/core-flows
Root Cause
Even though your project's package.json file correctly listed all the required Medusa.js dependencies (version 2.13.1), the actual packages were never installed in the node_modules folder. While a node_modules directory existed, the @medusajs packages inside it were completely missing. This meant TypeScript had no type definitions to reference when checking your seed.ts file.

The Fix
We resolved this in three steps:

Cleaned the node_modules folder - Removed the incomplete node_modules directory entirely to ensure a fresh start without any corrupted or partial installations.

Installed dependencies with Yarn - Since your project has both npm and yarn lock files, we used yarn install to download and install all the required packages. Yarn completed the installation successfully in about one minute, downloading all Medusa.js framework packages and their dependencies.

Restarted the TypeScript server - After the packages were installed, we restarted VS Code's TypeScript language server so it would recognize and load the newly available type definitions from the installed packages.

After these steps, TypeScript could successfully find all the required modules and type definitions, and all errors in your seed.ts file were resolved.

---

# VPS Deployment Readiness Analysis

**Date:** February 16, 2026  
**Status:** ⚠️ CRITICAL ISSUES FOUND - Must fix before VPS deployment

## 🔴 Critical Issues (MUST FIX)

### 1. Hardcoded localhost URLs in Environment Files
**Severity:** CRITICAL  
**Files Affected:**
- [.env](..env#L1-L2) - `STORE_CORS`, `ADMIN_CORS`, `AUTH_CORS` all use `http://localhost:8000` and `http://localhost:9000`
- [my-medusa-storefront/.env.local](my-medusa-storefront/.env.local#L2) - `MEDUSA_BACKEND_URL=http://localhost:9000`
- [my-medusa-storefront/.env.local](my-medusa-storefront/.env.local#L8) - `NEXT_PUBLIC_BASE_URL=http://localhost:8000`
- [docker-compose.yml](docker-compose.yml#L71) - `NEXT_PUBLIC_BASE_URL=http://localhost:8000`

**Impact:** The application will not be accessible from the internet. CORS will block all external requests.

**Solution:**
```bash
# Replace localhost with your actual VPS domain/IP
# In .env:
STORE_CORS=https://yourdomain.com,http://yourdomain.com:8000
ADMIN_CORS=https://yourdomain.com,http://yourdomain.com:9000
AUTH_CORS=https://yourdomain.com,http://yourdomain.com:9000

# In my-medusa-storefront/.env.local:
MEDUSA_BACKEND_URL=http://medusa:9000  # Inside Docker (correct)
NEXT_PUBLIC_BASE_URL=https://yourdomain.com  # Public URL

# In docker-compose.yml:
NEXT_PUBLIC_BASE_URL=https://yourdomain.com  # Or use env variable
```

### 2. Weak Security Secrets
**Severity:** CRITICAL  
**Files Affected:**
- [.env](..env#L6-L7) - `JWT_SECRET=supersecret`, `COOKIE_SECRET=supersecret`
- [medusa-config.ts](medusa-config.ts#L30-L31) - Hardcoded fallback to "supersecret"
- [my-medusa-storefront/.env.local](my-medusa-storefront/.env.local#L20) - `REVALIDATE_SECRET=supersecret`

**Impact:** Anyone can forge authentication tokens and session cookies. Your application can be easily compromised.

**Solution:**
```bash
# Generate strong secrets (Linux/Mac/Git Bash):
openssl rand -base64 32

# Or in PowerShell:
[System.Convert]::ToBase64String((1..32 | ForEach-Object { Get-Random -Maximum 256 }))

# Update .env with unique strong secrets for each:
JWT_SECRET=<strong-random-secret-1>
COOKIE_SECRET=<strong-random-secret-2>
REVALIDATE_SECRET=<strong-random-secret-3>
```

### 3. Hardcoded Admin Redirect URL
**Severity:** HIGH  
**File:** [my-medusa-storefront/src/lib/data/onboarding.ts](my-medusa-storefront/src/lib/data/onboarding.ts#L8)

**Issue:**
```typescript
redirect(`http://localhost:7001/a/orders/${orderId}`)
```

**Impact:** Onboarding redirects will fail, pointing to localhost instead of your VPS domain.

**Solution:**
```typescript
// Use environment variable
const ADMIN_URL = process.env.NEXT_PUBLIC_ADMIN_URL || 'http://localhost:7001'
redirect(`${ADMIN_URL}/a/orders/${orderId}`)
```

### 4. Default Database Credentials
**Severity:** HIGH  
**File:** [docker-compose.yml](docker-compose.yml#L8-L10)

**Issue:**
```yaml
POSTGRES_USER: postgres
POSTGRES_PASSWORD: postgres
```

**Impact:** Anyone can access your production database if they reach your VPS.

**Solution:**
```yaml
# Use environment variables from .env file
POSTGRES_USER: ${POSTGRES_USER:-medusa_user}
POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}  # Must be set in .env
```

### 5. HTTPS Not Configured
**Severity:** HIGH  
**Current State:** Application only configured for HTTP

**Impact:** 
- No encryption for data in transit
- Passwords and tokens sent in plaintext
- Browsers will show "Not Secure" warnings
- Modern APIs (like payment processors) may refuse to work

**Solution:**
- Set up Nginx reverse proxy with SSL/TLS
- Use Let's Encrypt for free SSL certificates
- Configure docker-compose to expose only 80/443, not internal ports

## 🟡 Important Issues (STRONGLY RECOMMENDED)

### 6. Missing .env.production File
**Severity:** MEDIUM  
**Impact:** No clear separation between dev and production configs

**Solution:**
Create `.env.production` and `.env.local.production` files with production-specific values, and load them on the VPS.

### 7. Exposed Internal Ports
**Severity:** MEDIUM  
**File:** [docker-compose.yml](docker-compose.yml#L11-L26)

**Issue:**
```yaml
ports:
  - "5432:5432"  # PostgreSQL exposed to world
  - "6379:6379"  # Redis exposed to world
  - "9000:9000"  # Backend exposed
  - "5173:5173"  # Vite dev server exposed
```

**Impact:** Internal services directly accessible from the internet, increasing attack surface.

**Solution:**
```yaml
# Only expose nginx/reverse proxy on 80/443
# Remove port mappings for postgres, redis
# Use Docker networks for inter-service communication
```

### 8. Development Mode in Production
**Severity:** MEDIUM  
**File:** [docker-compose.yml](docker-compose.yml#L39)

**Issue:**
```yaml
environment:
  - NODE_ENV=development
```

**Impact:** Running in dev mode has verbose logging, disabled optimizations, and exposes debug information.

**Solution:**
```yaml
environment:
  - NODE_ENV=production
```

### 9. Volume Mounts Not Production-Ready
**Severity:** MEDIUM  
**File:** [docker-compose.yml](docker-compose.yml#L45-L47)

**Issue:**
```yaml
volumes:
  - .:/server  # Mounts entire source code
  - /server/node_modules
```

**Impact:** Unnecessary file watching, source code exposed in container, not using built artifacts.

**Solution:**
For production, remove source code mounts or use a production Dockerfile that copies built assets only.

### 10. Fallback to HTTPS in env.ts
**Severity:** MEDIUM  
**File:** [my-medusa-storefront/src/lib/util/env.ts](my-medusa-storefront/src/lib/util/env.ts#L2)

**Issue:**
```typescript
return process.env.NEXT_PUBLIC_BASE_URL || "https://localhost:8000"
```

**Impact:** Fallback URL uses HTTPS with localhost which will fail. Should be HTTP for localhost or production URL.

**Solution:**
Remove fallback or make it sensible:
```typescript
if (!process.env.NEXT_PUBLIC_BASE_URL) {
  throw new Error('NEXT_PUBLIC_BASE_URL must be set')
}
return process.env.NEXT_PUBLIC_BASE_URL
```

## ⚪ Minor Issues (RECOMMENDED)

### 11. Commented-out Code in Dockerfile
**Severity:** LOW  
**File:** [my-medusa-storefront/Dockerfile](my-medusa-storefront/Dockerfile#L5-L14)

**Issue:**
```dockerfile
# RUN npm ci  # Commented out
RUN npm install  # Less efficient than npm ci
```

**Recommendation:** Use `npm ci` for production builds (faster, more reliable).

### 12. Missing Environment Variable Validation
**Severity:** LOW  
**Issue:** No validation for required env vars except `NEXT_PUBLIC_MEDUSA_PUBLISHABLE_KEY`

**Recommendation:** Add validation for all critical env vars (DB URL, secrets, CORS, etc.)

### 13. Docker Healthcheck Uses nc Command
**Severity:** LOW  
**File:** [docker-compose.yml](docker-compose.yml#L53)

**Issue:**
```yaml
test: ["CMD", "nc", "-z", "localhost", "9000"]
```

**Impact:** `nc` might not be available in Alpine images.

**Recommendation:** Use `wget` or `curl` for more reliable healthchecks:
```yaml
test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:9000/health"]
```

## 📋 Pre-Deployment Checklist

Before deploying to VPS:

- [ ] Replace all localhost URLs with actual domain/IP
- [ ] Generate and set strong secrets for JWT, COOKIE, and REVALIDATE
- [ ] Change database credentials
- [ ] Set NODE_ENV=production
- [ ] Set up SSL/TLS certificates (Let's Encrypt)
- [ ] Configure reverse proxy (Nginx/Traefik)
- [ ] Close unnecessary exposed ports
- [ ] Create production environment files
- [ ] Set up firewall rules on VPS
- [ ] Configure proper CORS origins
- [ ] Set up automated backups for PostgreSQL
- [ ] Configure log aggregation/monitoring
- [ ] Test application in production mode locally first
- [ ] Set up domain DNS records
- [ ] Configure rate limiting
- [ ] Enable Docker restart policies

## 🚀 Recommended VPS Setup Architecture

```
Internet
    ↓
[Nginx Reverse Proxy] :80, :443 (SSL/TLS)
    ↓
[Storefront Container] :8000 (internal)
    ↓
[Medusa Backend] :9000 (internal)
    ↓
[PostgreSQL] :5432 (internal only)
[Redis] :6379 (internal only)
```

Only Nginx should be exposed to the internet. All internal services should communicate via Docker networks without exposed ports.

## 📚 Additional Steps for Production

1. **Monitoring:** Set up monitoring (Grafana, Prometheus, or cloud provider monitoring)
2. **Backups:** Automated daily backups of PostgreSQL to external storage
3. **CI/CD:** Set up automated deployment pipeline
4. **Error Tracking:** Integrate Sentry or similar for error tracking
5. **Rate Limiting:** Protect API endpoints from abuse
6. **CDN:** Use CDN for static assets
7. **Redis Persistence:** Configure Redis persistence for sessions
8. **Docker Resource Limits:** Set memory and CPU limits for containers

---

**Summary:** Found 13 VPS deployment issues - 5 critical, 5 important, 3 minor. All critical issues must be fixed before going to production on VPS.

---

# my-medusa-storefront Code Analysis

**Date:** February 16, 2026  
**Status:** ⚠️ BREAKING ISSUES FOUND IN STOREFRONT CODE

## 🔴 Critical Breaking Issues

### 1. Missing Error Boundary Pages
**Severity:** CRITICAL  
**Location:** my-medusa-storefront/src/app/

**Issue:**
- No `error.tsx` files at any route level
- No global error handling for Next.js App Router
- Unhandled errors will crash the entire application

**Impact:** When any server component throws an error (database connection lost, API failure, etc.), the entire page will white screen with no user-friendly error message.

**Solution:**
Create error boundaries at critical levels:
```tsx
// my-medusa-storefront/src/app/error.tsx (global)
'use client'

export default function Error({
  error,
  reset,
}: {
  error: Error & { digest?: string }
  reset: () => void
}) {
  return (
    <div className="flex flex-col items-center justify-center min-h-screen p-4">
      <h2 className="text-2xl font-bold mb-4">Something went wrong!</h2>
      <p className="text-gray-600 mb-4">{error.message}</p>
      <button 
        onClick={() => reset()}
        className="px-4 py-2 bg-blue-500 text-white rounded"
      >
        Try again
      </button>
    </div>
  )
}

// Also create at:
// my-medusa-storefront/src/app/[countryCode]/error.tsx
// my-medusa-storefront/src/app/[countryCode]/(main)/error.tsx
// my-medusa-storefront/src/app/[countryCode]/(checkout)/error.tsx
```

### 2. Cart Page Missing Null Check
**Severity:** HIGH  
**File:** [my-medusa-storefront/src/app/[countryCode]/(main)/cart/page.tsx](my-medusa-storefront/src/app/[countryCode]/(main)/cart/page.tsx#L13)

**Issue:**
```tsx
const cart = await retrieveCart().catch((error) => {
  console.error(error)
  return notFound()  // notFound() is called but doesn't return null/undefined
})
// cart could still be null if catch doesn't throw
return <CartTemplate cart={cart} customer={customer} />
```

**Impact:** If `retrieveCart()` returns `null`, `cart` will be `null` but `notFound()` doesn't prevent execution. Template expects cart to have properties.

**Solution:**
```tsx
export default async function Cart() {
  const cart = await retrieveCart()

  if (!cart) {
    return notFound()
  }

  const customer = await retrieveCustomer()
  return <CartTemplate cart={cart} customer={customer} />
}
```

### 3. Middleware Missing MEDUSA_BACKEND_URL Fallback
**Severity:** CRITICAL  
**File:** [my-medusa-storefront/src/middleware.ts](my-medusa-storefront/src/middleware.ts#L4-L19)

**Issue:**
```typescript
const BACKEND_URL = process.env.MEDUSA_BACKEND_URL
// ...
if (!BACKEND_URL) {
  throw new Error(
    "Middleware.ts: Error fetching regions. Did you set up regions..."
  )
}
```

**Impact:** If `MEDUSA_BACKEND_URL` is missing, middleware throws error and **blocks ALL requests** to the storefront. Site becomes completely inaccessible.

**Solution:**
Better to fail gracefully:
```typescript
const BACKEND_URL = process.env.MEDUSA_BACKEND_URL

if (!BACKEND_URL) {
  console.error("CRITICAL: MEDUSA_BACKEND_URL not set")
  // Return error page instead of crashing middleware
  return new NextResponse(
    "Site configuration error. Please contact support.",
    { status: 500 }
  )
}
```

### 4. Unhandled Payment Session Errors
**Severity:** HIGH  
**File:** [my-medusa-storefront/src/modules/checkout/components/payment-button/index.tsx](my-medusa-storefront/src/modules/checkout/components/payment-button/index.tsx#L70-L80)

**Issue:**
```tsx
const handlePayment = async () => {
  setSubmitting(true)

  if (!stripe || !elements || !card || !cart) {
    setSubmitting(false)
    return  // Silent failure - user doesn't know what went wrong
  }

  await stripe.confirmCardPayment(session?.data.client_secret as string, {
    // Using 'as string' without null check
```

**Impact:** 
- Silent failures when payment elements aren't loaded
- Type assertion `as string` can fail if session data is undefined
- No user feedback for failures

**Solution:**
```tsx
const handlePayment = async () => {
  setSubmitting(true)

  if (!stripe || !elements || !card || !cart) {
    setErrorMessage("Payment system not ready. Please refresh the page.")
    setSubmitting(false)
    return
  }

  if (!session?.data?.client_secret) {
    setErrorMessage("Invalid payment session. Please try again.")
    setSubmitting(false)
    return
  }

  await stripe.confirmCardPayment(session.data.client_secret, {
    // ... rest of payment logic
  })
}
```

### 5. Missing Environment Variable Validation
**Severity:** HIGH  
**File:** [my-medusa-storefront/check-env-variables.js](my-medusa-storefront/check-env-variables.js)

**Issue:**
Only validates `NEXT_PUBLIC_MEDUSA_PUBLISHABLE_KEY` but critical variables are missing:
- `MEDUSA_BACKEND_URL` - Required for SDK and middleware
- `NEXT_PUBLIC_BASE_URL` - Used in metadata
- `NEXT_PUBLIC_DEFAULT_REGION` - Middleware fallback

**Impact:** App fails at runtime instead of build time when critical env vars are missing.

**Solution:**
```javascript
const requiredEnvs = [
  {
    key: "NEXT_PUBLIC_MEDUSA_PUBLISHABLE_KEY",
    description: "Publishable API key for store"
  },
  {
    key: "MEDUSA_BACKEND_URL",
    description: "Backend URL (e.g., http://localhost:9000)"
  },
  {
    key: "NEXT_PUBLIC_BASE_URL",
    description: "Storefront public URL"
  },
  {
    key: "NEXT_PUBLIC_DEFAULT_REGION",
    description: "Default region code (e.g., us)"
  }
]
```

## 🟡 Important Issues

### 6. Unsafe Array Access Without Null Checks
**Severity:** MEDIUM  
**Multiple Files**

**Examples:**
```typescript
// middleware.ts line 66 - GOOD ✅
const urlCountryCode = request.nextUrl.pathname.split("/")[1]?.toLowerCase()

// products/[handle]/page.tsx line 66 - BAD ❌
if (!variant || !variant.images.length) {
// No check if variant.images exists before .length

// cart/templates/index.tsx line 18 - GOOD ✅
{cart?.items?.length ? (
```

**Issue:** Inconsistent null/undefined checking across codebase.

**Impact:** Runtime errors when optional properties are undefined.

**Recommendation:** Audit all array/object accesses and use optional chaining consistently.

### 7. Payment Wrapper Missing Stripe Key Validation
**Severity:** MEDIUM  
**File:** [my-medusa-storefront/src/modules/checkout/components/payment-wrapper/index.tsx](my-medusa-storefront/src/modules/checkout/components/payment-wrapper/index.tsx#L14-L23)

**Issue:**
```tsx
const stripeKey =
  process.env.NEXT_PUBLIC_STRIPE_KEY ||
  process.env.NEXT_PUBLIC_MEDUSA_PAYMENTS_PUBLISHABLE_KEY

const stripePromise = stripeKey
  ? loadStripe(stripeKey, ...)
  : null
```

**Impact:** If Stripe key is invalid, `loadStripe()` may fail silently. No user feedback.

**Solution:** Add error handling for Stripe initialization and show user-friendly message if payment provider fails to load.

### 8. Region Map Cache Never Clears
**Severity:** MEDIUM  
**File:** [my-medusa-storefront/src/lib/data/regions.ts](my-medusa-storefront/src/lib/data/regions.ts#L38-L40)

**Issue:**
```typescript
const regionMap = new Map<string, HttpTypes.StoreRegion>()

export const getRegion = async (countryCode: string) => {
  if (regionMap.has(countryCode)) {
    return regionMap.get(countryCode)  // Returns cached forever
  }
```

**Impact:** 
- Region changes in admin won't reflect without restart
- Potential memory leak
- Stale data served to users

**Solution:**
```typescript
export const getRegion = async (countryCode: string) => {
  const regions = await listRegions() // Already cached with Next.js
  const regionMap = new Map<string, HttpTypes.StoreRegion>()
  
  regions?.forEach((region) => {
    region.countries?.forEach((c) => {
      regionMap.set(c?.iso_2 ?? "", region)
    })
  })
  
  return regionMap.get(countryCode)
}
```

### 9. Middleware Cache Race Conditions
**Severity:** MEDIUM  
**File:** [my-medusa-storefront/src/middleware.ts](my-medusa-storefront/src/middleware.ts#L12-L26)

**Issue:**
```typescript
const regionMapCache = {
  regionMap: new Map<string, HttpTypes.StoreRegion>(),
  regionMapUpdated: Date.now(),
}

if (
  !regionMap.keys().next().value ||
  regionMapUpdated < Date.now() - 3600 * 1000
) {
  // Refetch regions - multiple requests might trigger this
```

**Impact:** 
- Cache shared across ALL requests (middleware runs on edge)
- Race conditions when multiple requests trigger cache update
- Multiple concurrent requests may all refetch

**Recommendation:** Use Next.js native cache with `revalidate` instead of manual cache in middleware.

### 10. Console Logs Left in Production Code
**Severity:** LOW  
**Multiple Files**

**Found in:**
- [middleware.ts](my-medusa-storefront/src/middleware.ts#L96) - `console.error` in production
- [products/[handle]/page.tsx](my-medusa-storefront/src/app/[countryCode]/(main)/products/[handle]/page.tsx#L48) - `console.error` 
- [categories/[...category]/page.tsx](my-medusa-storefront/src/app/[countryCode]/(main)/categories/[...category]/page.tsx#L47) - `console.warn`
- [cart/page.tsx](my-medusa-storefront/src/app/[countryCode]/(main)/cart/page.tsx#L14) - `console.error`
- [util/medusa-error.ts](my-medusa-storefront/src/lib/util/medusa-error.ts#L6-L9) - Multiple console logs

**Impact:** Exposes debug information in production, clutters browser console.

**Solution:**
```typescript
if (process.env.NODE_ENV === "development") {
  console.error(error)
}
```

## ⚪ Minor/Best Practice Issues

### 11. Type Assertions Without Validation
**Severity:** LOW

**Examples:**
```typescript
// payment-button/index.tsx
session?.data.client_secret as string

// cart/templates/summary.tsx
<Summary cart={cart as any} />
```

**Recommendation:** Avoid `as string` and `as any`. Use proper type guards or optional chaining.

### 12. Commented Out Code
**Severity:** LOW  

**Found in:**
- [cart.ts](my-medusa-storefront/src/lib/data/cart.ts#L289) - `// throw error` commented
- [product-preview/index.tsx](my-medusa-storefront/src/modules/products/components/product-preview/index.tsx#L21) - Commented fetch call

**Recommendation:** Remove commented code before production.

### 13. Missing Loading States
**Severity:** LOW

**Issue:** Many components fetch data without showing loading indicators (product pages, cart, checkout).

**Impact:** Poor UX - users see blank screen during data fetching.

**Recommendation:** Use loading skeletons consistently (some exist in `/skeletons/` but not consistently applied).

### 14. No Rate Limiting on Client Actions
**Severity:** LOW

**Issue:** Client-side actions like `addToCart`, `updateLineItem` don't have rate limiting or debouncing.

**Impact:** Users can spam buttons, creating multiple concurrent requests.

**Recommendation:** Add debouncing or disable buttons during submission.

## 📋 Storefront-Specific Pre-Deployment Checklist

- [ ] Add error.tsx files at critical route levels (global, country, main, checkout)
- [ ] Fix null checks in cart page
- [ ] Improve middleware error handling to avoid blocking all requests
- [ ] Add validation for payment session data
- [ ] Expand environment variable validation to include all critical vars
- [ ] Remove or conditionally enable console.logs
- [ ] Test all payment flows thoroughly (success, failure, network errors)
- [ ] Verify regions load correctly and caching works
- [ ] Test cart operations (add/update/remove items)
- [ ] Test checkout flow end-to-end
- [ ] Test with empty cart/no items scenarios
- [ ] Test with invalid URLs/country codes
- [ ] Test customer authentication flows (signup, login, logout)
- [ ] Ensure all images have proper alt text
- [ ] Test on mobile devices and different browsers
- [ ] Test with slow/unreliable network connections
- [ ] Verify proper error messages show to users

## 🚨 High Priority Fixes Before VPS Deployment

1. **Add error boundaries** (1-2 hours) - Critical for production stability
2. **Fix middleware BACKEND_URL** handling (30 min) - App breaks without it
3. **Validate payment flows** (2-3 hours) - Money is involved, must be bulletproof
4. **Add comprehensive null checks** (1 hour) - Prevents runtime crashes
5. **Remove debug code** (30 min) - Clean up console logs

**Estimated Time to Fix Critical Storefront Issues:** 4-6 hours

## 💡 Testing Recommendations

### Test Scenarios:
1. **Network failures** - Kill backend during browsing
2. **Empty states** - No products, no cart items, no regions
3. **Invalid data** - Bad country codes, missing env vars
4. **Payment failures** - Declined cards, network errors during payment
5. **Concurrent operations** - Multiple users, rapid clicks
6. **Cache invalidation** - Change regions/products in admin

### Load Testing:
- Test with 50+ concurrent users
- Verify middleware doesn't become bottleneck
- Check memory usage over time
- Monitor region cache behavior

---

## 📊 Complete Issues Summary

**VPS Deployment Issues:** 13 total (5 critical, 5 important, 3 minor)  
**Storefront Code Issues:** 14 total (5 critical, 5 important, 4 minor)

**Total Issues Found:** 27  
- **Critical (Must Fix):** 10
- **Important (Strongly Recommended):** 10  
- **Minor (Nice to Have):** 7

**Critical Actions Required Before VPS Deployment:**
1. Replace localhost URLs with actual domain
2. Generate strong security secrets
3. Change database credentials
4. Add error boundaries throughout storefront
5. Fix middleware error handling
6. Improve null/undefined handling
7. Test payment flows extensively
8. Add comprehensive environment validation
9. Remove production console logs
10. Set up SSL/TLS and reverse proxy

**Estimated Total Time for All Critical Fixes:** 8-12 hours