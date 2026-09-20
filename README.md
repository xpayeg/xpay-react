# @xpayeg/react

React components and hooks for XPay payments. A thin wrapper around the [XPay JS SDK](https://www.npmjs.com/package/@xpayeg/sdk) that provides React-friendly APIs for embedding payment forms.

## Documentation

Full guides, API reference, and live examples: **<https://docs.xpay.app>**

This README is a quick-start. The docs site is the authoritative reference.

## Installation

```bash
npm install @xpayeg/sdk @xpayeg/react
```

Both ESM (`import`) and CommonJS (`require`) are supported, with matching TypeScript declarations. Your tooling selects the appropriate entrypoint automatically.

## Step 1: Create a Checkout Session [Server-side]

On your server, create a Checkout Session and return the `clientSecret` to your frontend. The checkout session defines what you're charging for — line items, currency, amounts, and what happens after payment.

```javascript
// Your server (Node.js example with Express)
app.post("/api/create-checkout", async (req, res) => {
  const response = await fetch("https://api.xpay.app/checkout/sessions", {
    method: "POST",
    headers: {
      Authorization: `Bearer ${process.env.XPAY_SECRET_KEY}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({
      uiMode: "custom", // 'custom' for Elements SDK, 'embedded' for drop-in, 'hosted' for redirect
      lineItems: [
        {
          priceData: {
            unitAmount: 50000, // Amount in smallest unit (500.00 EGP = 50000 piasters)
            currency: "EGP",
            productData: {
              name: "Premium Plan",
              description: "Monthly subscription",
            },
          },
          quantity: 1,
        },
      ],
      afterCompletion: {
        type: "redirect",
        redirect: {
          // {CHECKOUT_SESSION_ID} is automatically replaced with the session ID
          url: "https://yoursite.com/success?session_id={CHECKOUT_SESSION_ID}",
        },
      },
      // Optional
      customerDetails: { email: req.body.email },
      brandingSettings: { colorMode: "system" },
    }),
  });

  const session = await response.json();
  res.json({ clientSecret: session.clientSecret });
});
```

## Step 2: Set Up the Frontend [Client-side]

Load XPay at module level (outside any component) and wrap your checkout UI with `XPayProvider`.

The `clientSecret` option accepts either a string or a `Promise<string>`, so you can pass the fetch directly without managing loading state yourself.

```tsx
import { loadXPay } from "@xpayeg/sdk";
import { XPayProvider, PaymentElement, useCheckout } from "@xpayeg/react";

// Load once at module level — not inside a component
const xpayPromise = loadXPay("pk_test_xxx");

// Fetch clientSecret as a Promise — no useState/useEffect needed
const fetchClientSecret = fetch("/api/create-checkout", { method: "POST" })
  .then((r) => r.json())
  .then((d) => d.clientSecret as string);

function App() {
  return (
    <XPayProvider xpay={xpayPromise} options={{ clientSecret: fetchClientSecret }}>
      <CheckoutForm />
    </XPayProvider>
  );
}
```

You can also pass `clientSecret` as a plain string if you already have it:

```tsx
<XPayProvider xpay={xpayPromise} options={{ clientSecret: "cs_test_abc_secret_xyz" }}>
  <CheckoutForm />
