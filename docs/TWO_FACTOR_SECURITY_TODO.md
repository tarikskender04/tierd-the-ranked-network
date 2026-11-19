# Email 2FA security plan

## Why this is separate

The current email 2FA experience is implemented primarily as a frontend gate.
Once Supabase has issued a normal session, rendering an overlay does not prevent
that session from calling Supabase or backend endpoints directly.

The local development flag is guarded by `import.meta.env.DEV`, which prevents
that specific switch from enabling the bypass in a production Vite build.
However, the production security model still needs server-side enforcement.

This document focuses on the security contract rather than the appearance of the
verification modal.

## Current risks

### 1. The frontend is not a security boundary

The application renders its children and conditionally overlays
`TwoFactorGate`. A user who has the primary session token can bypass React and
make requests directly.

### 2. Status lookup currently fails open

The default 2FA state has `enabled: false`. If the status request fails, the
error is stored but the gate condition remains false. An enabled account can
therefore receive normal application access when status cannot be determined.

### 3. `sessionStorage` can be modified by the browser user

`tierd.2fa.session_verified.<userId>` is useful as UI state, but it cannot prove
verification to the server.

### 4. Direct Supabase access complicates enforcement

Even if Express middleware checks a trusted 2FA cookie, frontend code that
writes directly to Supabase can bypass that middleware unless RLS also knows and
requires the verified state.

## Required security properties

- A primary Supabase session alone is insufficient for protected operations
  when email 2FA is enabled.
- The proof of second-factor verification is created and validated by the
  server.
- Client-controlled storage is never treated as proof.
- Verification expires according to an explicit policy.
- Logout, password reset, account recovery, and disabling 2FA revoke applicable
  trusted state.
- Status-service failure does not silently grant protected access.
- Development bypasses cannot be enabled in a production deployment.
- Rate limiting and challenge expiry are enforced server-side.

## Step 1: define what 2FA protects

Do not begin with middleware before defining the policy.

- [ ] List read-only operations that may remain available before verification.
- [ ] List protected writes, for example:
  - [ ] profile/security setting changes
  - [ ] linking or unlinking OAuth identities
  - [ ] storing or removing OAuth tokens
  - [ ] changing email or account recovery details
  - [ ] deleting posts, projects, submissions, or the account
  - [ ] disabling 2FA
- [ ] Decide whether ordinary feed posting/commenting requires verification.
- [ ] Decide whether unverified sessions may view any private account data.
- [ ] Document the policy as a backend authorization rule, not only a UI rule.

## Step 2: choose an enforcement architecture

Choose one coherent approach and document why.

### Option A: route protected operations through Express

After a successful challenge, Express creates a short-lived, signed,
server-verifiable proof, preferably in an `HttpOnly`, `Secure`, appropriately
scoped cookie. Protected Express routes require both:

1. a valid Supabase bearer token
2. valid 2FA verification/trust state for that same user

This is straightforward for backend endpoints, but all protected direct
Supabase operations must be moved behind the backend or separately protected by
RLS.

### Option B: enforce a server-issued claim in Supabase RLS

A server-controlled claim or equivalent database-verifiable state represents
the verified factor, and protected RLS policies require it.

This can preserve direct Supabase access, but claim issuance, refresh, expiry,
and revocation must be designed carefully. Never allow the frontend to write its
own verified claim.

### Option C: hybrid

Use Express for sensitive commands and Supabase directly for public or
non-sensitive reads. This is often the most incremental option, provided the
boundary is explicit and no protected direct writes remain.

### Architecture decision checklist

- [ ] Identify every direct Supabase write performed by the frontend.
- [ ] Mark which writes are protected.
- [ ] Choose Express, RLS, or hybrid enforcement.
- [ ] Write an architecture decision record before implementation.
- [ ] Define verification lifetime and trusted-device lifetime separately.
- [ ] Define revocation behavior.

## Step 3: represent verified state safely

- [ ] Bind verified state to the authenticated Supabase user ID.
- [ ] Use cryptographically random opaque tokens or server-signed state.
- [ ] Store only a token hash server-side when using opaque tokens.
- [ ] Make the browser cookie `HttpOnly`.
- [ ] Make it `Secure` in production.
- [ ] Select `SameSite` according to the actual frontend/backend deployment.
- [ ] Scope `Domain` and `Path` as narrowly as the deployment permits.
- [ ] Give session verification and "trust this device" distinct expiry rules.
- [ ] Revoke applicable tokens on logout and when 2FA is disabled.
- [ ] Consider revocation after password reset, email change, or suspected
      account compromise.

The frontend may keep a non-authoritative UI hint, but the backend must ignore it
for authorization.

## Step 4: add server authorization middleware

Suggested conceptual pipeline:

```text
request
  -> validate Supabase bearer token
  -> load account security policy
  -> if 2FA disabled: continue
  -> validate server-controlled 2FA proof for the same user
  -> continue or return 403 second_factor_required
```

