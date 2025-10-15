# Convex User Corrections Implementation Plan

## Overview

Fix two critical issues in the Convex user system to ensure data integrity and query performance: add a missing clerkId index for efficient authentication lookups and fix the email update flow to maintain consistency between email and sanitizedEmail fields.

## Current State Analysis

The Convex user system has two issues that need correction:

1. **Missing clerkId Index**: The `clerkId` field has no index despite being a required unique identifier from Clerk authentication. All user lookups currently use the `by_sanitized_email` index, resulting in inefficient queries when clerkId-based lookups would be more appropriate.

2. **Email Update Inconsistency**: When users update their email address through the profile settings, only the `email` field is updated while `sanitizedEmail` remains unchanged. This creates a data inconsistency where the sanitized lookup key doesn't match the current email.

### Key Discoveries:
- User lookups exclusively use `by_sanitized_email` index (`convex/users/helpers.ts:20-23`)
- Email sanitization removes dots from local part for Gmail compatibility (`shared/utils/email.ts:5-9`)
- The patch mutation directly updates fields without processing (`convex/currentUser/mutations.ts:51`)
- No existing queries use clerkId for lookups, so adding an index won't break anything

## Desired End State

After implementing this plan:
- Users table will have an efficient `by_clerk_id` index for future authentication optimizations
- Email updates will automatically update both `email` and `sanitizedEmail` fields
- Data consistency will be maintained between actual and sanitized email values
- All user lookups will continue to work without disruption

### Verification:
- New index appears in Convex dashboard
- Email updates correctly update both fields
- No duplicate clerkId values exist in the database
- User authentication and lookups continue to work

## What We're NOT Doing

- NOT changing how user lookups work (still using sanitizedEmail)
- NOT migrating historical data with mismatched sanitizedEmail values
- NOT adding queries that use the clerkId index (future optimization)
- NOT modifying PostgreSQL database structure
- NOT changing the email sanitization logic

## Implementation Approach

Two independent changes that can be deployed sequentially:
1. Add the clerkId index to the schema (non-breaking addition)
2. Modify the patch mutation to handle email updates correctly (backward compatible)

## Phase 1: Add clerkId Index

### Overview
Add a new index on the `clerkId` field to enable efficient lookups by Clerk authentication ID.

### Changes Required:

#### 1. Update Users Table Schema
**File**: `convex/schema.ts`
**Changes**: Add clerkId index to users table definition

```typescript
users: defineTable(usersValidator)
  .index("by_sanitized_email", ["sanitizedEmail"])
  .index("by_clerk_id", ["clerkId"])  // Add this line
  .index("by_pg_id", ["pg_id"])
  .searchIndex("search_firstName", {
    searchField: "firstName",
  })
  .searchIndex("search_lastName", {
    searchField: "lastName",
  })
  .searchIndex("search_email", {
    searchField: "email",
  }),
```

✓ Completed - Added clerkId index to users table

### Success Criteria:

#### Automated Verification:
- [x] Deploy succeeds: `pnpm convex deploy` ✓ Index added successfully
- [x] No TypeScript errors: `pnpm typecheck` ✓ No new errors introduced
- [x] No linting errors: `pnpm lint` ✓ No new errors introduced

#### Manual Verification:
- [ ] New `by_clerk_id` index appears in Convex dashboard
- [ ] Existing user authentication continues to work
- [ ] No errors in Convex function logs
- [ ] User login/logout flows work correctly

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation from the human that the index appears in Convex dashboard before proceeding to the next phase.

---

## Phase 2: Fix Email Update Sanitization

### Overview
Modify the user patch mutation to automatically update `sanitizedEmail` whenever the `email` field is updated.

### Changes Required:

#### 1. Import Email Sanitization Utility
**File**: `convex/currentUser/mutations.ts`
**Changes**: Add import for sanitizeEmail function

```typescript
import { v } from "convex/values";
import { sanitizeEmail } from "@shared/utils/email";  // Add this import
import { UserRoleEnum } from "../../shared/constants";
```

#### 2. Update Patch Mutation Handler
**File**: `convex/currentUser/mutations.ts`
**Changes**: Modify the patch mutation to handle email sanitization

```typescript
export const patch = authenticatedMutation({
  args: userPatchValidator,
  returns: v.null(),
  handler: async (ctx, args) => {
    const currentUser = await ctx.user;

    // Filter out undefined values to only update provided fields
    // eslint-disable-next-line @typescript-eslint/no-explicit-any -- Legacy Code Lint Violation
    const updateFields: Record<string, any> = {};
    for (const [key, value] of Object.entries(args)) {
      if (value !== undefined) {
        updateFields[key] = value;
      }
    }

    // If email is being updated, also update sanitizedEmail
    if (updateFields.email) {
      updateFields.sanitizedEmail = sanitizeEmail(updateFields.email);
    }

    // Only proceed if there are actual fields to update
    if (Object.keys(updateFields).length > 0) {
      await ctx.db.patch(currentUser._id, updateFields);
    }

    return null;
  },
});
```

### Success Criteria:

#### Automated Verification:
- [ ] Deploy succeeds: `pnpm convex deploy`
- [ ] TypeScript compilation passes: `pnpm typecheck`
- [ ] Linting passes: `pnpm lint`
- [ ] Build succeeds: `pnpm build`

#### Manual Verification:
- [ ] Email update flow works end-to-end
- [ ] Both `email` and `sanitizedEmail` fields are updated in database
- [ ] User can log in with new email after update
- [ ] Non-email profile updates continue to work
- [ ] Email with dots (e.g., john.doe@gmail.com) correctly sanitizes

**Implementation Note**: After completing this phase and all automated verification passes, test the complete email update flow manually to ensure both fields are properly synchronized.

---

## Testing Strategy

### Unit Tests:
- Test sanitizeEmail function with various email formats
- Test patch mutation with email updates
- Test patch mutation with non-email updates

### Integration Tests:
- Complete email update flow from UI to database
- User authentication after email change
- Profile updates without email changes

### Manual Testing Steps:
1. Update user email through profile settings
2. Verify both `email` and `sanitizedEmail` updated in Convex dashboard
3. Log out and log back in with new email
4. Test email with dots (Gmail edge case)
5. Update other profile fields to ensure no regression

## Performance Considerations

- Adding the clerkId index will slightly increase write time but significantly improve potential query performance
- Index creation may take time for large user tables
- Email sanitization adds minimal overhead to email updates only

## Migration Notes

- No data migration required for clerkId index (additive change)
- Existing users with mismatched email/sanitizedEmail will be fixed on next email update
- Consider future batch update to fix historical mismatches if needed

## References

- Original research: `thoughts/shared/research/2025-10-15-ENG-1063-convex-user-corrections.md`
- User schema: `convex/modelValidators.ts:158-190`
- Database indexes: `convex/schema.ts:110-121`
- Email sanitization: `shared/utils/email.ts:5-9`
- Patch mutation: `convex/currentUser/mutations.ts:34-56`