# Bug Fixes Applied (v24)

## Critical Payment Gateway Fixes

### BUG-01: Paytm / JazzCash / Easypaisa / PayFast — Orders Never Confirmed
**File:** `src/components/CartModal.tsx`  
**Issue:** The payment callback `useEffect` handled Stripe, bKash, Nagad, PayPal, SSLCommerz, and Razorpay return URLs, but had ZERO handlers for Paytm, JazzCash, Easypaisa, and PayFast. When customers returned from these payment pages, the pending order sat in `localStorage` forever and was never placed.  
**Fix:** Added URL param reads (`?paytm=`, `?jazzcash=`, `?easypaisa=`, `?payfast=`) and complete success/failure callback handlers for all four gateways.

### BUG-02: Razorpay `ondismiss` Leaves Ghost Pending Order
**File:** `src/components/CartModal.tsx`  
**Issue:** When a customer dismissed the Razorpay modal without paying, only a toast was shown. The `qf_pending_order` key was never cleared from localStorage. On next checkout, the stale pending order could be re-submitted.  
**Fix:** `ondismiss` now clears `qf_pending_order` and `qf_pending_email` before showing the toast.

### BUG-03: PayPal Capture Missing Credentials
**File:** `src/components/CartModal.tsx`  
**Issue:** The `/api/paypal/capture-order` fetch sent only `orderId` and `sandboxMode`. If `PAYPAL_CLIENT_ID` / `PAYPAL_CLIENT_SECRET` env vars were absent on the server, PayPal capture failed with 400 errors.  
**Fix:** Now also sends `paypalClientId` and `paypalClientSecret` from admin CMS settings so the server can always acquire an access token.

### BUG-04: Payment Failure — No Customer Notification
**File:** `src/components/CartModal.tsx`  
**Issue:** When any payment failed or was cancelled, only a toast was shown in the browser. The customer received no SMS or email about the failed payment.  
**Fix:** Added a failure notification block that calls `/api/send-sms` and `/api/send-email` with the failed order details (phone + email) when `failStatuses.some(Boolean)` is true.

## Security Fixes

### BUG-05: Stripe Secret Key Sent From Browser
**File:** `src/components/CartModal.tsx`  
**Issue:** `stripeSecretKey: paymentSettings.stripeSecretKey` was included in the JSON body of the POST to `/api/stripe/create-checkout-session`. Anyone opening DevTools Network tab could steal the live Stripe secret key.  
**Fix:** Removed `stripeSecretKey` from the browser request. Server already falls back to `process.env.STRIPE_SECRET_KEY`.

### BUG-06: Admin Password Hash Stored in localStorage
**File:** `src/db.ts` (`saveAdminSettings`)  
**Issue:** `setLocal('qf_adminSettings', settings)` stored the full admin credentials including the password hash in plaintext localStorage, readable by any browser extension or XSS payload.  
**Fix:** Password is stripped (`{ ...settings, password: '' }`) before the localStorage cache write. The real value is persisted only in Firebase/Supabase and the in-memory store.

## Rate Limiting Fixes

### BUG-07: No Rate Limiting on Payment Creation Endpoints
**File:** `server.ts`  
**Issue:** `/api/stripe/create-checkout-session`, `/api/paypal/create-order`, `/api/razorpay/create-order`, `/api/bkash/create-payment`, and `/api/sslcommerz/create-payment` had no rate limiting. A bad actor could spam payment creation requests to drain API quotas or enumerate credentials.  
**Fix:** Added `checkRateLimit('pay:gateway:IP', 5, 60000)` at the top of each handler (max 5 payment initiations per IP per minute).

## Logic / Data Integrity Fixes

### BUG-08: Order Saved to DB Before Stock Validation
**File:** `src/context/AppContext.tsx` (`placeOrder`)  
**Issue:** `dbService.saveOrder(newOrder)` was called BEFORE the stock guard and atomic deduction loop. If any item was out of stock, the order was already persisted in the DB as a ghost order (never confirmed, never fulfilled).  
**Fix:** Reordered: stock validation → atomic deduction → `saveOrder`. Order is only written to DB after all guards pass.

### BUG-09: Coupon Expiry Not Re-Checked at Order Time
**File:** `src/context/AppContext.tsx` (`placeOrder`)  
**Issue:** `applyCouponCode()` checked expiry at apply time, but `placeOrder()` never re-checked. A coupon could expire between the time it was applied and the order was placed (multi-tab, slow checkout, midnight boundary).  
**Fix:** Added expiry date and usage limit re-check inside `placeOrder()` before stock validation.

### BUG-10: `taxPercentage` Default = 5% Applied to All New Installs
**File:** `src/db.ts` (`DEFAULT_PAYMENT_SETTINGS`)  
**Issue:** `taxPercentage: 0.05` auto-applied 5% tax to every order on fresh installs, even when the store owner never configured tax.  
**Fix:** Changed default to `taxPercentage: 0`.

## Summary

| # | Severity | File | Bug | Status |
|---|----------|------|-----|--------|
| 1 | 🔴 Critical | CartModal.tsx | Paytm/Jazz/Easypaisa/PayFast orders never confirmed | ✅ Fixed |
| 2 | 🔴 Critical | CartModal.tsx | Razorpay ghost pending order on dismiss | ✅ Fixed |
| 3 | 🔴 Critical | CartModal.tsx | PayPal capture fails without credentials | ✅ Fixed |
| 4 | 🟡 Major | CartModal.tsx | No SMS/email on payment failure | ✅ Fixed |
| 5 | 🔴 Security | CartModal.tsx | Stripe secret key exposed in browser network tab | ✅ Fixed |
| 6 | 🔴 Security | db.ts | Admin password hash in localStorage | ✅ Fixed |
| 7 | 🟡 Major | server.ts | No rate limiting on payment creation endpoints | ✅ Fixed |
| 8 | 🟡 Major | AppContext.tsx | Ghost orders saved before stock check | ✅ Fixed |
| 9 | 🟡 Major | AppContext.tsx | Coupon expiry not re-checked at order time | ✅ Fixed |
| 10 | 🟢 Minor | db.ts | Default tax 5% applied to all new stores | ✅ Fixed |
