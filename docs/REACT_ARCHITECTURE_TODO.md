# React architecture refactor: detailed TODO

This document turns the current React architecture review into an actionable
refactor plan. It is intentionally incremental: each phase should leave the
application deployable and should avoid a framework rewrite.

The separate [2FA security plan](./TWO_FACTOR_SECURITY_TODO.md) is the highest
priority because the current frontend gate is not a security boundary.

## Goals

- Keep client-only UI state separate from server-owned data.
- Reduce prop drilling without hiding dependencies behind broad object spreads.
- Make runtime API data match the application's TypeScript types.
- Keep feature hooks focused and testable.
- Use component patterns only when they improve a real consumer API.
- Establish tests around auth transitions and the Explore feature.

## Non-goals

- Migrating the Vite application to Next.js or React Server Components.
- Introducing Redux or another global state library without a measured need.
- Wrapping every component in `memo` or every function in `useCallback`.
- Converting fixed page layouts into compound components for stylistic reasons.
- Generating client UUIDs for records that already have stable backend IDs.

## Priority overview

| Priority | Workstream | Expected result |
| --- | --- | --- |
| P0 | Enforce 2FA server-side | A verified factor is required for protected operations |
| P1 | Reduce `AuthProvider` responsibilities | Context owns auth session; React Query owns remote data |
| P1 | Validate API boundaries with Zod | Invalid server payloads cannot silently enter the UI |
| P2 | Split the Explore controller | Smaller hooks and explicit feature view models |
| P2 | Remove pass-through and dead props | Clearer component dependencies |
| P3 | Improve primitives and composition | Delegated props only where they are semantically correct |
| P3 | Measure and improve performance | Route splitting and evidence-based render optimization |
| P2 | Add meaningful frontend tests | Refactors are protected by behavior-level tests |

---

## P0: complete the 2FA security boundary

Follow [TWO_FACTOR_SECURITY_TODO.md](./TWO_FACTOR_SECURITY_TODO.md).

- [ ] Agree which backend operations require a verified second factor.
- [ ] Represent successful verification with server-controlled state.
- [ ] Enforce that state in backend middleware and/or Supabase RLS.
- [ ] Ensure direct Supabase requests cannot bypass protected operations.
- [ ] Make status failures fail closed for accounts that require 2FA.
- [ ] Add tests covering bypass attempts and verification expiry.

Do this before treating the frontend overlay as an authentication guarantee.

---

## P1: make `AuthProvider` session-focused

### Current problem

`AuthProvider` currently manages several different concerns:

- Supabase session subscription
- profile fetching and refresh
- OAuth identity persistence
- 2FA status fetching and commands
- browser session markers
- rendering the 2FA gate

This creates manual async coordination and allows profile requests to complete
after the active user has changed. It also means any provider state change can
notify every `useAuth()` consumer.

### Proposed ownership

| Data or behavior | Recommended owner |
| --- | --- |
| Supabase session and user ID | Auth session context |
| Sign out and session subscription | Auth session context |
| Current profile | React Query keyed by user ID |
| 2FA status | React Query keyed by user ID |
| 2FA setup/verify/disable | React Query mutations |
| Explore profiles/posts/comments | Existing React Query feature hooks |
| Dialog/sidebar/lightbox visibility | Local feature state or reducer |
| OAuth persistence orchestration | Dedicated hook/service triggered by auth events |

### Suggested API

The names can change, but consumers should be able to request only what they
need:

```tsx
const { session, userId, loading, logout } = useAuthSession();
const { data: profile, isPending: profilePending } = useCurrentProfileQuery();
const { data: twoFactorStatus } = useTwoFactorStatusQuery();
```

Avoid returning profile and 2FA server data as part of one large context value.

### Implementation checklist

- [ ] Inventory all `useAuth()` consumers with `rg -n "useAuth\\(" frontend/src`.
- [ ] Add a narrow `AuthSessionContext`.
- [ ] Keep the Supabase auth subscription and `logout` in the session provider.
- [ ] Memoize the session context value so reference identity changes only when
      one of its public values changes.
- [ ] Move the context and consumer hook into separate modules if needed to
      remove the Fast Refresh warning.
- [ ] Add `useCurrentProfileQuery(userId)` using a query key that contains the
      active user ID.
- [ ] Replace `refreshProfile()` with targeted query invalidation.
- [ ] Add `useTwoFactorStatusQuery(userId)`.
- [ ] Convert setup, verification, resend, and disable operations to mutations.
- [ ] Invalidate the 2FA status query after successful 2FA mutations.
- [ ] Remove the manual profile request race or obsolete it by completing the
      query migration.
- [ ] Move OAuth identity persistence into a focused hook or service with clear
      idempotency behavior.
