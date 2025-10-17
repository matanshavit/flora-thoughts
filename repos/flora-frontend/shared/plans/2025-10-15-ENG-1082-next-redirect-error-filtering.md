# NEXT_REDIRECT Error Filtering Implementation Plan

## Overview

Fix the false error reporting of Next.js internal redirect control flow mechanisms (NEXT_REDIRECT) that are currently being logged as application errors in Sentry and Axiom.

## Current State Analysis

The join-workspace route at `src/app/(with-migration)/join-workspace/[invitationCode]/page.tsx` correctly implements server-side redirects using `redirect()` from Next.js. However, Next.js internally uses a `NEXT_REDIRECT` error for control flow, which is being caught by our global error handler and reported as an actual error.

### Key Discoveries:

- Global error handler at `src/app/global-error.tsx:30-48` logs ALL errors to Sentry and Axiom
- The `error.digest` field contains "NEXT_REDIRECT" for redirect control flow (line 45)
- No existing filtering for Next.js internal errors exists
- 9+ other server routes use the same redirect pattern successfully

## Desired End State

After implementation, NEXT_REDIRECT and NEXT_NOT_FOUND errors should be filtered out and not reported to Sentry or Axiom, while actual application errors continue to be logged normally.

### Verification:

- Join-workspace invitations work without error pages appearing
- NEXT_REDIRECT errors no longer appear in Sentry dashboard
- NEXT_REDIRECT errors no longer appear in Axiom logs
- Actual application errors continue to be reported normally

## What We're NOT Doing

- NOT changing the server-side redirect pattern (it's already correct)
- NOT adding client-side rendering to avoid the issue
- NOT modifying individual page components
- NOT creating new error handling infrastructure
- NOT changing the join-workspace or join-project page logic

## Implementation Approach

Add minimal filtering at the global error handler level to check for Next.js internal control flow errors before logging. This is a 4-line change that fixes the issue globally for all routes.

## Phase 1: Filter NEXT_REDIRECT in Global Error Handler

### Overview

Add a check for NEXT_REDIRECT and NEXT_NOT_FOUND in the global error handler to prevent logging these control flow mechanisms as errors.

### Changes Required:

#### 1. Global Error Handler

**File**: `src/app/global-error.tsx`
**Changes**: Add filtering before Sentry and Axiom logging

```tsx
useEffect(() => {
  // Filter out Next.js internal control flow errors
  if (error.digest?.startsWith("NEXT_REDIRECT") || error.digest?.startsWith("NEXT_NOT_FOUND")) {
    return; // These are not errors, they're control flow
  }

  Sentry.captureException(error);

  log.logHttpRequest(
    LogLevel.error,
    error.message,
    {
      host: window.location.href,
      path: pathname,
      statusCode: status,
    },
    {
      error: error.name,
      cause: error.cause,
      stack: error.stack,
      digest: error.digest,
    },
  );
}, [error]);
```

### Success Criteria:

#### Automated Verification:

- [x] Type checking passes: `pnpm typecheck`
- [x] Build completes successfully: `pnpm build`

#### Manual Verification:

- [ ] Join-workspace invitation links work without showing error page
- [ ] Join-project invitation links work without showing error page
- [ ] No NEXT_REDIRECT errors appear in browser console
- [ ] Actual errors (e.g., invalid invitation codes) still show error UI
- [ ] Check Sentry dashboard confirms no new NEXT_REDIRECT errors
- [ ] Check Axiom logs confirms no new NEXT_REDIRECT errors

**Implementation Note**: After completing this phase and all automated verification passes, test with actual invitation links to confirm the fix works before proceeding to the optional Phase 2.

---

## Phase 2: Add Sentry beforeSend Hook (Optional Enhancement)

### Overview

Add an additional layer of filtering at the Sentry configuration level as a belt-and-suspenders approach. This ensures even if the global error handler changes in the future, Sentry won't receive these non-errors.

### Changes Required:

#### 1. Sentry Server Configuration

**File**: `sentry.server.config.ts`
**Changes**: Add beforeSend hook to filter NEXT_REDIRECT

```typescript
Sentry.init({
  dsn: process.env.NEXT_PUBLIC_SENTRY_DSN,
  enabled: process.env.NEXT_PUBLIC_ENV === "production",
  tracesSampleRate: 1,
  debug: false,
  beforeSend(event, hint) {
    // Filter out Next.js internal control flow errors
    const error = hint.originalException;
    if (error && typeof error === "object" && "digest" in error) {
      const digest = (error as any).digest;
      if (
        typeof digest === "string" &&
        (digest.startsWith("NEXT_REDIRECT") || digest.startsWith("NEXT_NOT_FOUND"))
      ) {
        return null; // Don't send to Sentry
      }
    }
    return event;
  },
});
```

#### 2. Sentry Client Configuration

**File**: `sentry.client.config.ts`
**Changes**: Add same beforeSend hook (after line 24)

```typescript
beforeSend(event, hint) {
  // Filter out Next.js internal control flow errors
  const error = hint.originalException;
  if (error && typeof error === 'object' && 'digest' in error) {
    const digest = (error as any).digest;
    if (typeof digest === 'string' &&
        (digest.startsWith('NEXT_REDIRECT') || digest.startsWith('NEXT_NOT_FOUND'))) {
      return null; // Don't send to Sentry
    }
  }
  return event;
},
```

#### 3. Sentry Edge Configuration

**File**: `sentry.edge.config.ts`
**Changes**: Add same beforeSend hook (after line 16)

```typescript
beforeSend(event, hint) {
  // Filter out Next.js internal control flow errors
  const error = hint.originalException;
  if (error && typeof error === 'object' && 'digest' in error) {
    const digest = (error as any).digest;
    if (typeof digest === 'string' &&
        (digest.startsWith('NEXT_REDIRECT') || digest.startsWith('NEXT_NOT_FOUND'))) {
      return null; // Don't send to Sentry
    }
  }
  return event;
},
```

### Success Criteria:

#### Automated Verification:

- [ ] Type checking passes: `pnpm typecheck`
- [ ] Build completes successfully: `pnpm build`

#### Manual Verification:

- [ ] Deploy to staging environment
- [ ] Trigger a test error to confirm Sentry still works
- [ ] Use invitation links to confirm NEXT_REDIRECT not sent to Sentry
- [ ] Verify in Sentry dashboard that filtering is working

---

## Testing Strategy

### Manual Testing Steps:

1. Create a test workspace invitation link
2. Access the link in an incognito browser window
3. Verify redirect to projects page works without error
4. Check browser console for any error messages
5. Create an invalid invitation link
6. Verify proper error UI is shown for invalid link
7. Check Sentry and Axiom dashboards for error presence

### Edge Cases:

- Test with expired invitation codes (should show error)
- Test with already-used invitation codes (should show error)
- Test while logged out (handled by middleware)
- Test with malformed invitation codes (should show error)

## Performance Considerations

This change has minimal performance impact as it's just adding a string comparison check before logging. The early return actually improves performance slightly by avoiding unnecessary Sentry and Axiom API calls.

## Migration Notes

No migration needed. This fix can be deployed immediately and will start filtering errors as soon as it's live.

## References

- Original ticket: ENG-1082
- Related research: `thoughts/shared/research/2025-10-15-ENG-1082-join-workspace-authentication-context.md`
- Similar issue research: `thoughts/shared/research/2025-10-15-ENG-1082-view-only-project-sharing-redirect-error.md`
- Next.js documentation on redirect: https://nextjs.org/docs/app/building-your-application/routing/redirecting
