# Guest Permissions Fixes Implementation Plan

## Overview

This plan addresses the PR review comments for ENG-976 guest permissions implementation. The main issues to resolve are: UserRoleEnum duplication, routing logic duplication across 4 components, workspace invite flow allowing guests to create projects, and ensuring UI elements are properly hidden for guest users.

## Current State Analysis

The guest permissions system is mostly functional but has architectural issues:
- UserRoleEnum is properly defined in `/shared/constants` but there's legacy code with unused role types
- Routing logic for guest users is duplicated in 4 locations without abstraction
- Workspace invite flow redirects all users to `/projects` which triggers onboarding for guests
- Project-level redirects work correctly (contrary to review comment #4)
- UI elements are mostly hidden/disabled for guests but need verification

### Key Discoveries:
- UserRoleEnum (admin/editor/guest) is the active system, USER_ROLES (admin/member/viewer) is legacy
- The routing pattern `userRole === 'guest' ? readonlyView : regularView` appears in 4 files
- Guests joining workspaces get redirected to `/new` and can create projects (critical bug)
- Server-side redirects properly enforce readonly views when URLs are manually edited
- `canInviteToWorkspace()` function is duplicated in frontend and backend

## Desired End State

After implementation:
- Single source of truth for role definitions and helper functions
- Centralized routing logic with no duplication
- Guest users cannot create projects when joining workspaces
- All UI elements properly restricted based on user roles
- Clean, maintainable codebase with clear permission patterns

### Key Success Criteria:
- No code duplication for role-based routing
- Guests redirect to `/projects/shared` or existing projects, never to `/new`
- All permission helper functions defined once and imported where needed
- UI consistently hides/disables elements for guest users

## What We're NOT Doing

- Changing the fundamental permission model (workspace vs project roles)
- Modifying the Liveblocks integration
- Refactoring the entire authentication system
- Creating new UI/UX flows (using existing pages)
- Removing legacy code that might be referenced elsewhere

## Implementation Approach

We'll fix issues in order of criticality: first preventing guests from creating projects (security issue), then centralizing routing logic (maintainability), and finally cleaning up duplications and UI elements.

## Phase 1: Fix Workspace Invite Flow for Guest Users

### Overview
Prevent guest users from being redirected to `/new` when they join a workspace with no projects. Instead, redirect them to appropriate read-only views.

### Changes Required:

#### 1. Update Projects Page Onboarding Logic
**File**: `src/app/(dashboard)/projects/page.tsx`
**Changes**: Check user role before showing onboarding redirect

```typescript
// After line 20, add:
const currentUserRole = await fetchQuery(
  api.currentUser.queries.currentUserRole,
  {},
  convexAuth,
);

// Replace line 28 with:
<OnboardingRedirect
  shouldRedirect={ownedProjectsNumber === 0 && currentUserRole?.role !== UserRoleEnum.GUEST}
/>
```

#### 2. Add Role Check to New Project Route
**File**: `src/app/(with-migration)/new/route.ts`
**Changes**: Prevent guests from creating projects

```typescript
// Add after line 98 in handleNewProjectRequest:
const currentUserRole = await fetchQuery(
  api.currentUser.queries.currentUserRole,
  {},
  convexAuth,
);

if (currentUserRole?.role === UserRoleEnum.GUEST) {
  return NextResponse.redirect(new URL("/projects/shared", request.url));
}
```

#### 3. Add Role Check to SaveProject Mutation
**File**: `convex/projects/mutations.ts`
**Changes**: Add permission check for project creation

```typescript
// Add after line 36:
if (!args.projectId) {
  // Creating new project - check permissions
  const userRole = await getCurrentUserRole(ctx);
  if (userRole.role === UserRoleEnum.GUEST) {
    throw new Error("Guest users cannot create projects");
  }
}
```

#### 4. Fix Join Workspace Redirect
**File**: `src/app/(with-migration)/join-workspace/[invitationCode]/page.tsx`
**Changes**: Redirect based on user role

```typescript
// Add after line 24:
const currentUserRole = await fetchQuery(
  api.currentUser.queries.currentUserRole,
  {},
  convexAuth,
);

// Replace line 38 with:
if (currentUserRole?.role === UserRoleEnum.GUEST) {
  return redirect(appRoutes.sharedProjects);
} else {
  return redirect(appRoutes.projects);
}
```

### Success Criteria:

#### Automated Verification:
- [x] Type checking passes: `npm run typecheck`
- [x] Linting passes: `npm run lint`
- [x] Build succeeds: `npm run build`

#### Manual Verification:
- [ ] Guest user accepting workspace invite redirects to `/projects/shared`
- [ ] Guest user with 0 projects doesn't see onboarding redirect
- [ ] Guest user cannot access `/new` route (redirects to `/projects/shared`)
- [ ] Guest user cannot create project via API mutation
- [ ] Editor/Admin users can still create projects normally

---

## Phase 2: Centralize Routing Logic

### Overview
Create a single helper function for role-based routing and use it everywhere instead of duplicating the conditional logic.

### Changes Required:

#### 1. Create Role-Based Routing Helper
**File**: `src/lib/routing/role-helpers.ts` (new file)
**Changes**: Create centralized routing helper

```typescript
import { UserRoleEnum } from "@/shared/constants";
import { appRoutes } from "./routes";
import type { Id } from "@/convex/_generated/dataModel";

export function getProjectUrl(
  projectId: string | Id<"projects">,
  userRole?: string | UserRoleEnum
): string {
  const id = typeof projectId === "string" ? projectId : projectId.toString();
  return userRole === "guest" || userRole === UserRoleEnum.GUEST
    ? appRoutes.readonlyProjectView(id)
    : appRoutes.project(id);
}
```

#### 2. Update Project Block Component
**File**: `src/components/dashboard/home/project-block.tsx`
**Changes**: Use centralized helper

```typescript
// Add import at top:
import { getProjectUrl } from "@/lib/routing/role-helpers";

// Replace lines 322-324 with:
const projectUrl = getProjectUrl(project._id, userRole);
```

#### 3. Update Project Block with Context Menu
**File**: `src/components/dashboard/projects/projects-list/project-block-with-context-menu.tsx`
**Changes**: Use centralized helper

```typescript
// Add import at top:
import { getProjectUrl } from "@/lib/routing/role-helpers";

// Replace lines 103-105 with:
const url = getProjectUrl(projectId, userRole);
```

#### 4. Update Favorite Project Row
**File**: `src/components/sidebars/dashboard/dashboard-sidebar/favorite-project-row.tsx`
**Changes**: Use centralized helper in both locations

```typescript
// Add import at top:
import { getProjectUrl } from "@/lib/routing/role-helpers";

// Replace lines 95-97 with:
const url = getProjectUrl(project._id, userRole);

// Replace lines 116-118 with:
const projectUrl = getProjectUrl(project._id, userRole);
```

### Success Criteria:

#### Automated Verification:
- [x] Type checking passes: `npm run typecheck`
- [x] Linting passes: `npm run lint`
- [x] Unit tests pass: `npm run test:unit`
- [ ] Build succeeds: `npm run build`

#### Manual Verification:
- [ ] Project links work correctly for guest users (open readonly view)
- [ ] Project links work correctly for editor/admin users (open regular view)
- [ ] Context menus generate correct URLs
- [ ] Favorite project links work correctly

---

## Phase 3: Consolidate Permission Helper Functions

### Overview
Remove duplicated permission helper functions and ensure they're defined in one place.

### Changes Required:

#### 1. Move Backend Helper to Shared Location
**File**: `convex/workspaceMemberships/helpers.ts`
**Changes**: Remove duplicate function

```typescript
// Delete lines 178-180 (canInviteToWorkspace function)
// Import from shared location instead:
import { canInviteToWorkspace } from "@/convex/permissions/helpers";
```

#### 2. Create Centralized Permission Helpers
**File**: `convex/permissions/helpers.ts` (new file)
**Changes**: Centralize all permission helpers

```typescript
import { UserRoleEnum } from "@/shared/constants";

export function canInviteToWorkspace(currentUserRole: UserRoleEnum | undefined) {
  return currentUserRole === UserRoleEnum.ADMIN || currentUserRole === UserRoleEnum.EDITOR;
}

export function canEditProject(
  currentUserRole: UserRoleEnum | undefined,
  isOwner: boolean
) {
  return isOwner || currentUserRole === UserRoleEnum.ADMIN;
}

export function canManageFolder(
  currentUserRole: UserRoleEnum | undefined,
  isPrivateFolder: boolean,
  isOwner: boolean
) {
  return (
    currentUserRole === UserRoleEnum.ADMIN ||
    (currentUserRole === UserRoleEnum.EDITOR && isPrivateFolder && isOwner)
  );
}
```

#### 3. Update Frontend Imports
**File**: `src/lib/workspace-memberships/helpers.ts`
**Changes**: Re-export from centralized location

```typescript
// Re-export from convex helpers
export { canInviteToWorkspace } from "@/convex/permissions/helpers";

// Keep frontend-specific helper
export function isFolderActionDisabled(
  currentUserRole: UserRoleEnum | undefined,
  isPrivateFolder: boolean,
) {
  return currentUserRole === UserRoleEnum.EDITOR && !isPrivateFolder;
}
```

### Success Criteria:

#### Automated Verification:
- [x] Type checking passes: `npm run typecheck`
- [x] Linting passes: `npm run lint`
- [x] Build succeeds: `npm run build`

#### Manual Verification:
- [ ] Permission checks work correctly in all locations
- [ ] No regression in functionality
- [ ] Import paths resolve correctly

---

## Phase 4: Clean Up Legacy Code

### Overview
Remove unused legacy role definitions to avoid confusion.

### Changes Required:

#### 1. Remove Unused USER_ROLES Constant
**File**: `shared/constants/index.ts`
**Changes**: Remove legacy constant

```typescript
// Delete lines 5-9 (USER_ROLES constant)
```

#### 2. Remove Unused UserRoleType
**File**: `shared/types/user.ts`
**Changes**: Remove legacy type

```typescript
// Delete line 21 (UserRoleType definition)
// Update line 24 if UserWithRole is used anywhere (check first)
```

### Success Criteria:

#### Automated Verification:
- [x] Type checking passes: `npm run typecheck`
- [x] Build succeeds: `npm run build`
- [x] No broken imports

#### Manual Verification:
- [ ] Application functions normally
- [ ] No references to old role types remain

---

## Phase 5: Verify UI Element Restrictions

### Overview
Ensure all UI elements are properly hidden/disabled for guest users as documented.

### Changes Required:

This phase involves verifying existing implementations rather than making changes. Create a test checklist:

#### Testing Checklist for Guest Users:
- [ ] Cannot see "Create Project" button
- [ ] Cannot rename projects (EditableLabel disabled)
- [ ] Cannot duplicate projects (context menu item disabled)
- [ ] Cannot delete projects (context menu item disabled)
- [ ] Cannot create folders (button disabled)
- [ ] Cannot rename folders (context menu item disabled)
- [ ] Cannot delete folders (context menu item disabled)
- [ ] Cannot drag folders to reorder
- [ ] Cannot edit workspace name (input disabled)
- [ ] Cannot upload workspace logo (no upload overlay)
- [ ] Cannot see People & Credits tab
- [ ] Cannot see Billing tab
- [ ] Cannot see SAML tab
- [ ] Cannot see Plans & Pricing tab

### Success Criteria:

#### Manual Verification:
- [ ] All items in checklist verified as guest user
- [ ] No UI elements visible that shouldn't be
- [ ] Consistent styling for disabled elements
- [ ] No console errors when guest user interacts with UI

---

## Testing Strategy

### Unit Tests:
- Test `getProjectUrl()` helper with different role inputs
- Test permission helper functions with all role combinations
- Test that mutations throw errors for unauthorized operations

### Integration Tests:
- Test complete workspace invite flow for each role
- Test project creation restrictions
- Test navigation between regular and readonly views

### Manual Testing Steps:
1. Create test workspace with admin account
2. Invite user as guest via invitation link
3. Accept invitation and verify redirect to `/projects/shared`
4. Attempt to manually navigate to `/new` (should redirect)
5. Attempt to access regular project URL (should redirect to readonly)
6. Verify all UI elements are properly restricted
7. Test with editor and admin roles to ensure no regression

## Performance Considerations

- The centralized routing helper adds minimal overhead
- Server-side redirects already validate permissions (no change)
- No additional database queries introduced

## Migration Notes

No data migration required. Changes are code-only and backward compatible.

## References

- Original research: `thoughts/shared/research/research-2025-10-13-eng-976-guest-permissions.md`
- PR #2025 review comments
- UserRoleEnum definition: `shared/constants/index.ts:28-32`
- Project role types: `src/lib/projects/@types.ts:117-120`