- [ ] Update feature consumers to use the narrowest applicable hook.
- [ ] Remove the old context fields only after all consumers have migrated.

### Query-key suggestions

```ts
export const authKeys = {
  all: ["auth"] as const,
  profile: (userId: string) => [...authKeys.all, "profile", userId] as const,
  twoFactor: (userId: string) =>
    [...authKeys.all, "two-factor", userId] as const,
};
```

Never use a constant query key for user-specific data. A user switch must select
a different cache entry.

### Acceptance criteria

- [ ] User A's delayed profile response cannot be rendered after switching to
      User B.
- [ ] Logging out clears session-owned UI immediately.
- [ ] Profile updates invalidate and refetch the correct profile query.
- [ ] 2FA mutations refresh status without a manually exposed
      `refreshTwoFactorStatus()` method.
- [ ] Components that only need `userId` do not subscribe to profile or 2FA
      details through one combined context.

---

## P1: validate API data at runtime with Zod

### Current problem

`twoFactorRepository.apiRequest<T>()` parses JSON and ultimately returns
`payload as T`. The generic only tells TypeScript what the caller hopes the
server returned; it does not verify the runtime payload.

### Proposed shape

Pass a schema into the boundary function:

```ts
import { z } from "zod";

const twoFactorStatusSchema = z.object({
  enabled: z.boolean(),
  requiresVerification: z.boolean(),
  hasTrustedDevice: z.boolean(),
  challengeSent: z.boolean().optional(),
  emailMasked: z.string().nullable(),
});

async function apiRequest<T>(
  path: string,
  schema: z.ZodType<T>,
  init?: RequestInit,
): Promise<T> {
  const response = await fetch(path, init);
  const payload: unknown = await response.json();

  if (!response.ok) {
    throw parseApiError(response.status, payload);
  }

  return schema.parse(payload);
}
```

The exact error wrapper should preserve a useful error code, HTTP status, and a
safe user-facing message.

### Implementation checklist

- [ ] Define a reusable API error payload schema.
- [ ] Define schemas for 2FA status, setup, verification, resend, and disable
      responses.
- [ ] Change `apiRequest` to accept and parse a schema.
- [ ] Convert JSON payload variables from asserted records to `unknown`.
- [ ] Map `ZodError` to an observable application error that includes endpoint
      context without logging secrets.
- [ ] Add tests for missing, incorrectly typed, and additional payload fields.
- [ ] Review other custom HTTP repositories for `as T`, manual generics, or
      unchecked `response.json()`.
- [ ] Prioritize external/custom APIs. Do not add duplicate parsing everywhere
      solely for ceremony.

### Acceptance criteria

- [ ] A malformed successful response fails at the repository boundary.
- [ ] Rendering code never receives a partially valid 2FA object.
- [ ] Type inference comes from the schema where practical:

```ts
type TwoFactorStatus = z.infer<typeof twoFactorStatusSchema>;
```

---

## P2: split `useExplorePageController`

### Current problem

The controller currently returns roughly forty values, including raw mutation
objects. `ExplorePage` then destructures those values and rebuilds prop objects
for multiple descendants.

The issue is not the number of lines by itself. The controller's public API
exposes implementation details and combines unrelated reasons to change.

### Suggested feature boundaries

```ts
function useExploreSearchController() {
  return {
    query,
    setQuery,
    results,
    loading,
    visible,
  };
}

function useExploreComposerController() {
  return {
    content,
    setContent,
    imagePreview,
    selectImage,
    removeImage,
    submit,
    submitting,
  };
}

function useExploreCommentsController() {
  return {
    activePostId,
    comments,
    draft,
    setDraft,
    open,
    close,
    submit,
    submittingPostId,
  };
}

function useExploreFeedController() {
  return {
    posts,
    loading,
    loadMoreSentinelRef,
    fetchingNextPage,
    deletingPostId,
    updatingPostId,
    toggleLike,
    deletePost,
    editPost,
  };
}
```

These examples are view models, not mandatory names. Each return object should
contain only what its consumers need.

### Reducer candidate

The coordinated comments/lightbox transitions are a reasonable `useReducer`
candidate because several state values reset together:

- opening media closes inline comments
- switching posts clears the draft
- deleting the active post closes comments and lightbox
- closing the lightbox clears its active ID

Example action vocabulary:

```ts
type ExploreOverlayAction =
  | { type: "comments.open"; postId: string }
  | { type: "comments.close" }
  | { type: "lightbox.open"; postId: string }
  | { type: "lightbox.close" }
  | { type: "draft.change"; value: string }
  | { type: "post.deleted"; postId: string };
```

