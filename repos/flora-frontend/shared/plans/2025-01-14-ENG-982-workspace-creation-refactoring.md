# Workspace Creation Refactoring Implementation Plan

## Overview

Consolidate duplicate workspace creation logic from two locations into a shared TypeScript helper function to improve code maintainability and consistency.

## Current State Analysis

Currently, workspace creation with customer and membership setup is duplicated in:

- **User creation flow** (`convex/users/helpers.ts:127-152`): Creates workspace for new users with 500 default credits
- **Recovery flow** (`convex/currentUser/mutations.ts:118-147`): Creates recovery workspace with 0 credits when user is removed from last workspace

Both locations follow the same pattern but with slight variations:

1. Create workspace with owner
2. Create customer with credits
3. Create workspace membership as admin
4. Update user's current workspace

### Key Discoveries:

- The recovery flow uses direct `ctx.db.insert()` instead of the existing `createWorkspace()` helper (inconsistency)
- Credit allocation differs: 500 for new users, 0 for recovery scenarios
- Workspace naming differs slightly between the two flows
- The codebase already has atomic helper functions that can be composed: `createWorkspace`, `createCustomer`, `createWorkspaceMembership`

## Desired End State

A single, reusable helper function that encapsulates the complete workspace setup pattern, eliminating duplication and ensuring consistency across all workspace creation scenarios.

### Verification:

- Both user creation and recovery flows use the same shared helper
- No duplicate code for workspace-customer-membership creation
- Consistent use of helper functions (no direct inserts)
- Clear parameter-based control of credits and naming

## What We're NOT Doing

- Not refactoring other workspace creation locations (admin panel, migrations) as they have different requirements
- Not changing the business logic around credit allocation
- Not modifying the existing atomic helper functions
- Not changing the data model or schema

## Implementation Approach

Create a composite helper function in `convex/workspaces/helpers.ts` that orchestrates the existing atomic helpers, then update both locations to use this new helper.

## Phase 1: Create Shared Helper Function

### Overview

Create a new `createWorkspaceWithCustomerAndMembership` helper that encapsulates the complete workspace setup pattern.

### Changes Required:

#### 1. Add New Helper Function to workspaces/helpers.ts

**File**: `convex/workspaces/helpers.ts`
**Changes**: Add new composite helper function after the existing `createWorkspace` function

```typescript
export async function createWorkspaceWithCustomerAndMembership(
  ctx: MutationCtx,
  args: {
    name?: string;
    firstName?: string;
    lastName?: string;
    ownerId: Id<"users">;
    availableCredits: number;
    membershipRole: "admin" | "editor" | "guest";
  },
): Promise<{
  workspaceId: Id<"workspaces">;
  customerId: Id<"customers">;
  membershipId: Id<"workspaceMemberships">;
}> {
  // Create workspace
  const workspaceId = await createWorkspace(ctx, {
    name: args.name,
    firstName: args.firstName,
    lastName: args.lastName,
    ownerId: args.ownerId,
  });

  // Create customer with specified credits
  const customerId = await createCustomer(ctx, {
    workspaceId,
    availableCredits: args.availableCredits,
  });

  // Create workspace membership
  const membershipId = await createWorkspaceMembership(ctx, {
    userId: args.ownerId,
    workspaceId,
    role: args.membershipRole,
    active: true,
  });

  // Update user's current workspace
  await ctx.db.patch(args.ownerId, {
    currentWorkspaceId: workspaceId,
  });

  return {
    workspaceId,
    customerId,
    membershipId,
  };
}
```

#### 2. Add Required Imports

**File**: `convex/workspaces/helpers.ts`
**Changes**: Add imports at the top of the file

```typescript
import { createCustomer } from "../customers/helpers";
import { createWorkspaceMembership } from "../workspaceMemberships/helpers";
```

### Success Criteria:

#### Automated Verification:

- [x] Type checking passes: `pnpm typecheck`
- [ ] Code compiles without errors: `pnpm build` (skipped - database connection required)

#### Manual Verification:

- [x] New helper function is correctly typed
- [x] All parameters are properly defined

---

## Phase 2: Refactor User Creation Flow

### Overview

Update the user creation flow in `convex/users/helpers.ts` to use the new shared helper.

### Changes Required:

#### 1. Import the New Helper

**File**: `convex/users/helpers.ts`
**Changes**: Update imports to include the new helper

```typescript
import { createWorkspace, createWorkspaceWithCustomerAndMembership } from "../workspaces/helpers";
```

#### 2. Replace Duplicate Code with Helper Call

**File**: `convex/users/helpers.ts`
**Changes**: Replace lines 128-152 with the new helper

Replace this:

```typescript
// Create new workspace with the user as owner
targetWorkspaceId = await createWorkspace(ctx, {
  firstName: firstName,
  lastName: lastName,
  ownerId: userId,
});

await createCustomer(ctx, {
  workspaceId: targetWorkspaceId,
  availableCredits: 500, // Default free credits
});

// Update user with workspace reference
await ctx.db.patch(userId, {
  currentWorkspaceId: targetWorkspaceId,
});

// Create workspace membership using helper
const membershipId = await createWorkspaceMembership(ctx, {
  userId: userId,
  workspaceId: targetWorkspaceId,
  role: joinRole,
  active: true,
});
```

With this:

```typescript
// Create new workspace with customer and membership
const { workspaceId, membershipId } = await createWorkspaceWithCustomerAndMembership(ctx, {
  firstName: firstName,
  lastName: lastName,
  ownerId: userId,
  availableCredits: 500, // Default free credits
  membershipRole: joinRole,
});
targetWorkspaceId = workspaceId;
```

#### 3. Remove Unused Imports