</XPayProvider>
```

## Step 3: Build the Checkout Form

`useCheckout()` returns a disjoint union — handle each state explicitly before accessing session data or methods.

```tsx
function CheckoutForm() {
  const checkoutState = useCheckout();

  if (checkoutState.type === "loading") {
    return <div className="animate-pulse h-48 bg-gray-100 rounded-lg" />;
  }

  if (checkoutState.type === "error") {
    return <div className="text-red-500">Failed to load: {checkoutState.error.message}</div>;
  }

  const { currency, amountTotal, lineItems, confirm, applyPromotionCode } = checkoutState.checkout;

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();

    const result = await confirm({
      customerDetails: { email: "customer@example.com", name: "John Doe" },
      // Default: redirect: "if_required" — returns result to your code
      // Use redirect: "always" to redirect to afterCompletion.redirect.url
    });

    // This code only runs if redirect: "if_required" or if redirect fails
    if (result.type === "error") {
      console.error(result.error.message);
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <p className="text-lg font-semibold">
        Total: {currency} {(amountTotal / 100).toFixed(2)}
      </p>

      <PaymentElement />

      <button type="submit">
        Pay {currency} {(amountTotal / 100).toFixed(2)}
      </button>
    </form>
  );
}
```

All payment actions (3DS challenges, Valu/PayTabs modals) are handled automatically inside `confirm()`. The promise resolves only when the full payment flow completes.

### Handling Session Status

After narrowing to the `success` state, check the session status to handle expired or completed sessions:

```tsx
function CheckoutPage() {
  const checkoutState = useCheckout();

  if (checkoutState.type === "loading") return <Loading />;
  if (checkoutState.type === "error") return <Error message={checkoutState.error.message} />;

  const { checkout } = checkoutState;

  if (checkout.status.type === "expired") {
    return <div>This checkout session has expired. Please start a new order.</div>;
  }

  if (checkout.status.type === "complete") {
    return <div>Payment complete. Thank you!</div>;
  }

  // status.type === "open" — render the payment form
  return <CheckoutForm />;
}
```

### Displaying Session Data

The `checkout` object on the success state contains all session data. Read `amountTotal`, `currency`, `paymentMethods`, and `merchantName` directly — no separate `sessionData` accessor needed.

```tsx
function OrderSummary() {
  const checkoutState = useCheckout();

  if (checkoutState.type === "loading") return <div>Loading...</div>;
  if (checkoutState.type === "error") return <div>Error: {checkoutState.error.message}</div>;

  const { checkout } = checkoutState;

  return (
    <div>
      {/* Merchant name */}
      <h2>{checkout.merchantName}</h2>

      {/* Amount and currency from session */}
      <p className="text-2xl font-bold">
        {checkout.currency} {(checkout.amountTotal / 100).toFixed(2)}
      </p>

      {/* Available payment methods */}
      <p className="text-sm text-gray-500">
        Pay with: {checkout.paymentMethods.map((pm) => pm.displayName).join(", ")}
      </p>

      {/* Live/test mode indicator */}
      {!checkout.livemode && (
        <span className="text-xs bg-yellow-100 text-yellow-800 px-2 py-1 rounded">Test Mode</span>
      )}
    </div>
  );
}
```

**Session fields on `checkout`:**

| Field                     | Type                  | Description                                                                                    |
| ------------------------- | --------------------- | ---------------------------------------------------------------------------------------------- |
| `checkout.id`             | `string`              | Session ID                                                                                     |
| `checkout.amountSubtotal` | `number`              | Subtotal before discounts/fees (smallest unit)                                                 |
| `checkout.amountTotal`    | `number`              | Total amount in smallest currency unit (piasters)                                              |
| `checkout.currency`       | `string`              | ISO 4217 currency code                                                                         |
| `checkout.merchantName`   | `string`              | Merchant display name                                                                          |
| `checkout.livemode`       | `boolean`             | Whether this is a live mode session                                                            |
| `checkout.expiresAt`      | `string`              | Session expiration timestamp                                                                   |
| `checkout.status`         | `SessionStatus`       | Disjoint union: `{type: "open"}` \| `{type: "expired"}` \| `{type: "complete", paymentStatus}` |
| `checkout.canConfirm`     | `boolean`             | Whether the session is ready for confirmation                                                  |
| `checkout.paymentMethods` | `PaymentMethodInfo[]` | Available payment methods                                                                      |
| `checkout.lineItems`      | `LineItem[]`          | Line items with product name, quantity, amount                                                 |
| `checkout.totalDetails`   | `TotalDetails`        | Amounts breakdown (discount, shipping, tax, fees)                                              |
| `checkout.fees`           | `Fees`                | Fee breakdown (when feesPassThrough enabled)                                                   |
| `checkout.discounts`      | `Discount[]`          | Applied discounts                                                                              |

### Updating the Session (Promo Codes, Quantities)

Action methods return `Promise<ActionResult>` — a tagged union of `{type: "success", session}` or `{type: "error", error}`. The hook's state updates reactively after each action, so `checkout.amountTotal`, `checkout.lineItems`, and `checkout.totalDetails` always reflect the latest values.

```tsx
function CheckoutWithPromo() {
  const checkoutState = useCheckout();
  const [promoCode, setPromoCode] = useState("");
  const [promoError, setPromoError] = useState("");

  if (checkoutState.type !== "success") return <div>Loading...</div>;

  const { checkout } = checkoutState;

  const handleApplyPromo = async () => {
    setPromoError("");
    const result = await checkout.applyPromotionCode(promoCode);
    if (result.type === "error") {
      setPromoError(result.error.message);
    }
  };

  return (
    <div>
      {/* Promo code input */}
      <input
        value={promoCode}
        onChange={(e) => setPromoCode(e.target.value)}
        placeholder="Promo code"
      />
      <button onClick={handleApplyPromo}>Apply</button>
      <button onClick={() => checkout.removePromotionCode()}>Remove</button>
      {promoError && <p className="text-red-500 text-sm">{promoError}</p>}

      {/* Line items with quantity controls */}
      {checkout.lineItems?.map((item) => (
        <div key={item.id}>
          <span>{item.description}</span>
          <button
            onClick={() =>
              checkout.updateLineItemQuantity({ lineItem: item.id, quantity: item.quantity + 1 })
            }
          >
            +
          </button>
          <button
            onClick={() =>
              checkout.updateLineItemQuantity({
                lineItem: item.id,
                quantity: Math.max(1, item.quantity - 1),
              })
            }
          >
            -
          </button>
        </div>
      ))}

      {/* Total updates reactively */}
      <p>
        Total: {checkout.currency} {(checkout.amountTotal / 100).toFixed(2)}
      </p>

      {/* Fee breakdown (when feesPassThrough enabled) */}
      {checkout.totalDetails?.amountPlatformFee && (
        <p>Processing Fee: {(checkout.totalDetails.amountPlatformFee / 100).toFixed(2)}</p>
      )}

      <PaymentElement />
      <button onClick={() => checkout.confirm()}>Pay</button>
    </div>
  );
}
```

**Reactive state:** `useCheckout()` updates automatically whenever the session changes — after applying a promo code, changing quantities, or any server-side update. There is no need for a separate `onChange` callback in React. The component re-renders with the latest `checkout` data.

### Error Event

Use `checkout.on("error", handler)` to listen for unsolicited errors -- errors that occur outside of a merchant-initiated action. Examples: session expired during fee recalculation when switching payment methods, BIN detection failure.

```tsx
function CheckoutForm({ checkout }: { checkout: Checkout }) {
  const [error, setError] = useState("");

  useEffect(() => {
    checkout.on("error", (err) => {
      console.error("[XPay SDK] Unsolicited error:", err.code, err.message);
      setError(err.message);
    });
  }, [checkout]);

  return (
    <div>
      {error && <p className="text-red-500">{error}</p>}
      <PaymentElement />
    </div>
  );
}
```

The error object shape:

```typescript
{
  type: "invalid_request_error";  // error category
  code: "checkout_session_expired"; // machine-readable code
  message: "This checkout session has expired"; // human-readable message
  docUrl?: "https://docs.xpay.app/api/errors#checkout_session_expired";
}
```

### Action Methods

All action methods live on the `checkout` object returned from the `success` state.

| Method                   | Signature                                         | Description                                                                       |
| ------------------------ | ------------------------------------------------- | --------------------------------------------------------------------------------- |
| `confirm`                | `(options?) => Promise<ActionResult>`             | Confirm payment. Handles 3DS and redirects.                                       |
| `applyPromotionCode`     | `(code: string) => Promise<ActionResult>`         | Apply a promotion code                                                            |
| `removePromotionCode`    | `() => Promise<ActionResult>`                     | Remove the applied promotion code                                                 |
| `updateLineItemQuantity` | `({lineItem, quantity}) => Promise<ActionResult>` | Update a line item's quantity                                                     |
| `submit`                 | `() => Promise<{error?, selectedPaymentMethod?}>` | Validate all fields before confirming                                             |
| `fetchUpdates`           | `() => Promise<ActionResult>`                     | Re-fetch the session from the server                                              |
| `changeAppearance`       | `(appearance: Appearance) => void`                | Update appearance at runtime                                                      |
| `on`                     | `("change", handler) => void`                     | Listen for session changes (rarely needed in React — state updates automatically) |
| `getElements`            | `() => Elements`                                  | Access the underlying Elements instance                                           |

**`ActionResult` type:**

```typescript
type ActionResult =
  { type: "success"; session: CheckoutSession } | { type: "error"; error: XPayError };