Do not put React Query's post/comment data into this reducer. It should manage
only coordinated client UI state.

### Implementation checklist

- [ ] Add characterization tests for the current Explore interactions before
      moving state.
- [ ] Extract the search controller without changing behavior.
- [ ] Extract composer state and commands.
- [ ] Add object URL cleanup:
  - [ ] revoke the previous URL when selecting a replacement image
  - [ ] revoke the active URL when removing an image
  - [ ] revoke the active URL on unmount
- [ ] Extract comments/lightbox UI state, using a reducer if it makes the
      transitions clearer.
- [ ] Keep feed and comment server state in React Query.
- [ ] Stop exposing complete mutation objects from the page controller.
- [ ] Return semantic fields such as `submitting`, `deletingPostId`, and
      `deletePost`.
- [ ] Remove or reduce the top-level controller after the focused hooks are in
      place.
- [ ] Ensure a category filter is reset or validated when changing challenge
      tabs so a stale category cannot make a populated tab appear empty.

### Acceptance criteria

- [ ] No Explore hook exposes an entire React Query mutation object solely so a
      page can read `isPending` or `variables`.
- [ ] Each hook has a small, named responsibility.
- [ ] Opening, closing, switching, and deleting posts produces the same visible
      behavior as before.
- [ ] Repeated image selection does not leak object URLs.

---

## P2: remove avoidable prop drilling and pass-through components

### Auth-derived props

Feature-level components that are inherently tied to the signed-in application
can consume a narrow auth hook directly. Candidates include:

- `ExploreComposer`
- `ExploreLeftSidebar`
- `ExploreRightSidebar`
- potentially `ExplorePostsSection`

Do not turn generic presentational components into auth-aware components. For
example, a reusable avatar or post card should still receive the data required
to render.

### Current cleanup checklist

- [ ] Remove unused `profilesLoading`, `profilesCount`, and `postsCount` from
      `ExploreRightSidebar`.
- [ ] Decide whether `ExploreFeedColumn` should:
  - [ ] own its relevant feature hooks, or
  - [ ] accept composed children/slots, or
  - [ ] be removed as an unnecessary pass-through layer.
- [ ] Avoid forwarding `composerProps` and `postsSectionProps` through a
      component that does not understand or transform them.
- [ ] Connect the sidebar "sign up" and "claim your Tierd" buttons to real
      actions, or render them as non-interactive copy until implemented.
- [ ] Audit other feature wrappers that accept prop bags only to spread them
      into one child.

### Broad spreads to avoid

Do not do this:

```tsx
const controller = useExplorePageController();
return <ExploreFeedColumn {...controller} />;
```

It makes dependencies invisible and lets unrelated controller changes leak into
the component API.

An intentional, exact view model is acceptable:

```tsx
const composer = useExploreComposerController();
return <ExploreComposer {...composer} />;
```

The difference is that `composer` is already the complete and narrowly typed
public contract of `ExploreComposer`.

---

## P3: use delegated props for primitives, not business state

`...delegated` is simply a descriptive name for rest props. It is appropriate
when a component intentionally mirrors or decorates one underlying primitive.

Good candidates:

- `InlineSpinner`
- `SkeletonLine`
- `SkeletonAvatar`
- `SkeletonButton`
- other loading primitives in `components/ui/loading.tsx`

Example:

```tsx
type SkeletonLineProps = React.ComponentPropsWithoutRef<typeof Skeleton> & {
  tone?: "dark" | "light";
};

export function SkeletonLine({
  className,
  tone = "dark",
  ...delegated
}: SkeletonLineProps) {
  return (
    <Skeleton
      {...delegated}
      tone={tone}
      className={cn("h-3.5", className)}
    />
  );
}
```

Put delegated props before protected explicit props when the wrapper must retain
control of those values. Decide deliberately whether consumers may override
`role`, `aria-hidden`, or other accessibility behavior.

### Checklist

- [ ] Update primitive prop types using `ComponentPropsWithoutRef`.
- [ ] Forward valid DOM/custom primitive attributes.
- [ ] Preserve class merging.
- [ ] Document which accessibility attributes are fixed versus overridable.
- [ ] Do not spread auth state, mutation objects, or page controllers into DOM
      elements.

---

## P3: compound components decision guide

Do not implement compound components as a general cleanup.

Use the pattern only when all or most of these are true:

- a component family is reused in several places
- children share implicit state
- consumers need to reorder, omit, or compose child sections
- repeated prop threading makes the consumer API worse
- the shared context has a narrow, stable purpose

Radix dialog primitives already demonstrate an appropriate compound API:

```tsx
<Dialog>
  <DialogTrigger />
  <DialogContent />
</Dialog>
```

### Possible future candidate