**File**: `convex/users/helpers.ts`
**Changes**: Remove imports that are no longer directly used

Remove:

```typescript
import { createCustomer } from "../customers/helpers";
import { createWorkspaceMembership } from "../workspaceMemberships/helpers";
```

### Success Criteria:

#### Automated Verification:

- [x] Type checking passes: `pnpm typecheck`
- [x] Unit tests pass: `pnpm test:unit`
- [ ] Build succeeds: `pnpm build` (skipped - database connection required)

#### Manual Verification:

- [ ] New user signup still creates workspace correctly
- [ ] User receives 500 default credits
- [ ] User is set as admin of new workspace
- [ ] User's currentWorkspaceId is updated

---

## Phase 3: Refactor Recovery Flow

### Overview

Update the recovery workspace creation in `convex/currentUser/mutations.ts` to use the new shared helper and fix the inconsistency with direct database insert.

### Changes Required:

#### 1. Import the New Helper

**File**: `convex/currentUser/mutations.ts`
**Changes**: Add import for the new helper

```typescript
import { createWorkspaceWithCustomerAndMembership } from "../workspaces/helpers";
```

#### 2. Replace Duplicate Code with Helper Call

**File**: `convex/currentUser/mutations.ts`
**Changes**: Replace lines 119-146 with the new helper

Replace this:

```typescript
// Edge case: User has no workspaces (shouldn't happen in normal flow)
// Create a recovery workspace using existing helper
const workspaceName = `${removedUser.firstName || removedUser.email.split("@")[0]}'s Workspace`;

const workspaceId = await ctx.db.insert("workspaces", {
  name: workspaceName,
  ownerId: removedUser._id,
  editorSeats: 1,
});

// Create customer with 0 credits (not 500 - this is a recovery scenario)
await createCustomer(ctx, {
  workspaceId,
  availableCredits: 0,
});

// Create membership
await createWorkspaceMembership(ctx, {
  userId: removedUser._id,
  workspaceId,
  role: UserRoleEnum.ADMIN,
  active: true,
});

// Update user's workspace
await ctx.db.patch(removedUserId, {
  currentWorkspaceId: workspaceId,
});
```

With this:

```typescript
// Edge case: User has no workspaces (shouldn't happen in normal flow)
// Create a recovery workspace
const workspaceName = `${removedUser.firstName || removedUser.email.split("@")[0]}'s Workspace`;

const { workspaceId } = await createWorkspaceWithCustomerAndMembership(ctx, {
  name: workspaceName,
  ownerId: removedUser._id,
  availableCredits: 0, // Recovery scenario, not initial signup
  membershipRole: UserRoleEnum.ADMIN,
});
```

#### 3. Remove Unused Imports

**File**: `convex/currentUser/mutations.ts`
**Changes**: Remove imports that are no longer directly used

Remove:

```typescript
import { createCustomer } from "../customers/helpers";
import { createWorkspaceMembership } from "../workspaceMemberships/helpers";
```

### Success Criteria:

#### Automated Verification:

- [x] Type checking passes: `pnpm typecheck`
- [x] Unit tests pass: `pnpm test:unit`
- [ ] Build succeeds: `pnpm build` (skipped - database connection required)

#### Manual Verification:

- [ ] User removal from workspace still works correctly
- [ ] Recovery workspace is created when user has no remaining workspaces
- [ ] Recovery workspace has 0 credits (not 500)
- [ ] User is switched to recovery workspace
- [ ] Email invitations are still deleted properly

---

## Phase 4: Final Verification and Cleanup

### Overview

Ensure both flows work correctly with the new shared helper and verify consistency.

### Changes Required:

None - this phase is for verification only.

### Success Criteria:

#### Automated Verification:

- [x] Full type checking passes: `pnpm typecheck`
- [x] All unit tests pass: `pnpm test:unit`
- [ ] Build completes without errors: `pnpm build` (skipped - database connection required)
- [x] Linting passes: `pnpm lint`

#### Manual Verification:

- [ ] Test new user signup flow:
  - Create a new user account
  - Verify workspace is created with 500 credits
  - Verify user is admin of the workspace
- [ ] Test workspace removal recovery flow:
  - Remove a user from their only workspace
  - Verify recovery workspace is created with 0 credits
  - Verify user is switched to recovery workspace
- [ ] Test existing user creating additional workspace (should still work unchanged)
- [ ] Verify no regressions in related features

## Testing Strategy

### Unit Tests:

- Test the new `createWorkspaceWithCustomerAndMembership` helper with various inputs
- Verify credit allocation logic (500 vs 0)
- Test workspace naming variations
- Verify all three entities are created correctly

### Integration Tests:

- Test complete user signup flow
- Test user removal from workspace with recovery
- Test that other workspace creation flows remain unchanged

### Manual Testing Steps:

1. Create a new user account and verify workspace setup
2. Remove a user from their last workspace and verify recovery workspace
3. Test that existing users can still create additional workspaces
4. Verify that admin panel workspace creation still works

## Performance Considerations

- The new helper performs the same database operations as before, just organized differently
- No additional queries or operations are introduced
- Transaction consistency is maintained as before

## Migration Notes

- No data migration required as this is purely a code refactoring
- Both old and new code create the same data structures
- Backward compatibility is maintained

## References

- Original research: `thoughts/shared/research/2025-01-14-ENG-982-workspace-user-membership-creation-refactoring.md`
- User creation flow: `convex/users/helpers.ts:127-152`
- Recovery flow: `convex/currentUser/mutations.ts:118-147`
- Existing helpers: `convex/workspaces/helpers.ts`, `convex/customers/helpers.ts`, `convex/workspaceMemberships/helpers.ts`
