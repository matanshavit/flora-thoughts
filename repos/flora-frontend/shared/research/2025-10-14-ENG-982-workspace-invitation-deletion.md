---
date: 2025-10-14T14:59:10-04:00
researcher: Claude Code
git_commit: a1182efa38a0521b3cd0cf81a058dec46fae179e
branch: eng-982-remove-from-workspace
repository: eng-982-remove-from-workspace
topic: "ENG-982: Delete outstanding invitations when removing user from workspace"
tags: [research, codebase, workspace-invitations, user-management, permissions]
status: complete
last_updated: 2025-10-14
last_updated_by: Claude Code
---

# Research: ENG-982 - Delete Outstanding Invitations When Removing User from Workspace

**Date**: 2025-10-14T14:59:10-04:00
**Researcher**: Claude Code
**Git Commit**: a1182efa38a0521b3cd0cf81a058dec46fae179e
**Branch**: eng-982-remove-from-workspace
**Repository**: eng-982-remove-from-workspace

## Research Question
In convex/currentUser/mutations.ts, removeUserFromWorkspace was modified to ensure permissions are revoked to a workspace when removing a user. Now in addition, we would like to delete any outstanding invitation to that user for the workspace. Find all relevant files and code and compile a summary to use in planning.

## Summary
The `removeUserFromWorkspace` function in `convex/currentUser/mutations.ts` currently removes a user's membership from a workspace and handles workspace switching, but does not delete any outstanding invitations for that user. The workspace invitation system stores invitations in a `workspaceInvitations` table with email-based and link-based invitations. To complete ENG-982, we need to query and delete any email-specific invitations for the removed user after deleting their membership.

## Detailed Findings

### Current removeUserFromWorkspace Implementation

**File**: `convex/currentUser/mutations.ts:58-140`

The current implementation:
1. Validates admin permissions (lines 65-67)
2. Gets and validates the membership to remove (lines 70-78)
3. **Deletes the membership** (line 81)
4. Updates the removed user's `currentWorkspaceId` if needed (lines 84-136)
   - Switches to another workspace if available
   - Creates a recovery workspace if no other workspaces exist

**Missing functionality**: Does not delete outstanding invitations for the removed user.

### Workspace Invitation System Architecture

#### Database Schema

**PostgreSQL Schema** (`src/db/schema.ts:69-77`):
```typescript
export const workspaceInvitations = pgTable("flora_workspace_invitations", {
  id: serial("id").primaryKey().notNull(),
  workspaceId: uuid("workspace_id").notNull(),
  email: text("email"), // null = first come, first served link invitation
  role: text("role").notNull(),
  code: uuid("code").notNull().defaultRandom(),
  createdAt: timestamp("created_at", { mode: "date" }).default(sql`CURRENT_TIMESTAMP`),
  expiresAt: timestamp("expires_at", { mode: "date" }).notNull(),
});
```

**Convex Schema** (`convex/schema.ts:64-67`):
```typescript
workspaceInvitations: defineTable(workspaceInvitationsValidator)
  .index("by_pg_id", ["pg_id"])
  .index("by_email", ["email"])
  .index("by_workspace_id", ["workspaceId"]),
```

**Convex Validator** (`convex/modelValidators.ts:29-37`):
```typescript
export const workspaceInvitationsValidator = v.object({
  pg_id: v.optional(v.number()),
  workspaceId: v.id("workspaces"),
  email: v.optional(v.string()), // undefined = link invitation
  role: v.string(),
  code: v.string(), // UUID
  createdAt: v.optional(v.number()),
  expiresAt: v.number(),
});
```

#### Invitation Types

1. **Email Invitations**:
   - Have `email` field set to specific user email
   - Single-use (deleted after acceptance at `convex/currentUser/mutations.ts:309-311`)
   - Created via `inviteEmailToWorkspace` mutation

2. **Link Invitations**:
   - Have `email` field as null/undefined
   - Reusable (never automatically deleted)
   - First-come, first-served
   - Created via `createWorkspaceInviteLink` mutation

### Key Query Patterns

#### Query Invitations by Email
**Pattern used in** `convex/currentUser/queries.ts:157-161`:
```typescript
const invitations = await ctx.db
  .query("workspaceInvitations")
  .filter((q) => q.eq(q.field("email"), user.email))
  .collect();
```