```

**`XPayError` type:**

```typescript
interface XPayError {
  type: string;
  message: string;
  code: string | null;
  declineCode?: string | null;
}
```

### `submit()` and `fetchUpdates()`

Use `submit()` to validate all payment fields before calling `confirm()`. This is useful when you want to validate the form as a separate step (for example, when showing a confirmation dialog).

```tsx
const handleSubmit = async () => {
  // Step 1: validate fields
  const { error, selectedPaymentMethod } = await checkout.submit();
  if (error) {
    showError(error.message);
    return;
  }

  // Step 2: show confirmation dialog
  const confirmed = await showConfirmDialog(selectedPaymentMethod);
  if (!confirmed) return;

  // Step 3: confirm payment
  const result = await checkout.confirm();
  if (result.type === "error") {
    showError(result.error.message);
  }
};
```

Use `fetchUpdates()` to re-fetch the session from the server. This is useful when you know the session has been updated server-side (for example, after adding a shipping address that changes the total).

```tsx
const handleAddressChange = async () => {
  await saveAddressToServer(address);
  const result = await checkout.fetchUpdates();
  if (result.type === "success") {
    // checkout state is now up to date
  }
};
```

## Step 4: Show a Success Page [Server-side + Client-side]

After payment, the customer is redirected to your `afterCompletion.redirect.url`. Retrieve the session from **your server** (using your API key) to display order details.

**Server endpoint:**

```javascript
// Your server — retrieves session using your API key (not from the client SDK)
app.get("/api/order-status", async (req, res) => {
  const response = await fetch(`https://api.xpay.app/checkout/sessions/${req.query.session_id}`, {
    headers: { Authorization: `Bearer ${process.env.XPAY_SECRET_KEY}` },
  });
  const session = await response.json();
  res.json(session);
});
```

**React success page:**

```tsx
function SuccessPage() {
  const [session, setSession] = useState(null);
  const sessionId = new URLSearchParams(window.location.search).get("session_id");

  useEffect(() => {
    fetch(`/api/order-status?session_id=${sessionId}`)
      .then((r) => r.json())
      .then(setSession);
  }, [sessionId]);

  if (!session) return <div>Loading order details...</div>;

  return (
    <div>
      <h1>Payment {session.paymentStatus === "paid" ? "Confirmed" : "Processing"}</h1>

      <p>
        Order Total: {session.currency} {(session.amountTotal / 100).toFixed(2)}
      </p>

      {session.lineItems?.map((item) => (
        <p key={item.id}>
          {item.price?.product?.name} x {item.quantity}
        </p>
      ))}

      {/* Fee breakdown */}
      {session.totalDetails && (
        <dl>
          <dt>Subtotal</dt>
          <dd>{(session.amountSubtotal / 100).toFixed(2)}</dd>

          {session.totalDetails.amountDiscount !== 0 && (
            <>
              <dt>Discount</dt>
              <dd>-{(session.totalDetails.amountDiscount / 100).toFixed(2)}</dd>
            </>
          )}

          {session.totalDetails.amountPlatformFee && (
            <>
              <dt>Processing Fee</dt>
              <dd>{(session.totalDetails.amountPlatformFee / 100).toFixed(2)}</dd>
            </>
          )}

          {session.totalDetails.amountCollectedVat && (
            <>
              <dt>VAT</dt>
              <dd>{(session.totalDetails.amountCollectedVat / 100).toFixed(2)}</dd>
            </>
          )}
        </dl>
      )}
    </div>
  );
}
```

**Session fields for display:**

| Field                                     | Type                                | Description                                    |
| ----------------------------------------- | ----------------------------------- | ---------------------------------------------- |
| `session.status`                          | `'open' \| 'complete' \| 'expired'` | Session status                                 |
| `session.paymentStatus`                   | `'unpaid' \| 'paid'`                | Payment status                                 |
| `session.amountSubtotal`                  | `number`                            | Subtotal before discounts/fees (smallest unit) |
| `session.amountTotal`                     | `number`                            | Total amount charged (smallest unit)           |
| `session.currency`                        | `string`                            | Currency code (e.g., `'EGP'`)                  |
| `session.lineItems`                       | `Array`                             | Line items with product name, quantity, amount |
| `session.totalDetails.amountDiscount`     | `number`                            | Discount amount                                |
| `session.totalDetails.amountShipping`     | `number`                            | Shipping amount                                |
| `session.totalDetails.amountTax`          | `number`                            | Tax amount                                     |
| `session.totalDetails.amountPlatformFee`  | `number`                            | Platform fee (if feesPassThrough enabled)      |
| `session.totalDetails.amountCollectedVat` | `number`                            | Collected VAT                                  |
| `session.customer`                        | `object`                            | Customer name, email, phone                    |
| `session.merchantName`                    | `string`                            | Merchant display name                          |

## Step 5: Handle Webhooks [Server-side]

XPay sends webhook events when payment state changes. Listen for these on your server — don't rely solely on the client-side result.

```javascript
// Your server
app.post("/webhooks/xpay", (req, res) => {
  const event = req.body;

  switch (event.type) {
    case "checkout.session.completed":
      // Payment succeeded — fulfill the order
      // Send confirmation email, update database, start shipping
      fulfillOrder(event.data);
      break;
    case "checkout.session.expired":
      // Session expired without payment
      break;
  }

  res.json({ received: true });
});
```

The webhook is the **source of truth** for order fulfillment. The client-side `confirm()` result is for UX only (showing success/error to the customer). A customer could close their browser after paying but before seeing the success page — the webhook still fires.

---

## API Reference

### `<XPayProvider>`

Wraps your checkout UI. Provides XPay context to all child components.

```tsx
<XPayProvider xpay={xpayPromise} options={{ clientSecret, appearance, locale }}>
  {children}
