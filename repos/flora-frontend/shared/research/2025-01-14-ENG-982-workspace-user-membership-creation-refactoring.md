---
date: 2025-01-14T18:44:09-05:00
researcher: Claude
git_commit: 012cb25b20c8720be1fd9b5c2303ba15b3e1f8b3
branch: eng-982-remove-from-workspace
repository: flora-frontend
topic: "ENG-982 Refactoring workspace, user, and membership creation patterns"
tags: [research, codebase, workspaces, users, memberships, refactoring, convex, helpers]
status: complete
last_updated: 2025-01-14
last_updated_by: Claude
---

# Research: ENG-982 Refactoring Workspace, User, and Membership Creation Patterns

**Date**: 2025-01-14T18:44:09-05:00
**Researcher**: Claude
**Git Commit**: 012cb25b20c8720be1fd9b5c2303ba15b3e1f8b3
**Branch**: eng-982-remove-from-workspace
**Repository**: flora-frontend

## Research Question

ENG-982: Investigate refactoring the creation of workspace, user, and membership from two locations into a shared TypeScript file that both can use. Specifically examining:
- `convex/users/helpers.ts` lines 127-152
- `convex/currentUser/mutations.ts` lines 118-147

## Summary

The codebase currently has two locations where workspaces are created alongside customers and memberships in nearly identical patterns:

1. **User Creation Flow** (`convex/users/helpers.ts:127-152`): Creates workspace when new user doesn't join an enterprise workspace. Allocates 500 default credits.

