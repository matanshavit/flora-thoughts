---
date: 2025-10-15T11:25:57-04:00
researcher: matanshavit
git_commit: 0a212379dd23cade8bb7211802c1b3c8cf3298c4
branch: eng-1063
repository: eng-1063
topic: "ENG-1063: Convex user corrections - clerkId constraint and sanitizedEmail updates"
tags: [research, codebase, convex, users, database, email-sanitization, clerkid]
status: complete
last_updated: 2025-10-15
last_updated_by: matanshavit
---

# Research: ENG-1063 - Convex User Corrections

**Date**: 2025-10-15 11:25:57 EDT
**Researcher**: matanshavit
**Git Commit**: 0a212379dd23cade8bb7211802c1b3c8cf3298c4
**Branch**: eng-1063
**Repository**: eng-1063

## Research Question
Research the Convex database for user corrections needed on the r3f canvas migration branch. Specifically:
1. Add a unique constraint to the clerkId column
2. Make it so updating email will also update sanitizedEmail, which will be sanitized like on user creation

This is Convex-specific and does not affect PostgreSQL. Since this is on the r3f canvas migration branch, ReactFlow canvas functionality is not a concern.

## Summary
The Convex user system currently has two key issues that need correction:

1. **Missing clerkId Index**: The `clerkId` field has no index or constraint, meaning queries filtering by clerkId require full table scans. All user lookups currently use the `by_sanitized_email` index.

2. **Email Update Inconsistency**: When users update their email address, only the `email` field is updated while `sanitizedEmail` remains unchanged. This creates a data inconsistency where the sanitized lookup key doesn't match the current email.

## Detailed Findings

### User Schema Structure

The user schema is defined in `convex/modelValidators.ts:158-190` with these key fields:
- `clerkId: v.string()` - Required field for Clerk authentication
- `email: v.string()` - Required field storing the user's email
- `sanitizedEmail: v.string()` - Required field storing normalized email for lookups

The database indexes are defined in `convex/schema.ts:110-121`:
```typescript
users: defineTable(usersValidator)
  .index("by_sanitized_email", ["sanitizedEmail"])
  .index("by_pg_id", ["pg_id"])
  .searchIndex("search_firstName", { searchField: "firstName" })
  .searchIndex("search_lastName", { searchField: "lastName" })
  .searchIndex("search_email", { searchField: "email" })
```

**Issue 1**: There is no index on the `clerkId` field, despite it being a required identifier from the authentication system.

### Email Sanitization Implementation

Email sanitization is handled by `shared/utils/email.ts:5-9`:
```typescript
export function sanitizeEmail(email: string): string {
  const [localPart, domain] = email.split("@");
  const sanitizedLocalPart = localPart.replace(/\./g, "");
  return `${sanitizedLocalPart}@${domain}`;
}
```

This removes dots from the email's local part (e.g., `john.doe@gmail.com` → `johndoe@gmail.com`) to handle Gmail's dot-insensitive addresses.

### User Creation Flow

User creation properly sets both fields (`convex/users/helpers.ts:59-168`):

1. Email is sanitized: `const sanitizedEmail = sanitizeEmail(email)` (line 68)
2. Duplicate check uses sanitized email index (lines 71-73)
3. User is created with both fields:
   - `email` - Original email (line 87)
   - `sanitizedEmail` - Normalized version (line 90)

All user lookups throughout the codebase use the `by_sanitized_email` index consistently:
- `convex/users/queries.ts:21` - Current user query
- `convex/users/helpers.ts:22` - Get current user helper
- `convex/serverApi/users/queries.ts:19` - Server API query
- `convex/workspaceMemberships/helpers.ts:204` - Workspace membership checks
- `convex/projects/sharing.ts:327` - Project sharing logic

### Email Update Problem

**Issue 2**: The email update flow doesn't update `sanitizedEmail` (`convex/currentUser/mutations.ts:34-56`):

The `patch` mutation accepts an optional `email` field (line 15) but directly patches whatever fields are provided without processing:
```typescript
await ctx.db.patch(currentUser._id, updateFields);  // line 51
```

The frontend email update flow (`src/components/profile/edit-account-form.tsx:177-196`):
1. Updates email in Clerk authentication system
2. Calls `updateUser({ email: newEmail })`
3. This calls the `patch` mutation which updates only the `email` field
4. `sanitizedEmail` remains unchanged with the old email's sanitized value

This creates inconsistency where:
- `email` contains the new email address
- `sanitizedEmail` still contains the sanitized version of the OLD email
- User lookups still work but use outdated sanitized email

### ClerkId Handling

The `clerkId` is properly set during user creation:

**Standard creation** (`convex/users/mutations.ts:23`):
- Extracted from Clerk identity
- Passed to `_createUser` helper
- Stored in database

**Migration creation** (`convex/migration/users.ts:83,141,174`):
- Included in migration data from PostgreSQL
- Set during insert or patch operations

However, there's no way to efficiently query users by `clerkId` due to the missing index.

### Migration Considerations

The migration process (`convex/migration/users.ts:80-211`) properly handles both fields:
- Accepts `sanitizedEmail` as input (already sanitized from PostgreSQL)
- Checks for existing users by `pg_id` and `sanitizedEmail` indexes
- Updates or inserts with proper field values

## Code References

- `convex/modelValidators.ts:158-190` - User schema definition
- `convex/schema.ts:110-121` - Database indexes (missing clerkId index)
- `shared/utils/email.ts:5-9` - Email sanitization utility
- `convex/users/helpers.ts:68,90` - User creation with sanitization
- `convex/currentUser/mutations.ts:51` - Email update without sanitization
- `src/components/profile/edit-account-form.tsx:185` - Frontend email update trigger

## Architecture Documentation

The Convex user system follows these patterns:

1. **Email Normalization**: All user lookups use sanitized emails to handle email variations (Gmail dot notation)
2. **Index Strategy**: Primary user lookups use `by_sanitized_email` index exclusively
3. **Authentication Integration**: Clerk provides `clerkId` but it's not indexed for queries
4. **Mutation Pattern**: The `patch` mutation is generic and doesn't perform field-specific processing

## Required Changes

Based on the findings, two changes are needed:

1. **Add clerkId Index**: Add `.index("by_clerk_id", ["clerkId"])` to the users table schema
2. **Fix Email Updates**: Modify the `patch` mutation to sanitize email when updating:
   - Check if `email` field is being updated
   - If yes, also compute and set `sanitizedEmail = sanitizeEmail(email)`

## Related Research

This research specifically targets Convex database corrections and does not impact:
- PostgreSQL database structure
- ReactFlow canvas functionality (since we're on r3f migration branch)
- User authentication flow (Clerk integration remains unchanged)

## Open Questions

1. Should we add a unique constraint or just an index on `clerkId`?
2. Should historical data be migrated to fix existing `sanitizedEmail` mismatches?
3. Are there other mutations that might update email without sanitization?