</XPayProvider>
```

| Prop                   | Type                                            | Description                                                          |
| ---------------------- | ----------------------------------------------- | -------------------------------------------------------------------- |
| `xpay`                 | `XPayInstance \| Promise<XPayInstance> \| null` | XPay instance or promise from `loadXPay()`. Call at module level.    |
| `options`              | `{ clientSecret, appearance?, locale? }`        | Must include the checkout session's `clientSecret`.                  |
| `options.clientSecret` | `string \| Promise<string>`                     | The session's client secret. Accepts a Promise for deferred loading. |
| `options.appearance`   | `Appearance`                                    | Override the session's branding settings at runtime.                 |
| `options.locale`       | `"en" \| "ar"`                                  | Locale for the payment form.                                         |

### `useCheckout()`

The primary hook. Returns a disjoint union — narrow the `type` before accessing data.

```tsx
const checkoutState = useCheckout();
```

**Return type: `UseCheckoutResult`**

```typescript
type UseCheckoutResult =
  | { type: "loading" }
  | { type: "error"; error: { message: string } }
  | { type: "success"; checkout: Checkout };
```

After narrowing to `type: "success"`, the `checkout` object contains all session fields and action methods merged together.

**Session fields** (on `checkout`):

| Field            | Type                  | Description                                                                    |
| ---------------- | --------------------- | ------------------------------------------------------------------------------ |
| `id`             | `string`              | Session ID                                                                     |
| `amountSubtotal` | `number`              | Subtotal before discounts/fees                                                 |
| `amountTotal`    | `number`              | Total amount (smallest unit)                                                   |
| `currency`       | `string`              | Currency code                                                                  |
| `merchantName`   | `string`              | Merchant display name                                                          |
| `livemode`       | `boolean`             | Whether live mode                                                              |
| `expiresAt`      | `string`              | Session expiration                                                             |
| `status`         | `SessionStatus`       | `{type: "open"}` \| `{type: "expired"}` \| `{type: "complete", paymentStatus}` |
| `canConfirm`     | `boolean`             | Whether the session is ready for confirmation                                  |
| `paymentMethods` | `PaymentMethodInfo[]` | Available payment methods                                                      |
| `lineItems`      | `LineItem[]`          | Line items                                                                     |
| `totalDetails`   | `TotalDetails`        | Amounts breakdown                                                              |
| `fees`           | `Fees`                | Fee breakdown                                                                  |
| `discounts`      | `Discount[]`          | Applied discounts                                                              |

**Action methods** (on `checkout`):

| Method                   | Signature                                         | Description                                                                                    |
| ------------------------ | ------------------------------------------------- | ---------------------------------------------------------------------------------------------- |
| `confirm`                | `(options?) => Promise<ActionResult>`             | Confirm payment                                                                                |
| `applyPromotionCode`     | `(code) => Promise<ActionResult>`                 | Apply a promo code                                                                             |
| `removePromotionCode`    | `() => Promise<ActionResult>`                     | Remove promo code                                                                              |
| `updateLineItemQuantity` | `({lineItem, quantity}) => Promise<ActionResult>` | Update line item quantity                                                                      |
| `submit`                 | `() => Promise<{error?, selectedPaymentMethod?}>` | Validate fields                                                                                |
| `fetchUpdates`           | `() => Promise<ActionResult>`                     | Re-fetch session                                                                               |
| `changeAppearance`       | `(appearance) => void`                            | Update appearance                                                                              |
| `on`                     | `("change", handler) => void`                     | Listen for session changes                                                                     |
| `on`                     | `("error", handler) => void`                      | Listen for unsolicited errors (session expired during internal updates, BIN detection failure) |
| `getElements`            | `() => Elements`                                  | Access underlying Elements                                                                     |

### `<PaymentElement>`

Renders the payment method selector and card form.

```tsx
<PaymentElement
  options={{ layout: "accordion" }}
  onChange={(e) => console.log(e.complete, e.value.type)}
