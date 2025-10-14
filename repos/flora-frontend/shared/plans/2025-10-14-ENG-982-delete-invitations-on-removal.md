# Delete Outstanding Invitations When Removing User from Workspace Implementation Plan

## Overview

When removing a user from a workspace, the system currently deletes their membership and handles workspace switching, but does not delete any outstanding email invitations for that user. This leaves orphaned invitations that could allow the removed user to rejoin the workspace.

## Current State Analysis

The `removeUserFromWorkspace` mutation in `convex/currentUser/mutations.ts:58-140` currently:
1. Validates admin permissions
2. Deletes the workspace membership (line 81)
3. Updates the removed user's `currentWorkspaceId` if needed
4. Creates a recovery workspace if no other workspaces exist

**Missing**: Deletion of outstanding email invitations for the removed user to that workspace.

### Key Discoveries:
- Email invitations are stored with the user's email address and are single-use
- Link invitations (email=null) are reusable and should NOT be deleted
- The pattern for deleting email invitations already exists at `convex/currentUser/mutations.ts:309-311`
- Appropriate indexes exist for efficient querying: `by_email` and filtering by `workspaceId`

## Desired End State

After implementation, when an admin removes a user from a workspace:
1. The user's membership is deleted (existing behavior)
2. All email-specific invitations for that user to that workspace are deleted (new behavior)
3. Link invitations remain untouched (they are not user-specific)
4. The removed user cannot rejoin using an old invitation link sent to their email

### Verification:
- Removing a user deletes their outstanding email invitations
- Link invitations continue to work for any user
- No orphaned invitations remain in the database after user removal

## What We're NOT Doing

- NOT deleting link invitations (they are reusable and not user-specific)
- NOT modifying the invitation acceptance flow
- NOT adding any new database indexes
- NOT changing the invitation creation process
- NOT implementing automated cleanup of expired invitations (separate concern)
- NOT modifying project invitations (only workspace invitations)

## Implementation Approach

Add invitation deletion logic immediately after deleting the membership (after line 81) in the `removeUserFromWorkspace` mutation. Use the existing query patterns with proper null-checking and only delete email-specific invitations.

## Phase 1: Implement Invitation Deletion Logic

### Overview
Add the logic to delete outstanding email invitations when removing a user from a workspace.

### Changes Required:

#### 1. Update removeUserFromWorkspace Mutation
**File**: `convex/currentUser/mutations.ts`
**Changes**: Add invitation deletion logic after line 81

```typescript
// After line 81: await ctx.db.delete(membershipToRemove._id);

// Delete any outstanding email invitations for this user to this workspace
// Get the removed user's details (we already fetch this later, so move it up)
const removedUserId = membershipToRemove.userId;
const removedUser = await ctx.db.get(removedUserId);

if (removedUser && removedUser.email) {
  // Find all email invitations for this user to this workspace
  const invitations = await ctx.db
    .query("workspaceInvitations")
    .withIndex("by_email", (q) => q.eq("email", removedUser.email))
    .filter((q) => q.eq(q.field("workspaceId"), membershipToRemove.workspaceId))
    .collect();

  // Delete all matching email invitations
  for (const invitation of invitations) {
    await ctx.db.delete(invitation._id);
  }
}

// Continue with existing logic to update removed user's workspace...
// Note: Remove the duplicate lines 84-85 since we moved them up
```

The complete updated function will be:

```typescript
export const removeUserFromWorkspace = authenticatedMutation({
  args: { membershipId: v.id("workspaceMemberships") },
  returns: v.null(),
  handler: async (ctx, args) => {
    const currentUserRole = await getCurrentUserRole(ctx);

    // Only admins can remove users
    if (currentUserRole.role !== UserRoleEnum.ADMIN) {
      throw new Error("Only admins can remove users from workspace");
    }

    // Get the membership to remove
    const membershipToRemove = await ctx.db.get(args.membershipId);
    if (!membershipToRemove) {
      throw new Error("User not found in workspace");
    }

    // Prevent removing self
    if (membershipToRemove.userId === ctx.user._id) {
      throw new Error("Cannot remove yourself from workspace");
    }

    // Delete the membership
    await ctx.db.delete(membershipToRemove._id);

    // Get the removed user's details
    const removedUserId = membershipToRemove.userId;
    const removedUser = await ctx.db.get(removedUserId);

    if (!removedUser) {
      return null;
    }

    // Delete any outstanding email invitations for this user to this workspace
    if (removedUser.email) {
      // Find all email invitations for this user to this workspace
      const invitations = await ctx.db
        .query("workspaceInvitations")
        .withIndex("by_email", (q) => q.eq("email", removedUser.email))
        .filter((q) => q.eq(q.field("workspaceId"), membershipToRemove.workspaceId))
        .collect();

      // Delete all matching email invitations
      for (const invitation of invitations) {
        await ctx.db.delete(invitation._id);
      }
    }

    // Only update if user's current workspace is the one they're being removed from
    if (removedUser.currentWorkspaceId === membershipToRemove.workspaceId) {
      // ... existing workspace switching logic remains unchanged
    }

    return null;
  },
});
```