2. **Recovery Flow** (`convex/currentUser/mutations.ts:118-147`): Creates recovery workspace when user is removed from their last workspace. Allocates 0 credits (since it's a recovery scenario).

Both locations follow the same pattern:
1. Create workspace with owner
2. Create customer with credits
3. Create workspace membership as admin
4. Update user's current workspace

The codebase already has established patterns for shared helper functions in `convex/[domain]/helpers.ts` files. There are existing helper functions for individual operations (`createWorkspace`, `createCustomer`, `createWorkspaceMembership`) that could be composed into a higher-level helper for this common pattern.

## Detailed Findings

### Current Implementation Locations

#### Location 1: User Creation Flow
**File**: `convex/users/helpers.ts:127-152`

```typescript
// Line 128-139: Create new workspace with the user as owner
targetWorkspaceId = await createWorkspace(ctx, {
  firstName: firstName,
  lastName: lastName,
  ownerId: userId,
});

await createCustomer(ctx, {
  workspaceId: targetWorkspaceId,
  availableCredits: 500, // Default free credits
});

// Lines 141-152: Update user and create membership
await ctx.db.patch(userId, {
  currentWorkspaceId: targetWorkspaceId,
});

const membershipId = await createWorkspaceMembership(ctx, {
  userId: userId,
  workspaceId: targetWorkspaceId,
  role: joinRole,
  active: true,
});
```

This creates a workspace for new users who don't join an existing enterprise workspace, with 500 default credits.

#### Location 2: Recovery Workspace Creation
**File**: `convex/currentUser/mutations.ts:118-147`

```typescript
// Line 123-127: Direct database insert (not using helper)
const workspaceId = await ctx.db.insert("workspaces", {
  name: workspaceName,
  ownerId: removedUser._id,
  editorSeats: 1,
});

// Line 130-133: Create customer with 0 credits
await createCustomer(ctx, {
  workspaceId,
  availableCredits: 0, // Recovery scenario, not initial signup
});

// Line 136-141: Create membership
await createWorkspaceMembership(ctx, {
  userId: removedUser._id,
  workspaceId,
  role: UserRoleEnum.ADMIN,
  active: true,
});

// Line 144-146: Update user's workspace
await ctx.db.patch(removedUserId, {
  currentWorkspaceId: workspaceId,
});
```

This creates a recovery workspace when a user has no remaining workspaces, with 0 credits since it's a recovery scenario.

### Existing Helper Functions

The codebase already provides atomic helper functions that can be composed:

#### createWorkspace Helper
**File**: `convex/workspaces/helpers.ts:4-23`

- Creates workspace with name, owner, and editor seats
- Returns workspace ID
- Used by user creation flow

#### createCustomer Helper
**File**: `convex/customers/helpers.ts:5-19`

- Creates customer record for a workspace
- Sets plan to "free" by default
- Returns customer ID
- Used by both locations

#### createWorkspaceMembership Helper
**File**: `convex/workspaceMemberships/helpers.ts:9-31`

- Creates membership linking user to workspace
- Sets role and active status
- Returns membership ID
- Used by both locations

### Architecture Patterns

#### Helper Function Organization

The convex directory follows a domain-based organization:
```
convex/
  ├── [domain]/
  │   ├── helpers.ts       # Shared logic and utilities
  │   ├── mutations.ts     # Write operations
  │   ├── queries.ts       # Read operations
```

#### Cross-Domain Helper Composition

Helpers commonly import from other domains to compose functionality:

```typescript
// Example from convex/workspaces/mutations.ts
import { createCustomer } from "../customers/helpers";
import { switchWorkspace } from "../users/helpers";
import { createWorkspaceMembership } from "../workspaceMemberships/helpers";
import { createWorkspace } from "./helpers";
```

#### Helper Characteristics

1. **Naming**: Verb-based (`createWorkspace`, `updateCustomer`)
2. **First Parameter**: Always `ctx` (MutationCtx or QueryCtx)
3. **Return Types**: Strongly typed (`Promise<Id<"table">>`)
4. **Single Responsibility**: Each helper does one thing
5. **Composability**: Helpers call other helpers

### Other Workspace Creation Locations

The research identified 5 total locations where workspaces are created:

1. **User Creation** (`convex/users/helpers.ts:129`) - Uses helper
2. **Recovery Workspace** (`convex/currentUser/mutations.ts:123`) - Direct insert
3. **Manual Creation** (`convex/workspaces/mutations.ts:38`) - Uses helper
4. **Admin Creation** (`convex/admin/workspaces/queries.ts:261`) - Direct insert
5. **Migration** (`convex/migration/workspaces.ts:54`) - Direct insert

### Credit Allocation Patterns

Different scenarios allocate different default credits:

- **First workspace for new user**: 500 credits
- **Subsequent workspaces**: 0 credits
- **Recovery workspaces**: 0 credits
- **Admin-created workspaces**: 0 credits
- **Migrated workspaces**: Preserve original credits

## Code References

- `convex/users/helpers.ts:127-152` - User creation workspace flow
- `convex/currentUser/mutations.ts:118-147` - Recovery workspace flow
- `convex/workspaces/helpers.ts:4-23` - createWorkspace helper
- `convex/customers/helpers.ts:5-19` - createCustomer helper
- `convex/workspaceMemberships/helpers.ts:9-31` - createWorkspaceMembership helper
- `convex/workspaces/mutations.ts:10-72` - Manual workspace creation
- `shared/constants/index.ts:28-32` - UserRoleEnum definition

## Architecture Documentation

### Current Helper Pattern

The codebase uses a layered helper approach:

1. **Atomic Helpers**: Single-purpose functions (`createWorkspace`, `createCustomer`)
2. **Composite Helpers**: Combine atomic helpers for workflows
3. **Domain Organization**: Helpers grouped by feature domain
4. **Cross-Domain Imports**: Helpers freely import from other domains

### Workspace-Customer-Membership Relationship

Each workspace creation follows this pattern:

```
1. Create Workspace (with owner)
   └─> Returns workspace ID
2. Create Customer (linked to workspace)
   └─> Sets credits and plan
3. Create Membership (user → workspace)
   └─> Sets role (usually admin for creator)
4. Update User (set current workspace)
   └─> Makes new workspace active
```

### Differences Between Locations

The two target locations differ in:

1. **Workspace Creation Method**:
   - User flow: Uses `createWorkspace()` helper
   - Recovery flow: Direct `ctx.db.insert()`

2. **Credit Allocation**:
   - User flow: 500 credits (initial signup bonus)
   - Recovery flow: 0 credits (recovery scenario)

3. **Workspace Naming**:
   - User flow: `"${firstName} ${lastName}'s workspace"`
   - Recovery flow: `"${firstName || email.split("@")[0]}'s Workspace"`

4. **Context**:
   - User flow: Part of initial user creation
   - Recovery flow: Edge case handling when user loses all workspaces

## Historical Context (from thoughts/)

No existing research documents were found in the thoughts directory related to workspace creation patterns or this specific refactoring task.

## Related Research

No related research documents found in `thoughts/shared/research/`.

## Open Questions

1. Should the recovery workspace creation use the existing `createWorkspace` helper instead of direct database insert for consistency?

2. Should the credit allocation logic (500 vs 0) be parameterized in a shared helper, or should it remain context-specific?

3. Should the shared helper be placed in `workspaces/helpers.ts` since it primarily creates a workspace, or in a new location like `shared/helpers/`?

4. Should other workspace creation locations (admin, migration) also use this shared pattern, or are they intentionally different?

5. What should the shared helper be named? Options might include:
   - `createWorkspaceWithCustomerAndMembership`
   - `createCompleteWorkspace`
   - `setupUserWorkspace`