/>
```

| Prop            | Type                                                      | Description                                                           |
| --------------- | --------------------------------------------------------- | --------------------------------------------------------------------- |
| `options`       | `{ layout?, defaultPaymentMethod?, paymentMethodOrder? }` | Configuration                                                         |
| `onChange`      | `(event: PaymentElementChangeEvent) => void`              | Form state changed                                                    |
| `onReady`       | `() => void`                                              | Element initialized (async, fires from `XPAY_SDK_INITIALIZED`)        |
| `onLoaderStart` | `() => void`                                              | Loader animation started (fires synchronously when iframe is created) |
| `onLoadError`   | `(event) => void`                                         | Element failed to load                                                |
| `className`     | `string`                                                  | CSS class for the container div                                       |
| `id`            | `string`                                                  | ID for the container div                                              |

### `<CheckoutButton>`

Opens the drop-in checkout modal on click.

```tsx
<CheckoutButton
  clientSecret="cs_test_abc_secret_xyz"
  checkoutOptions={{ onComplete: (r) => router.push("/success") }}
>
  Pay Now
</CheckoutButton>
```

### `useXPay()` / `useElements()` / `useConfirmPayment()`

Lower-level hooks for advanced use cases.

---

## Integration Patterns

### Drop-in Checkout (Simplest)

A button that opens the full checkout in a modal overlay. No form needed.

```tsx
<XPayProvider xpay={xpayPromise}>
  <CheckoutButton
    clientSecret={clientSecret}
    checkoutOptions={{
      onComplete: (result) => (window.location.href = `/orders/${orderId}`),
      onClose: () => console.log("Closed"),
    }}
  >
    Subscribe Now
  </CheckoutButton>
