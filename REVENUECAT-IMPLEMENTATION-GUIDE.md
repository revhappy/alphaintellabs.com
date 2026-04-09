# Cross-Platform Subscription Billing — Implementation Guide

**Alpha Intel Labs** — Reusable across all apps (Aura Check, Tara, Ahara, etc.)

**Stack:** React + Capacitor 6 + Firebase (Auth, Firestore, Cloud Functions) + RevenueCat + Stripe

**Last Updated:** 2026-04-08

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Prerequisites & Accounts](#2-prerequisites--accounts)
3. [RevenueCat Dashboard Setup](#3-revenuecat-dashboard-setup)
4. [Store Product Configuration](#4-store-product-configuration)
5. [Stripe Setup (Web)](#5-stripe-setup-web)
6. [Cloud Function: Webhook Handler](#6-cloud-function-webhook-handler)
7. [Client SDK Integration (Capacitor)](#7-client-sdk-integration-capacitor)
8. [Paywall UI](#8-paywall-ui)
9. [Subscription Management UI](#9-subscription-management-ui)
10. [Testing](#10-testing)
11. [Troubleshooting](#11-troubleshooting)
12. [Per-App Checklist](#12-per-app-checklist)

---

## 1. Architecture Overview

```
User Device                     RevenueCat                    Your Backend
─────────────                   ──────────                    ────────────
App (Capacitor)                 Dashboard + API               Firebase Cloud Functions
  │                               │                             │
  ├─ Purchases.configure()        │                             │
  ├─ Purchases.logIn(firebaseUid) │                             │
  ├─ Purchases.getOfferings() ───→│ Returns products/prices     │
  ├─ Purchases.purchasePackage()─→│ Handles receipt validation  │
  │                               │                             │
  │                               ├─ Webhook POST ────────────→ │ revenueCatWebhook()
  │                               │   (INITIAL_PURCHASE,        │   ├─ Verify auth header
  │                               │    RENEWAL, CANCELLATION,   │   ├─ Map product → tier
  │                               │    EXPIRATION, etc.)        │   └─ Update Firestore
  │                               │                             │       users/{uid}.subscriptionTier
  │                               │                             │
  ├─ Existing scan logic reads ←──┼─────────────────────────────┘
  │   subscriptionTier from       │
  │   Firestore (unchanged)       │
```

**Key principle:** RevenueCat handles all payment complexity (receipt validation, cross-platform sync, store APIs). Your backend just listens for webhook events and updates a single Firestore field (`subscriptionTier`). All existing scan-limit logic stays untouched.

---

## 2. Prerequisites & Accounts

### Accounts to Create

| Account | URL | Purpose | Cost |
|---------|-----|---------|------|
| **RevenueCat** | https://app.revenuecat.com/signup | Subscription hub | Free up to $2.5k/mo MTR, then 1% |
| **Stripe** | https://dashboard.stripe.com/register | Web payments | 2.9% + $0.30 per transaction |
| **Apple Developer** | https://developer.apple.com/programs | iOS IAP | $99/year |
| **Google Play Console** | https://play.google.com/console | Android billing | $25 one-time |

### Cost Optimization

- **Apple Small Business Program** — Enroll at https://developer.apple.com/app-store/small-business-program/ to reduce commission from 30% to **15%**
- **Google Play 15% tier** — First $1M in annual revenue qualifies for 15% rate automatically

### Dependencies to Install

```bash
npm install @revenuecat/purchases-capacitor
npx cap sync
```

---

## 3. RevenueCat Dashboard Setup

### 3.1. Create a Project

1. Log into https://app.revenuecat.com
2. Click **+ New Project**
3. Name it after your app (e.g., "Aura Check")

### 3.2. Add Platform Apps

In your RevenueCat project, add apps for each platform:

**Apple App Store:**
1. Click **+ New App** → Apple App Store
2. App name: your app name
3. Bundle ID: your app's bundle ID (e.g., `com.auracheck.app`)
4. Shared Secret: Get from App Store Connect → Your App → App Information → App-Specific Shared Secret → **Generate**
5. Paste the shared secret into RevenueCat
6. Copy the **Apple Server Notification URL** from RevenueCat and paste it into App Store Connect → Your App → App Information → App Store Server Notifications → **Production Server URL**

**Google Play:**
1. Click **+ New App** → Google Play Store
2. Package name: your app's package name (e.g., `com.auracheck.app`)
3. Service Account JSON: Create a service account in Google Cloud Console with the "Android Publisher" role. Download the JSON key file and upload it to RevenueCat. (See https://www.revenuecat.com/docs/getting-started/entitlements/google-play-store)
4. Copy the **Google Real-Time Developer Notification Topic** from RevenueCat and paste it into Google Play Console → Your App → Monetization setup → Real-time developer notifications

**Stripe (Web):**
1. Click **+ New App** → Stripe
2. Click **Connect Stripe Account** — this redirects to Stripe to authorize RevenueCat

### 3.3. Create Entitlements

Entitlements represent what features the user unlocks. Go to **Project Settings → Entitlements**:

| Entitlement ID | Description |
|----------------|-------------|
| `basic` | Basic tier access (e.g., 60 scans/month) |
| `pro` | Pro tier access (e.g., 200 scans/month) |
| `unlimited` | Unlimited access |

### 3.4. Create Products

Go to **Products** and add your store products. These map 1:1 to what you'll create in the stores (Step 4).

### 3.5. Create Offerings

Go to **Offerings** and create a **Default** offering:

| Package | Product (Apple) | Product (Google) | Product (Stripe) | Entitlement |
|---------|----------------|------------------|-------------------|-------------|
| `$rc_monthly` (Basic) | `com.yourapp.basic.monthly` | `basic_monthly` | Stripe price ID | `basic` |
| `$rc_monthly` (Pro) | `com.yourapp.pro.monthly` | `pro_monthly` | Stripe price ID | `pro` |
| `$rc_monthly` (Unlimited) | `com.yourapp.unlimited.monthly` | `unlimited_monthly` | Stripe price ID | `unlimited` |

> **Note:** You'll likely create 3 separate packages within the offering, one per tier.

### 3.6. Get API Keys

Go to **Project Settings → API Keys**. You'll need:
- **Apple API Key** — Used in iOS builds
- **Google API Key** — Used in Android builds
- **Public API Key** — Can be used for web (or use platform-specific keys)

Save these — you'll use them in Step 7.

### 3.7. Configure Webhook

Go to **Project Settings → Integrations → Webhooks**:

1. Webhook URL: `https://us-central1-YOUR_PROJECT_ID.cloudfunctions.net/revenueCatWebhook`
   (Replace `YOUR_PROJECT_ID` with your Firebase project ID)
2. Authorization header: Generate a random secret string (e.g., `whsec_xxxxxxxxxxxxxxxx`). Save this — you'll set it as a Firebase secret in Step 6.

---

## 4. Store Product Configuration

### 4.1. Apple App Store Connect

1. Go to https://appstoreconnect.apple.com → Your App → **Subscriptions**
2. Create a **Subscription Group** (e.g., "Premium Plans")
3. Add subscriptions:

| Reference Name | Product ID | Duration | Price |
|---------------|-----------|----------|-------|
| Basic Monthly | `com.yourapp.basic.monthly` | 1 Month | $6.99 |
| Pro Monthly | `com.yourapp.pro.monthly` | 1 Month | $11.99 |
| Unlimited Monthly | `com.yourapp.unlimited.monthly` | 1 Month | $22.99 |

4. Add a localization for each (display name, description)
5. Submit for review (happens with app review)

### 4.2. Google Play Console

1. Go to https://play.google.com/console → Your App → **Monetization → Products → Subscriptions**
2. Create subscriptions:

| Product ID | Name | Billing Period | Price |
|-----------|------|---------------|-------|
| `basic_monthly` | Basic | Monthly | $6.99 |
| `pro_monthly` | Pro | Monthly | $11.99 |
| `unlimited_monthly` | Unlimited | Monthly | $22.99 |

3. For each subscription, add a **Base Plan** (monthly, auto-renewing)
4. Activate each subscription

### 4.3. Stripe Products (Web)

1. Go to https://dashboard.stripe.com → **Products**
2. Create 3 products with monthly recurring prices:

| Product | Price | Lookup Key |
|---------|-------|------------|
| Basic | $5.99/month | `basic_monthly` |
| Pro | $9.99/month | `pro_monthly` |
| Unlimited | $19.99/month | `unlimited_monthly` |

> **Mobile prices are higher** to offset the 15-30% platform cut. Web prices are lower since Stripe only charges ~3%.

> **Anti-steering warning:** Do NOT advertise cheaper web prices from within the iOS app. This violates Apple's guidelines. Drive web purchases through external marketing only.

---

## 5. Stripe Setup (Web)

RevenueCat offers two web approaches:

### Option A: RevenueCat Web Billing (Recommended)

RevenueCat handles checkout UI and Stripe processing:
1. In RevenueCat dashboard, enable **Web Billing** for your Stripe app
2. RevenueCat generates **Web Purchase Links** for each product
3. Redirect users to these links from your web paywall
4. RevenueCat handles the rest — subscription state syncs automatically

### Option B: Stripe Billing (Self-Managed)

You manage Stripe Checkout yourself:
1. Create a `createCheckoutSession` Cloud Function that creates a Stripe Checkout session
2. Redirect users to Stripe Checkout
3. RevenueCat's Stripe integration auto-imports purchases

**Recommendation:** Start with Option A (simpler). Switch to Option B only if you need custom checkout UI.

---

## 6. Cloud Function: Webhook Handler

This is the bridge between RevenueCat and your Firestore `subscriptionTier` field.

### 6.1. Set the Webhook Secret

```bash
firebase functions:secrets:set REVENUECAT_WEBHOOK_SECRET --project YOUR_PROJECT_ID
# Paste the authorization header value you set in RevenueCat
```

### 6.2. Create the Handler

**File: `functions/src/handlers/revenueCatWebhook.ts`**

```typescript
import { onRequest } from "firebase-functions/v2/https";
import { defineSecret } from "firebase-functions/params";
import * as admin from "firebase-admin";

const revenueCatWebhookSecret = defineSecret("REVENUECAT_WEBHOOK_SECRET");

// Map RevenueCat product IDs → your Firestore subscription tier values
// CUSTOMIZE THIS PER APP
const PRODUCT_TO_TIER: Record<string, string> = {
  // Apple products
  "com.yourapp.basic.monthly": "basic",
  "com.yourapp.pro.monthly": "pro",
  "com.yourapp.unlimited.monthly": "unlimited",
  // Google products
  "basic_monthly": "basic",
  "pro_monthly": "pro",
  "unlimited_monthly": "unlimited",
  // Stripe products (use the RevenueCat product identifier)
  "price_basic_monthly": "basic",
  "price_pro_monthly": "pro",
  "price_unlimited_monthly": "unlimited",
};

// Events that mean the user has an active subscription
const ACTIVE_EVENTS = new Set([
  "INITIAL_PURCHASE",
  "RENEWAL",
  "UNCANCELLATION",
  "SUBSCRIPTION_EXTENDED",
]);

// Events that mean the subscription has ended
const INACTIVE_EVENTS = new Set([
  "EXPIRATION",
]);

// Events to log but not act on (subscription is still active until expiration)
const INFO_EVENTS = new Set([
  "CANCELLATION",        // User cancelled — still active until period ends
  "BILLING_ISSUE",       // Payment failed — grace period active
  "PRODUCT_CHANGE",      // Upgrade/downgrade — handled by next RENEWAL
  "SUBSCRIPTION_PAUSED", // Paused — still has access during pause period
  "TRANSFER",            // User transferred to another account
]);

export const revenueCatWebhook = onRequest(
  {
    timeoutSeconds: 30,
    memory: "256MiB",
    maxInstances: 10,
    secrets: [revenueCatWebhookSecret],
  },
  async (req, res) => {
    // Only accept POST
    if (req.method !== "POST") {
      res.status(405).send("Method Not Allowed");
      return;
    }

    // Verify authorization header
    const authHeader = req.headers["authorization"];
    const expectedSecret = revenueCatWebhookSecret.value();

    if (!authHeader || authHeader !== `Bearer ${expectedSecret}`) {
      console.error("[RevenueCat Webhook] Invalid authorization header");
      res.status(401).send("Unauthorized");
      return;
    }

    try {
      const event = req.body?.event;

      if (!event) {
        console.error("[RevenueCat Webhook] No event in body");
        res.status(400).send("Bad Request");
        return;
      }

      const eventType: string = event.type;
      const appUserId: string = event.app_user_id;
      const productId: string = event.product_id;
      const environment: string = event.environment;

      console.log(`[RevenueCat Webhook] ${eventType} | user: ${appUserId} | product: ${productId} | env: ${environment}`);

      // Skip sandbox events in production (optional — remove if you want to test)
      // if (environment === "SANDBOX") {
      //   console.log("[RevenueCat Webhook] Skipping sandbox event");
      //   res.status(200).send("OK (sandbox ignored)");
      //   return;
      // }

      // Skip if no user ID (anonymous purchases shouldn't happen, but guard against it)
      if (!appUserId || appUserId.startsWith("$RCAnonymousID:")) {
        console.warn("[RevenueCat Webhook] Anonymous user, skipping");
        res.status(200).send("OK");
        return;
      }

      const db = admin.firestore();
      const userRef = db.collection("users").doc(appUserId);

      if (ACTIVE_EVENTS.has(eventType)) {
        // User has an active subscription — update their tier
        const tier = PRODUCT_TO_TIER[productId] || "basic";

        await userRef.set({
          subscriptionTier: tier,
          subscriptionProductId: productId,
          subscriptionStore: event.store || null,
          subscriptionUpdatedAt: admin.firestore.FieldValue.serverTimestamp(),
          subscriptionExpiresAt: event.expiration_at_ms
            ? new Date(event.expiration_at_ms)
            : null,
        }, { merge: true });

        console.log(`[RevenueCat Webhook] Updated user ${appUserId} to tier: ${tier}`);

      } else if (INACTIVE_EVENTS.has(eventType)) {
        // Subscription expired — downgrade to free
        // BUT: don't downgrade promo users
        const userDoc = await userRef.get();

        if (userDoc.exists) {
          const userData = userDoc.data()!;

          // If user has a promo code and no paid subscription, don't downgrade
          if (userData.promoCode && !userData.subscriptionProductId) {
            console.log(`[RevenueCat Webhook] User ${appUserId} has promo code, not downgrading`);
          } else {
            await userRef.set({
              subscriptionTier: "free",
              subscriptionProductId: null,
              subscriptionStore: null,
              subscriptionUpdatedAt: admin.firestore.FieldValue.serverTimestamp(),
              subscriptionExpiresAt: null,
            }, { merge: true });

            console.log(`[RevenueCat Webhook] Downgraded user ${appUserId} to free`);
          }
        }

      } else if (INFO_EVENTS.has(eventType)) {
        // Log but don't change tier (subscription still active)
        console.log(`[RevenueCat Webhook] Info event ${eventType} for user ${appUserId}, no tier change`);

      } else if (eventType === "TEST") {
        console.log("[RevenueCat Webhook] Test event received successfully");

      } else {
        console.warn(`[RevenueCat Webhook] Unknown event type: ${eventType}`);
      }

      // Always return 200 — RevenueCat will retry on non-200
      res.status(200).send("OK");

    } catch (error) {
      console.error("[RevenueCat Webhook] Error processing event:", error);
      // Return 500 so RevenueCat retries
      res.status(500).send("Internal Server Error");
    }
  }
);
```

### 6.3. Export in `functions/src/index.ts`

```typescript
export { revenueCatWebhook } from "./handlers/revenueCatWebhook";
```

### 6.4. Deploy

```bash
cd functions && npx tsc && cd ..
firebase deploy --only functions:revenueCatWebhook --project YOUR_PROJECT_ID
```

**IMPORTANT:** This is an `onRequest` function (HTTP endpoint), NOT an `onCall` function. It does NOT need the Cloud Run IAM `allUsers` invoker grant because RevenueCat authenticates via the Authorization header, and `onRequest` functions are publicly accessible by default on Cloud Functions v2.

Wait — actually, Cloud Functions v2 (Cloud Run) **does** block unauthenticated requests by default. You MUST grant `allUsers` invoker access:

```bash
gcloud run services add-iam-policy-binding revenuecatwebhook \
  --region=us-central1 \
  --project=YOUR_PROJECT_ID \
  --member="allUsers" \
  --role="roles/run.invoker" \
  --quiet
```

### 6.5. Copy Webhook URL to RevenueCat

After deployment, the URL will be:
```
https://us-central1-YOUR_PROJECT_ID.cloudfunctions.net/revenueCatWebhook
```

Paste this into RevenueCat → Project Settings → Integrations → Webhooks → Webhook URL.

Click **Send Test Event** to verify — you should see "Test event received successfully" in your function logs.

---

## 7. Client SDK Integration (Capacitor)

### 7.1. Create a RevenueCat Service

**File: `src/services/revenueCatService.ts`**

```typescript
import { Capacitor } from "@capacitor/core";
import { Purchases, LOG_LEVEL, PURCHASES_ERROR_CODE, PurchasesOffering, Package } from "@revenuecat/purchases-capacitor";

// CUSTOMIZE THESE PER APP
const RC_API_KEY_APPLE = "appl_XXXXXXXXXXXXXXXX";    // From RevenueCat dashboard
const RC_API_KEY_GOOGLE = "goog_XXXXXXXXXXXXXXXX";   // From RevenueCat dashboard
const RC_API_KEY_WEB = "rcweb_XXXXXXXXXXXXXXXX";     // Optional — for web billing

/**
 * Initialize RevenueCat SDK.
 * Call this ONCE on app start, after Firebase Auth is ready.
 */
export async function initRevenueCat(): Promise<void> {
  const platform = Capacitor.getPlatform();

  let apiKey: string;
  if (platform === "ios") {
    apiKey = RC_API_KEY_APPLE;
  } else if (platform === "android") {
    apiKey = RC_API_KEY_GOOGLE;
  } else {
    // Web — RevenueCat web billing or skip
    if (RC_API_KEY_WEB) {
      apiKey = RC_API_KEY_WEB;
    } else {
      console.log("[RevenueCat] No web API key, skipping init");
      return;
    }
  }

  try {
    await Purchases.setLogLevel({ level: LOG_LEVEL.DEBUG }); // Change to WARN in production
    await Purchases.configure({ apiKey });
    console.log("[RevenueCat] Configured successfully");
  } catch (error) {
    console.error("[RevenueCat] Failed to configure:", error);
  }
}

/**
 * Identify the user with their Firebase UID.
 * Call this after sign-in / sign-up.
 * This links purchases across platforms to this user.
 */
export async function identifyUser(firebaseUid: string): Promise<void> {
  try {
    await Purchases.logIn({ appUserID: firebaseUid });
    console.log("[RevenueCat] Logged in as:", firebaseUid);
  } catch (error) {
    console.error("[RevenueCat] Failed to logIn:", error);
  }
}

/**
 * Log out the RevenueCat user (on sign-out).
 */
export async function logOutRevenueCat(): Promise<void> {
  try {
    await Purchases.logOut();
    console.log("[RevenueCat] Logged out");
  } catch (error) {
    console.error("[RevenueCat] Failed to logOut:", error);
  }
}

/**
 * Fetch available subscription offerings.
 * Returns the current offering with packages (tiers).
 */
export async function getOfferings(): Promise<PurchasesOffering | null> {
  try {
    const offerings = await Purchases.getOfferings();

    if (!offerings.current) {
      console.warn("[RevenueCat] No current offering");
      return null;
    }

    return offerings.current;
  } catch (error) {
    console.error("[RevenueCat] Failed to get offerings:", error);
    return null;
  }
}

/**
 * Purchase a package (subscription tier).
 * Returns true on success, false on failure/cancellation.
 */
export async function purchasePackage(pkg: Package): Promise<{
  success: boolean;
  error?: string;
  cancelled?: boolean;
}> {
  try {
    const result = await Purchases.purchasePackage({ aPackage: pkg });

    // Check if entitlement is now active
    const entitlements = result.customerInfo.entitlements.active;
    const hasEntitlement = Object.keys(entitlements).length > 0;

    if (hasEntitlement) {
      console.log("[RevenueCat] Purchase successful, active entitlements:", Object.keys(entitlements));
      return { success: true };
    } else {
      console.warn("[RevenueCat] Purchase completed but no active entitlements");
      return { success: false, error: "Purchase completed but subscription not activated. Please try again." };
    }
  } catch (error: any) {
    // RevenueCat throws specific error codes
    if (error.code === PURCHASES_ERROR_CODE.PURCHASE_CANCELLED_ERROR) {
      return { success: false, cancelled: true };
    }

    console.error("[RevenueCat] Purchase failed:", error);
    return {
      success: false,
      error: error.message || "Purchase failed. Please try again.",
    };
  }
}

/**
 * Check current entitlement status directly from RevenueCat.
 * Useful for quick checks, but Firestore is the primary source of truth.
 */
export async function getActiveEntitlements(): Promise<string[]> {
  try {
    const customerInfo = await Purchases.getCustomerInfo();
    return Object.keys(customerInfo.customerInfo.entitlements.active);
  } catch (error) {
    console.error("[RevenueCat] Failed to get customer info:", error);
    return [];
  }
}

/**
 * Restore purchases (e.g., after reinstall or switching devices).
 * RevenueCat handles finding the user's existing purchases.
 */
export async function restorePurchases(): Promise<{
  success: boolean;
  entitlements: string[];
}> {
  try {
    const customerInfo = await Purchases.restorePurchases();
    const entitlements = Object.keys(customerInfo.customerInfo.entitlements.active);
    console.log("[RevenueCat] Restored purchases, entitlements:", entitlements);
    return { success: true, entitlements };
  } catch (error) {
    console.error("[RevenueCat] Failed to restore purchases:", error);
    return { success: false, entitlements: [] };
  }
}
```

### 7.2. Initialize on App Start

In your main app component (e.g., `App.tsx`), after Firebase initializes:

```typescript
import { initRevenueCat, identifyUser, logOutRevenueCat } from './services/revenueCatService';

// In your app init / useEffect:
useEffect(() => {
  initRevenueCat();
}, []);

// After successful sign-in:
const handleAuthSuccess = async () => {
  const user = auth.currentUser;
  if (user && !user.isAnonymous) {
    await identifyUser(user.uid);
  }
};

// On sign-out:
const handleSignOut = async () => {
  await logOutRevenueCat();
  await signOut();
};
```

### 7.3. Wire Into Auth Flow

In your auth service, after successful sign-in/sign-up, call `identifyUser()`:

```typescript
// In signInWithEmail, signInWithGoogle, signUpWithEmail — after success:
await identifyUser(user.uid);
```

---

## 8. Paywall UI

### 8.1. Paywall Component

This component replaces the "Coming soon" subscription tiers. It:
- Fetches RevenueCat offerings on mount
- Shows tier cards with real prices from the store
- Handles purchases
- Shows platform-appropriate prices (higher on mobile, lower on web)

```typescript
// Pseudocode / structure for PaywallView component

import { getOfferings, purchasePackage } from '../services/revenueCatService';
import { Capacitor } from '@capacitor/core';

function PaywallView({ onSuccess, onClose }) {
  const [offerings, setOfferings] = useState(null);
  const [loading, setLoading] = useState(true);
  const [purchasing, setPurchasing] = useState(null); // which tier is being purchased

  useEffect(() => {
    getOfferings().then(offering => {
      setOfferings(offering);
      setLoading(false);
    });
  }, []);

  const handlePurchase = async (pkg) => {
    setPurchasing(pkg.identifier);
    const result = await purchasePackage(pkg);

    if (result.success) {
      onSuccess(); // Dismiss paywall, refresh scan status
    } else if (!result.cancelled) {
      // Show error (cancelled = user backed out, don't show error)
      setError(result.error);
    }
    setPurchasing(null);
  };

  // For web: redirect to RevenueCat Web Purchase Link or Stripe Checkout
  const handleWebPurchase = (tier) => {
    // Option A: RevenueCat Web Purchase Links
    window.open(`https://pay.revenuecat.com/YOUR_LINK_HERE`, '_blank');

    // Option B: Your own createCheckoutSession cloud function
    // const session = await createCheckoutSession(tier);
    // window.location.href = session.url;
  };

  const isWeb = Capacitor.getPlatform() === 'web';

  // Render tier cards with prices from offerings
  // offerings.availablePackages contains the packages with localized prices
}
```

### 8.2. Integrate Into AuthWall

Replace the "Coming soon" section in your AuthWall with the real paywall when `reason === 'subscription_required' || reason === 'free_limit_reached'`.

---

## 9. Subscription Management UI

Add to your Settings page:

```typescript
// Current plan display
const { userProfile } = useAuth();
const tier = userProfile?.subscriptionTier || 'free';

// Manage subscription button
const handleManageSubscription = () => {
  const platform = Capacitor.getPlatform();

  if (platform === 'ios') {
    // Deep link to iOS subscription settings
    window.open('https://apps.apple.com/account/subscriptions', '_blank');
  } else if (platform === 'android') {
    // Deep link to Google Play subscriptions
    window.open('https://play.google.com/store/account/subscriptions', '_blank');
  } else {
    // Web: link to your subscription management page or Stripe portal
    window.open('YOUR_STRIPE_CUSTOMER_PORTAL_URL', '_blank');
  }
};

// Restore purchases button (for users switching devices)
const handleRestore = async () => {
  const result = await restorePurchases();
  if (result.success && result.entitlements.length > 0) {
    showToast('Purchases restored!');
    refreshStatus();
  } else {
    showToast('No active subscriptions found.');
  }
};
```

---

## 10. Testing

### 10.1. RevenueCat Sandbox

- RevenueCat has a **Sandbox** mode enabled by default during development
- Apple sandbox subscriptions renew every few minutes (not monthly)
- Google test purchases are free — add your Google account as a **License Tester** in Google Play Console

### 10.2. Test Scenarios

| Scenario | Expected Result |
|----------|----------------|
| Purchase Basic on Android | Firestore `subscriptionTier` → `"basic"`, scan limit → 60 |
| Purchase Pro on iOS | Firestore `subscriptionTier` → `"pro"`, scan limit → 200 |
| Purchase Unlimited on web | Stripe charges, webhook fires, Firestore → `"unlimited"` |
| Cancel subscription | `CANCELLATION` event logged, tier stays active until period ends |
| Subscription expires | `EXPIRATION` event, Firestore → `"free"` |
| User has promo code + cancels | Promo tier preserved, not downgraded |
| Restore purchases on new device | RevenueCat finds existing subscription, Firestore already correct |
| Purchase while anonymous | Should not be possible — require auth before showing paywall |

### 10.3. Test the Webhook

```bash
# Send a test event from RevenueCat dashboard
# Or use curl:
curl -X POST https://us-central1-YOUR_PROJECT_ID.cloudfunctions.net/revenueCatWebhook \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_WEBHOOK_SECRET" \
  -d '{
    "event": {
      "type": "TEST",
      "app_user_id": "test_user",
      "product_id": "basic_monthly"
    }
  }'

# Check logs:
firebase functions:log --only revenueCatWebhook --project YOUR_PROJECT_ID
```

### 10.4. Stripe Test Mode

- Stripe is in test mode by default
- Use test card: `4242 4242 4242 4242`, any future expiry, any CVC
- Test webhooks: Stripe CLI or Stripe dashboard webhook tester

---

## 11. Troubleshooting

### Common Issues

| Issue | Cause | Fix |
|-------|-------|-----|
| Webhook returns 401 | Auth header mismatch | Verify `REVENUECAT_WEBHOOK_SECRET` matches what's in RevenueCat dashboard. Format: `Bearer YOUR_SECRET` |
| Webhook returns 403 | Cloud Run IAM blocking | Run the `gcloud run services add-iam-policy-binding` command from Step 6.4 |
| Offerings return null | Products not configured in RevenueCat | Verify products exist in stores AND are mapped to offerings in RevenueCat dashboard |
| Purchase fails on Android | App not signed with correct key | Google Play Billing requires the app to be signed with the same key as the Play Console upload |
| Purchase fails on iOS simulator | Simulators can't do IAP | Test on real device with sandbox Apple ID |
| `subscriptionTier` not updating | Webhook not reaching your function | Check function logs; verify webhook URL in RevenueCat; test with curl |
| Promo user downgraded | Webhook EXPIRATION fired | Check promo code guard in webhook handler — `userData.promoCode` check |
| User has subscription but app shows free | Firestore not synced | Check webhook logs; call `restorePurchases()` to trigger a sync |
| "Product not found" in RevenueCat | Product IDs don't match | Product IDs in stores must exactly match what's configured in RevenueCat |

### Useful Debug Commands

```bash
# Check webhook function logs
firebase functions:log --only revenueCatWebhook --project YOUR_PROJECT_ID --limit 50

# Check if webhook is deployed
gcloud run services list --project YOUR_PROJECT_ID --region us-central1

# Check IAM on webhook
gcloud run services get-iam-policy revenuecatwebhook \
  --region=us-central1 \
  --project=YOUR_PROJECT_ID

# Verify webhook secret is set
firebase functions:secrets:access REVENUECAT_WEBHOOK_SECRET --project YOUR_PROJECT_ID

# Check a user's Firestore subscription data
# (use Firebase Console or)
firebase firestore:get users/USER_UID --project YOUR_PROJECT_ID
```

---

## 12. Per-App Checklist

Copy this checklist for each new app:

```
## [APP NAME] — Payment Integration Checklist

### Accounts & Dashboard
- [ ] RevenueCat project created for this app
- [ ] Apple app added in RevenueCat (bundle ID, shared secret, server notifications URL)
- [ ] Google app added in RevenueCat (package name, service account JSON, RTDN topic)
- [ ] Stripe app connected in RevenueCat
- [ ] Entitlements created (basic, pro, unlimited — or your app's tiers)
- [ ] Products created and mapped to entitlements
- [ ] Offerings created with all packages
- [ ] API keys saved (Apple, Google, Web)
- [ ] Webhook URL configured with auth secret

### Store Products
- [ ] Apple App Store Connect: subscription group + products created
- [ ] Google Play Console: subscriptions created and activated
- [ ] Stripe: products + monthly prices created

### Cloud Function
- [ ] `REVENUECAT_WEBHOOK_SECRET` set via firebase secrets
- [ ] `revenueCatWebhook` handler created with correct product-to-tier mapping
- [ ] Exported in `functions/src/index.ts`
- [ ] Deployed and IAM granted
- [ ] Test event successful from RevenueCat dashboard

### Client
- [ ] `@revenuecat/purchases-capacitor` installed
- [ ] `revenueCatService.ts` created with correct API keys
- [ ] `initRevenueCat()` called on app start
- [ ] `identifyUser(uid)` called after sign-in
- [ ] `logOutRevenueCat()` called on sign-out
- [ ] Paywall UI shows real offerings with store prices
- [ ] Purchase flow works (native IAP on mobile, Stripe/web links on web)
- [ ] Subscription management in Settings (manage / restore / current plan)

### Testing
- [ ] Purchase on Android sandbox → Firestore tier updates
- [ ] Purchase on iOS sandbox → Firestore tier updates
- [ ] Purchase on web (Stripe test) → Firestore tier updates
- [ ] Cancellation → tier stays active until expiration
- [ ] Expiration → tier reverts to free
- [ ] Promo code users → not downgraded by webhook
- [ ] Restore purchases on new device → works
- [ ] Anonymous users → cannot reach paywall (auth required first)
```

---

## References

- [RevenueCat Capacitor SDK Docs](https://www.revenuecat.com/docs/getting-started/installation/capacitor)
- [RevenueCat Webhook Events & Fields](https://www.revenuecat.com/docs/integrations/webhooks/event-types-and-fields)
- [RevenueCat Firebase Integration](https://www.revenuecat.com/docs/integrations/third-party-integrations/firebase-integration)
- [RevenueCat Web Billing](https://www.revenuecat.com/docs/web/web-billing/overview)
- [Stripe Billing Integration](https://www.revenuecat.com/docs/web/integrations/stripe)
- [RevenueCat Webhook Sample Events](https://www.revenuecat.com/docs/integrations/webhooks/sample-events)
- [RevenueCat purchases-capacitor GitHub](https://github.com/RevenueCat/purchases-capacitor)
