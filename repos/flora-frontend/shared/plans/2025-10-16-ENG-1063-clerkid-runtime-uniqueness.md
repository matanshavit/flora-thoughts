# ClerkId Runtime Uniqueness Enforcement Implementation Plan

## Overview

Implement runtime uniqueness validation for the `clerkId` field in Convex user creation to prevent duplicate Clerk authentication IDs from being inserted into the database.

## Current State Analysis

The Convex user system currently has a `clerkId` field with a non-unique index (`by_clerk_id`) added in a previous implementation. However, Convex does not support database-level unique constraints - uniqueness must be enforced at the application level through runtime validation in mutations.

Current situation:

- `clerkId` has an index at `convex/schema.ts:115` but it's not a unique constraint
- User creation in `convex/users/helpers.ts:58-93` sets `clerkId` without checking for duplicates
- The pattern for uniqueness checking already exists for `sanitizedEmail` at lines 70-78

### Key Discoveries:

- Convex requires application-level uniqueness enforcement in mutations
- The codebase uses a "check-then-insert" pattern for `sanitizedEmail` uniqueness
- `clerkId` is only set during user creation, never updated afterward
- Migration imports have separate logic and are excluded from this plan

## Desired End State

After implementing this plan:

- The `_createUser` helper will check for existing users with the same `clerkId` before insertion
- Duplicate `clerkId` attempts will throw a clear error message
- The system will prevent any new duplicate `clerkId` entries at runtime
- Existing duplicate detection patterns will be consistently applied

### Verification:

- Attempting to create a user with an existing `clerkId` will fail with an error
- Normal user creation with unique `clerkId` values continues to work
- Error messages clearly indicate the duplicate `clerkId` issue
- All existing user creation flows remain functional

## What We're NOT Doing

- NOT modifying migration import logic (`convex/bulkMigration/users.ts`)
- NOT adding database-level constraints (not supported by Convex)
- NOT changing the existing `clerkId` index
- NOT modifying how `clerkId` is obtained from Clerk authentication
- NOT updating any existing users or cleaning up potential duplicates
- NOT changing the email uniqueness validation logic

## Implementation Approach

Add runtime validation to the `_createUser` helper function following the existing pattern used for `sanitizedEmail` uniqueness. This centralizes the validation logic in one place since all user creation flows go through this helper.

## Phase 1: Add ClerkId Uniqueness Check

### Overview

Modify the `_createUser` helper function to check for existing users with the same `clerkId` before creating a new user, following the pattern already established for `sanitizedEmail`.

### Changes Required:

#### 1. Update \_createUser Helper Function

**File**: `convex/users/helpers.ts`
**Changes**: Add clerkId uniqueness check before user insertion

```typescript
export async function _createUser(
  ctx: MutationCtx,
  {
    clerkId,
    email,
    profileImg,
    username,
    firstName,
    lastName,
  }: {
    clerkId: string;
    email: string;
    profileImg?: string;
    username?: string;
    firstName?: string;
    lastName?: string;
  },
  enterpriseWorkspaceDomain?: string,
) {
  const sanitizedEmail = sanitizeEmail(email);

  // Check if user already exists with this email
  const existingUser = await ctx.db
    .query("users")
    .withIndex("by_sanitized_email", (q) => q.eq("sanitizedEmail", sanitizedEmail))
    .first();

  if (existingUser !== null) {
    console.log("User already exists for email ", sanitizedEmail);
    return existingUser._id;
  }

  // Check if user already exists with this clerkId
  const existingClerkUser = await ctx.db
    .query("users")
    .withIndex("by_clerk_id", (q) => q.eq("clerkId", clerkId))
    .first();

  if (existingClerkUser !== null) {
    throw new Error(`User already exists with clerkId: ${clerkId}`);
  }

  // Step 1: Create the user first
  console.log("Creating user in createUser func", { email: sanitizedEmail });
  const userId = await ctx.db.insert("users", {
    clerkId,
    firstName,
    lastName,
    email,
    username,
    profileImg,
    sanitizedEmail: sanitizedEmail,
    createdAt: Date.now(),
  });

  // ... rest of the function remains unchanged
```

### Key Implementation Details:

1. **Check Order**: Check `sanitizedEmail` first (maintains existing behavior), then check `clerkId`
2. **Error Handling**:
   - Email duplicates: Return existing user ID (idempotent, existing behavior)
   - ClerkId duplicates: Throw error (strict enforcement, prevents data corruption)
3. **Index Usage**: Utilize the existing `by_clerk_id` index for efficient lookups
4. **Error Message**: Include the duplicate `clerkId` in the error for debugging

### Success Criteria:

#### Automated Verification:

- [x] TypeScript compilation passes: `pnpm typecheck`
- [x] Linting passes: `pnpm lint`
- [x] Build succeeds: `pnpm build`
- [x] Convex deployment succeeds: `pnpm convex deploy`

#### Manual Verification:

- [ ] Normal user registration works with unique clerkId
- [ ] Attempting to create a user with duplicate clerkId throws an error
- [ ] Error message clearly indicates the duplicate clerkId
- [ ] Existing email duplicate handling still returns the existing user
- [ ] User creation through all entry points works correctly:
  - [ ] Standard signup flow
  - [ ] OAuth signup flow
  - [ ] API-based user creation

**Implementation Note**: After completing this phase and all automated verification passes, test the duplicate prevention by attempting to create multiple users with the same clerkId through different flows.

---

## Testing Strategy

### Unit Tests:

- Mock `ctx.db.query` to test duplicate clerkId detection
- Test that duplicate clerkId throws an error
- Test that unique clerkId allows user creation
- Test that email duplicate still returns existing user ID

### Integration Tests:

- Create user with unique clerkId - should succeed
- Attempt to create another user with same clerkId - should fail
- Verify error message contains the duplicate clerkId
- Ensure email uniqueness logic is not affected

### Manual Testing Steps:

1. Create a new user through the standard signup flow
2. Attempt to create another user with the same clerkId (may require API testing)
3. Verify error is thrown and logged
4. Test that normal user creation still works
5. Verify existing email duplicate handling still works

## Performance Considerations

- The additional query uses the existing `by_clerk_id` index, so lookup is efficient (O(log n))
- Adds one additional indexed query per user creation
- No performance impact on existing user operations
- Migration imports are unaffected (excluded from this implementation)

## Migration Notes

- No data migration required (validation only applies to new users)
- Existing duplicate clerkIds (if any) will remain but new duplicates will be prevented
- Migration imports through `convex/bulkMigration/users.ts` are explicitly excluded from this validation
- Future consideration: Add a separate cleanup task if duplicate clerkIds are discovered

## Error Handling

The implementation follows these principles:

1. **Email Duplicates** (existing behavior): Return existing user ID silently

   - Rationale: Idempotent operation, user might be re-authenticating

2. **ClerkId Duplicates** (new behavior): Throw an error
   - Rationale: Indicates a serious issue (Clerk should never provide duplicate IDs)
   - Error format: `"User already exists with clerkId: <clerkId>"`

## References

- Original ticket: `thoughts/shared/tickets/ENG-1063.md`
- Previous research: `thoughts/shared/research/2025-10-15-ENG-1063-convex-user-corrections.md`
- Previous implementation: `thoughts/shared/plans/2025-10-15-ENG-1063-convex-user-corrections.md`
- User creation helper: `convex/users/helpers.ts:58-93`
- Existing uniqueness pattern: `convex/users/helpers.ts:70-78`
- ClerkId index: `convex/schema.ts:115`