</XPayProvider>
```

### Dark Mode Toggle

Sync XPay appearance with your site's theme at runtime using `changeAppearance()`.

```tsx
function ThemeAwareCheckout() {
  const checkoutState = useCheckout();
  const { theme } = useTheme(); // your app's theme hook

  useEffect(() => {
    if (checkoutState.type === "success") {
      checkoutState.checkout.changeAppearance({ colorMode: theme as "light" | "dark" });
    }
  }, [theme, checkoutState]);

  return <PaymentElement />;
}
```

### Redirect Behavior

By default, `confirm()` returns the result to your code (`redirect: "if_required"`). Use `redirect: "always"` to redirect to the server's `afterCompletion.redirect.url` after success.

```tsx
// Default: returns result to your code (no redirect)
const result = await checkout.confirm({ customerDetails: { email, name } });
if (result.type === "success") {
  router.push("/thank-you");
}

// Redirect after success instead:
await checkout.confirm({
  customerDetails: { email, name },
  redirect: "always",
});
// ^ If successful, the page navigates away. Code below only runs on error.
```

| `redirect`        | Behavior                                                  |
| ----------------- | --------------------------------------------------------- |
| Not set (default) | `"if_required"` — returns result to your code             |
| `"always"`        | Redirects to the session's `afterCompletion.redirect.url` |
| `"if_required"`   | Returns result to your code — no redirect                 |

Your server sets that URL when it creates the session. XPay navigates there unchanged, appending nothing.

---

## Appearance

Override the session's `brandingSettings` at runtime. Uses the same shape.

```tsx
<XPayProvider
  xpay={xpayPromise}
  options={{
    clientSecret,
    appearance: {
      colorMode: "dark",
      borderStyle: "rounded",
      inputStyle: "outlined",
      inputSize: "large",
      spacing: "normal",
      formLayout: "compact",
      colors: { primary: "#0066FF", background: "#1a1a1a" },
      fontFamily: "Inter, sans-serif",
    },
  }}
>
```

---

## TypeScript

All components and hooks are fully typed. `@xpayeg/react` re-exports key SDK types for convenience:

```tsx
import type { Checkout, CheckoutSession, CheckoutActions, UseCheckoutResult } from "@xpayeg/react";

// Or import additional types from @xpayeg/sdk directly:
import type {
  ActionResult,
  XPayError,
  PaymentMethodInfo,
  CustomerDetails,
  Appearance,
  SessionStatus,
  CheckoutLineItem,
  CheckoutTotalDetails,
  CheckoutFees,
  CheckoutDiscount,
} from "@xpayeg/sdk";
```

## Development checks

From the monorepo, run `pnpm --filter @xpayeg/react test` for the React DOM
contracts. These use jsdom and a synthetic Elements boundary to check option
updates, StrictMode teardown, and current callbacks without subscription churn.
They do not create checkout sessions or submit payments. Run
`pnpm --filter @xpayeg/react typecheck` for source, test, and public consumer types.