A reusable feed shell could eventually become:

```tsx
<Feed>
  <Feed.Header />
  <Feed.Composer />
  <Feed.Posts />
  <Feed.LoadMore />
</Feed>
```

Only build this if another page needs a rearrangeable feed family. The current
fixed Explore layout alone does not justify another context.

### State reducer pattern

Do not add a public `stateReducer` prop to application-specific components. It
is useful when building a reusable library component whose consumers genuinely
need to intercept internal transitions. Internal `useReducer` and the public
state-reducer pattern are separate decisions.

---

## P3: performance and bundle polish

### Current observations

- The production build reports a JavaScript chunk over 1 MB before gzip.
- All route components are imported eagerly in `App.tsx`.
- Search is already debounced, so `useDeferredValue` is not automatically an
  improvement.
- Fixed skeleton arrays use index keys appropriately because those placeholders
  are never reordered or edited.

### Checklist

- [ ] Establish a bundle-size baseline.
- [ ] Lazy-load page-level route components with `React.lazy`.
- [ ] Add a route-level `Suspense` fallback that matches the application shell.
- [ ] Rebuild and record the initial chunk reduction.
- [ ] Use the React Profiler before adding `memo` or more `useCallback`.
- [ ] Stabilize context values where identity directly triggers context
      propagation.
- [ ] Consider `useDeferredValue` only for measured expensive client rendering,
      not as a replacement for network-query debouncing.
- [ ] Continue using backend entity IDs for list keys.
- [ ] Generate a UUID once only for genuinely client-created entities that do
      not yet have a backend ID.

### Acceptance criteria

- [ ] Initial navigation does not download every page implementation.
- [ ] Loading a lazy route has an intentional fallback.
- [ ] Performance optimizations include a recorded reason or measurement.

---

## P2: test and tooling coverage

The frontend currently has only an example test. Build and TypeScript success do
not protect auth transitions or coordinated feature behavior.

### Suggested test order

- [ ] Auth session provider:
  - [ ] initial session resolution
  - [ ] sign-in event
  - [ ] sign-out event
  - [ ] user switch while a profile request is pending
- [ ] Profile query:
  - [ ] query disabled without user ID
  - [ ] cache separated by user ID
  - [ ] invalidation after profile update
- [ ] 2FA:
  - [ ] unknown/loading status behavior
  - [ ] status request failure
  - [ ] verification success
  - [ ] verification expiry
  - [ ] protected request rejection without verification
- [ ] Explore UI reducer/controller:
  - [ ] opening comments
  - [ ] switching comment targets
  - [ ] opening and closing the lightbox
  - [ ] deleting the active post
  - [ ] clearing drafts at the intended transitions
- [ ] Composer:
  - [ ] replacing an image revokes the old preview URL
  - [ ] unmounting revokes the active preview URL
  - [ ] successful submission clears state
  - [ ] failed submission preserves user input
- [ ] API boundary:
  - [ ] valid Zod response
  - [ ] malformed success response
  - [ ] structured API error

### Routine verification commands

```bash
npm --prefix frontend run lint
npm --prefix frontend exec -- tsc --noEmit --project frontend/tsconfig.json
npm --prefix frontend run test
npm --prefix frontend run build
npm --prefix backend test
```

Track existing lint warnings separately so new warnings are not normalized as
expected output.

---

## Suggested delivery sequence

Keep pull requests small enough to review:

1. **Security contract and tests**
   - Define protected operations and server verification behavior.
2. **Auth session extraction**
   - Add narrow context without moving every consumer at once.
3. **Profile query migration**
   - Remove manual profile fetching and its race.
4. **2FA query/mutation migration**
   - Preserve the UI while switching remote-state ownership.
5. **Zod API boundary**
   - Validate 2FA responses and errors.
6. **Explore behavior tests**
   - Characterize current transitions.
7. **Explore controller split**
   - Extract search, composer, feed, and overlay responsibilities.
8. **Prop cleanup and primitive delegation**
   - Remove dead/pass-through props and improve loading primitives.
9. **Route code splitting**
   - Address the measured initial bundle warning.

## Final definition of done

- [ ] Protected operations cannot be performed with only an unverified primary
      session.
- [ ] Auth context does not own profile or 2FA server state.
- [ ] Custom HTTP success payloads are runtime-validated.
- [ ] Explore has focused state owners rather than one broad controller API.
- [ ] Rest/delegated props are used only for intentional primitive forwarding.
- [ ] Compound components exist only where multiple consumers benefit from the
      composition API.
- [ ] Frontend tests cover the main auth, 2FA, and Explore transitions.
- [ ] TypeScript, lint, frontend tests, backend tests, and production build pass.