### Success Criteria:

#### Automated Verification:
- [ ] Code compiles without errors: `pnpm typecheck`
- [ ] Convex deployment succeeds: `pnpm convex deploy`
- [ ] No linting errors: `pnpm lint`

#### Manual Verification:
- [ ] Create an email invitation for a user to a workspace
- [ ] Remove that user from the workspace via the admin UI
- [ ] Verify the invitation is deleted from the database
- [ ] Verify the removed user cannot use the old invitation link
- [ ] Verify link invitations (without email) remain active
- [ ] Verify workspace switching works correctly for the removed user

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation from the human that the manual testing was successful before proceeding to the next phase.

---

## Phase 2: Add Integration Tests

### Overview
Add tests to verify the invitation deletion behavior works correctly.

### Changes Required:

#### 1. Create Test for Invitation Deletion
**File**: `convex/currentUser/mutations.test.ts` (create if doesn't exist)
**Changes**: Add test cases for the new behavior

```typescript
import { expect, test } from "vitest";
import { convexTest } from "convex-test";
import schema from "../schema";

test("removeUserFromWorkspace deletes email invitations", async () => {
  const t = convexTest(schema);

  // Setup test data
  const adminId = await t.run(async (ctx) => {
    return await ctx.db.insert("users", {
      email: "admin@test.com",
      // ... other required fields
    });
  });

  const userId = await t.run(async (ctx) => {
    return await ctx.db.insert("users", {
      email: "user@test.com",
      // ... other required fields
    });
  });

  const workspaceId = await t.run(async (ctx) => {
    return await ctx.db.insert("workspaces", {
      name: "Test Workspace",
      ownerId: adminId,
      // ... other required fields
    });
  });

  // Create membership for user
  const membershipId = await t.run(async (ctx) => {
    return await ctx.db.insert("workspaceMemberships", {
      userId,
      workspaceId,
      role: "editor",
      active: true,
    });
  });

  // Create email invitation
  const invitationId = await t.run(async (ctx) => {
    return await ctx.db.insert("workspaceInvitations", {
      workspaceId,
      email: "user@test.com",
      role: "editor",
      code: "test-code",
      expiresAt: Date.now() + 86400000,
    });
  });

  // Remove user from workspace
  await t.mutation(api.currentUser.mutations.removeUserFromWorkspace, {
    membershipId,
  });

  // Verify invitation was deleted
  const invitation = await t.run(async (ctx) => {
    return await ctx.db.get(invitationId);
  });

  expect(invitation).toBeNull();
});

test("removeUserFromWorkspace preserves link invitations", async () => {
  // Similar setup...

  // Create link invitation (no email)
  const linkInvitationId = await t.run(async (ctx) => {
    return await ctx.db.insert("workspaceInvitations", {
      workspaceId,
      email: undefined, // Link invitation
      role: "editor",
      code: "link-code",
      expiresAt: Date.now() + 86400000,
    });
  });

  // Remove user from workspace
  await t.mutation(api.currentUser.mutations.removeUserFromWorkspace, {
    membershipId,
  });

  // Verify link invitation was NOT deleted
  const linkInvitation = await t.run(async (ctx) => {
    return await ctx.db.get(linkInvitationId);
  });

  expect(linkInvitation).not.toBeNull();
});
```

### Success Criteria:

#### Automated Verification:
- [ ] Tests pass: `pnpm test:unit`
- [ ] Test coverage includes both email and link invitation scenarios

#### Manual Verification:
- [ ] Tests accurately reflect the production behavior
- [ ] Edge cases are covered (user with no email, multiple invitations, etc.)

---

## Testing Strategy

### Unit Tests:
- Test that email invitations are deleted when user is removed
- Test that link invitations are NOT deleted when user is removed
- Test that invitations for other workspaces are not affected
- Test null safety (user without email, no invitations exist)

### Integration Tests:
- Full flow: invite user → remove user → verify invitation deleted
- Verify removed user cannot accept deleted invitation
- Verify link invitations continue to work after a different user is removed

### Manual Testing Steps:
1. As admin, invite user@example.com to workspace via email
2. Before user accepts, remove them from workspace member list
3. Check database to confirm invitation was deleted
4. Have user try to accept invitation link - should fail
5. Create a link invitation and verify it remains after removing any user
6. Test with multiple invitations for same user/workspace pair

## Performance Considerations

- Using indexed query (`by_email`) ensures efficient lookup
- Filtering by `workspaceId` is performed in memory but on a small result set
- Deletion loop is minimal as email invitations are single-use (typically 0-1 per user/workspace)
- No performance regression expected

## Migration Notes

No data migration required. The change is backwards compatible:
- Existing invitations remain unchanged
- New behavior only applies to future removals
- No schema changes needed

## References

- Original research: `thoughts/shared/research/2025-10-14-ENG-982-workspace-invitation-deletion.md`
- Invitation deletion pattern: `convex/currentUser/mutations.ts:309-311`
- Query pattern: `convex/workspaceMemberships/helpers.ts:224-236`
- Similar project invitation pattern: `convex/projects/mutations.ts:592-594`