- [ ] Extract the duplicated bearer-token caller resolution into shared auth
      middleware.
- [ ] Add a `requireSecondFactor` middleware or equivalent policy function.
- [ ] Return a stable machine-readable error such as
      `second_factor_required`.
- [ ] Do not reveal sensitive account state in error details.
- [ ] Apply the policy to every protected route.
- [ ] Verify there is no equivalent direct Supabase path that bypasses it.
- [ ] Log security events without logging bearer tokens, verification codes, or
      raw trusted-device tokens.

## Step 5: make frontend status explicit

Use a state model that distinguishes "not loaded" from "disabled":

```ts
type TwoFactorGateState =
  | { status: "loading" }
  | { status: "disabled" }
  | { status: "verification-required"; emailMasked: string | null }
  | { status: "verified" }
  | { status: "error"; message: string };
```

- [ ] Remove defaults that equate unknown status with disabled.
- [ ] Keep sensitive UI/actions blocked while status is loading.
- [ ] Handle status errors explicitly.
- [ ] When a protected request returns `second_factor_required`, open the gate
      even if local UI state believed the session was verified.
- [ ] Retry status safely or offer logout/recovery rather than silently
      continuing.
- [ ] Treat the UI gate as guidance; keep server checks authoritative.

## Step 6: challenge and resend hardening

- [ ] Confirm verification codes are stored hashed.
- [ ] Confirm codes expire after a short, documented TTL.
- [ ] Invalidate a code after successful use.
- [ ] Rate-limit challenge creation per user, IP, and destination as
      appropriate.
- [ ] Rate-limit verification attempts.
- [ ] Introduce a maximum attempt count per challenge.
- [ ] Prevent resend from creating an unlimited number of simultaneously valid
      challenges.
- [ ] Return generic failure messages that do not help enumerate accounts.
- [ ] Ensure email sending failures do not leave misleading UI state.
- [ ] Add operational metrics for sends, verification success/failure, rate
      limits, and provider failures.

## Step 7: local development behavior

- [ ] Keep `VITE_DISABLE_2FA_GATE` documented as local UI behavior only.
- [ ] Preserve the `import.meta.env.DEV` build-time guard.
- [ ] Ensure backend enforcement does not accidentally accept a frontend Vite
      variable.
- [ ] If backend developers require a bypass, create a separate backend-only
      flag that:
  - [ ] defaults to disabled
  - [ ] refuses to start or refuses the bypass in production
  - [ ] emits a prominent startup warning
  - [ ] is never committed as enabled
- [ ] Add a production-build test proving the frontend bypass condition is
      false.

## Step 8: security tests

### Backend route tests

- [ ] Primary token without 2FA proof is rejected for a protected operation.
- [ ] A valid proof for the same user is accepted.
- [ ] A proof belonging to another user is rejected.
- [ ] An expired proof is rejected.
- [ ] A revoked proof is rejected.
- [ ] A malformed proof is rejected.
- [ ] A disabled-2FA account follows the documented policy.
- [ ] Disabling 2FA revokes trusted devices.
- [ ] Logout/recovery revokes the expected state.
- [ ] Rate limits apply to resend and verification attempts.

### Bypass tests

- [ ] Editing `sessionStorage` does not authorize a protected request.
- [ ] Omitting the React gate does not authorize a protected request.
- [ ] Calling the backend directly does not bypass verification.
- [ ] Calling Supabase directly cannot perform a protected write.
- [ ] Status-service failure does not grant protected access.

### Frontend behavior tests

- [ ] Loading status does not flash protected controls.
- [ ] `second_factor_required` opens the verification experience.
- [ ] Successful verification retries or enables the intended operation.
- [ ] Status errors provide a retry and logout/recovery path.
- [ ] Local development bypass affects only the intended UI behavior.

## Rollout plan

1. **Inventory and policy**
   - Identify protected operations and direct Supabase paths.
2. **Backend proof and middleware**
   - Implement verification state without enabling enforcement globally.
3. **Protect one low-risk operation**
   - Validate telemetry and frontend error handling.
4. **Protect all sensitive backend operations**
   - Add regression tests for each route family.
5. **Close direct Supabase bypasses**
   - Move writes behind Express or update RLS.
6. **Change frontend status to explicit fail-closed states**
   - Retain recovery and logout paths.
7. **Enable enforcement for all 2FA-enabled accounts**
   - Monitor rejection, verification, and email-delivery rates.

## Definition of done

- [ ] The threat model and protected-operation list are documented.
- [ ] A server-controlled second-factor proof is bound to the authenticated
      user.
- [ ] Every protected backend and Supabase path enforces the policy.
- [ ] Client storage cannot grant authorization.
- [ ] Unknown/error status does not silently become "2FA disabled."
- [ ] Expiry, trust, and revocation policies are explicit and tested.
- [ ] Rate limiting and challenge-attempt controls are active.
- [ ] Production cannot enable a development bypass.
- [ ] Automated tests demonstrate that direct requests cannot bypass 2FA.