#### Query by Email and Workspace
**Pattern used in** `convex/workspaceMemberships/helpers.ts:224-236`:
```typescript
const invitation = await ctx.db
  .query("workspaceInvitations")
  .withIndex("by_email", (q) => q.eq("email", email))
  .filter((q) => q.eq(q.field("workspaceId"), workspaceId))
  .first();
```

### Invitation Creation Flow

#### Email Invitation Creation
**File**: `convex/workspaceMemberships/mutations.ts:89-193`
- Validates permissions and plan
- Checks if user is already a member
- Creates or regenerates invitation with email field set
- Sends email via Resend

#### Helper Functions
**File**: `convex/workspaceMemberships/helpers.ts`

**createWorkspaceInvitation** (lines 241-260):
```typescript
const invitationId = await ctx.db.insert("workspaceInvitations", {
  workspaceId,
  email,
  role,
  code: crypto.randomUUID(),
  createdAt: Date.now(),
  expiresAt: Date.now() + 7 * 24 * 60 * 60 * 1000, // 7 days
});
```

**regenerateWorkspaceInvitationCode** (lines 265-278):
- Updates existing invitation with new code and expiration

### Invitation Acceptance and Deletion

**File**: `convex/currentUser/mutations.ts:254-331` (`acceptInvitation`)

Key behavior at lines 309-311:
```typescript
// Link invitations are reusable, but email invitations should be deleted
if (invitation.email) {
  await ctx.db.delete(invitation._id);
}
```

This shows that:
- Email invitations are deleted after successful acceptance
- Link invitations (email=null) are kept for reuse
- No other automatic cleanup exists

### No Automated Cleanup

**Important finding**: After searching all cron jobs in `src/app/api/cron/`, there is no automated cleanup process for expired invitations. Expired invitations remain in the database indefinitely.

## Code References

### Core Files
- `convex/currentUser/mutations.ts:58-140` - removeUserFromWorkspace function to modify
- `convex/currentUser/mutations.ts:254-331` - acceptInvitation shows deletion pattern
- `convex/workspaceMemberships/mutations.ts:89-193` - Email invitation creation
- `convex/workspaceMemberships/helpers.ts:224-278` - Invitation helper functions

### Schema Files
- `src/db/schema.ts:69-77` - PostgreSQL schema definition
- `convex/schema.ts:64-67` - Convex schema with indexes
- `convex/modelValidators.ts:29-37` - Convex validator

### Query Patterns
- `convex/currentUser/queries.ts:157-161` - Query by email
- `convex/workspaceMemberships/helpers.ts:224-236` - Query by email and workspace
- `convex/admin/workspaces/queries.ts:178-182` - Query by workspace

## Architecture Documentation

### Invitation Lifecycle
1. **Creation**: UUID code generated, 7-day expiration set
2. **Storage**: Saved with email (specific) or without (link)
3. **Acceptance**: Validated for expiration and email match
4. **Deletion**:
   - Email invitations: Deleted after use
   - Link invitations: Never deleted
   - **Gap**: No deletion when user is removed from workspace

### Database Indexes
- `by_email`: Efficient lookup by user email
- `by_workspace_id`: List all invitations for a workspace
- `by_pg_id`: PostgreSQL migration support

### Permission Model
- Only admins can remove users from workspaces
- Admins and editors can invite users (guests cannot)
- Email invitations require pro/agency plan

## Implementation Requirements

To complete ENG-982, we need to:

1. **Add invitation deletion logic** to `removeUserFromWorkspace` after line 81
2. **Query pattern** to use:
   ```typescript
   // Get the removed user's email
   const removedUser = await ctx.db.get(removedUserId);
   if (removedUser && removedUser.email) {
     // Find all email invitations for this user to this workspace
     const invitations = await ctx.db
       .query("workspaceInvitations")
       .withIndex("by_email", (q) => q.eq("email", removedUser.email))
       .filter((q) => q.eq(q.field("workspaceId"), membershipToRemove.workspaceId))
       .collect();

     // Delete all matching invitations
     for (const invitation of invitations) {
       await ctx.db.delete(invitation._id);
     }
   }
   ```

3. **Important considerations**:
   - Only delete email-specific invitations (where email is set)
   - Link invitations should remain active
   - Use existing indexes for efficient queries
   - Null-check user and email before querying

## Historical Context

No existing research documents or thoughts were found specifically about invitation cleanup or this feature. This appears to be a new enhancement to close a gap in the user removal process.

## Related Research

- Workspace membership management patterns
- User permission system
- Invitation system architecture

## Open Questions

None - all required information for implementation has been identified.