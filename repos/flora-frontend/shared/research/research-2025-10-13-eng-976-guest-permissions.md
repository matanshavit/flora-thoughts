---
date: 2025-10-13T15:24:02-04:00
researcher: matanshavit
git_commit: f04bc244858869e55f5600fab482b044b7b4b2cf
branch: eng-976-guest-permissions
repository: florafauna-ai/flora-frontend
topic: "PR 2025 Guest Permissions Review Comments Analysis"
tags: [research, codebase, guest-permissions, routing, workspace-invites, ui-elements]
status: complete
last_updated: 2025-10-13
last_updated_by: matanshavit
---

# Research: PR 2025 Guest Permissions Review Comments Analysis

**Date**: 2025-10-13T15:24:02-04:00
**Researcher**: matanshavit
**Git Commit**: f04bc244858869e55f5600fab482b044b7b4b2cf
**Branch**: eng-976-guest-permissions
**Repository**: florafauna-ai/flora-frontend

## Research Question

Analysis of PR 2025 review comments regarding guest permissions implementation:
1. UserRoleEnum exists in /shared - should be used instead of duplicating
2. Duplicated routing logic for guest users (third instance of code duplication)
3. Workspace invite flow incorrectly creates projects for guest users
4. Project routing doesn't redirect guests to readonly view when URL is manually edited
5. UI elements that should be hidden for guest users

## Summary

The codebase currently implements guest permissions with several architectural patterns that need documentation:

1. **UserRoleEnum is defined in two locations**: Both `/shared/constants/index.ts` and `/shared/types/user.ts` define role types, with the enum in constants being the primary definition.

2. **Routing logic duplication exists in 4 locations**: The pattern `userRole === 'guest' ? appRoutes.readonlyProjectView() : appRoutes.project()` appears in multiple components.

3. **Workspace invite flow has an onboarding gap**: Guest users joining workspaces are redirected to `/new` which creates a project, regardless of their role permissions.

4. **Project-level redirects work correctly**: The system properly redirects guests to readonly views at the project level, but this happens after initial access attempts.

5. **UI elements use mixed patterns**: Some elements are disabled for guests while others are hidden, with no consistent approach across components.

## Detailed Findings

### 1. UserRoleEnum Definition

#### Current Implementation
**Location**: `/shared/constants/index.ts:28-32`

```typescript
export enum UserRoleEnum {
  ADMIN = "admin",
  EDITOR = "editor",
  GUEST = "guest",
}
```

#### Alternative Type Definition
**Location**: `/shared/types/user.ts:21-25`

```typescript
export type UserRoleType = "admin" | "member" | "viewer";

export interface UserWithRole extends BaseUser {
  role: UserRoleType;
}
```

The codebase uses `UserRoleEnum` from constants throughout, but also has the `UserRoleType` which uses different terminology ("member" vs "editor", "viewer" vs "guest").

### 2. Routing Logic Duplication

#### Pattern Instance Locations

The routing pattern checking for guest users appears in these files:

1. **`src/components/dashboard/home/project-block.tsx:322-324`**
   ```typescript
   const projectUrl = userRole === 'guest'
     ? appRoutes.readonlyProjectView(project._id.toString())
     : appRoutes.project(project._id);
   ```

2. **`src/components/dashboard/projects/projects-list/project-block-with-context-menu.tsx:103-105`**
   ```typescript
   const url = userRole === 'guest'
     ? appRoutes.readonlyProjectView(projectId.toString())
     : appRoutes.project(projectId);
   ```

3. **`src/components/sidebars/dashboard/dashboard-sidebar/favorite-project-row.tsx`**
   - Lines 95-98 (router.push)
   - Lines 116-118 (href attribute)

4. **`src/app/(with-migration)/projects/[id]/page.tsx:62-64`** (Server-side redirect)
   ```typescript
   if (!projectResponse || projectResponse.currentProjectRole === ProjectRole.GUEST) {
     return redirect(appRoutes.readonlyProjectView(convexProjectId.toString()));
   }
   ```

### 3. Workspace Invite Flow Issue

#### Current Flow
1. User accepts workspace invite at `/join-workspace/[invitationCode]`
2. System creates workspace membership with invitation role (guest/editor/admin)
3. **All users redirected to `/projects`** regardless of role
4. Projects page checks if user owns 0 projects
5. **If 0 projects, redirects to `/new`** which creates a project
6. No role checking prevents guests from creating projects

#### Key Files
- `src/app/(with-migration)/join-workspace/[invitationCode]/page.tsx:38` - Redirects all users to `/projects`
- `src/app/(dashboard)/projects/page.tsx:28` - Triggers onboarding redirect
- `src/app/(with-migration)/new/route.ts:72` - Creates project without role check
- `convex/projects/mutations.ts:70-89` - No workspace role validation

### 4. Project Routing and Redirects

#### Two-Tier Permission System

**Workspace-Level** (`workspaceMemberships`)
- Controls workspace access and seat allocation
- Roles: admin, editor, guest

**Project-Level** (`projectMemberships`)
- Controls project-specific permissions
- Roles: editor, guest
- Determines routing to regular vs readonly views

#### Redirect Logic
- `/projects/[id]/page.tsx` checks project-level role and redirects guests to `/projects/readonly/[id]`
- `/projects/readonly/[id]/page.tsx` redirects non-guests back to `/projects/[id]`
- Bidirectional routing ensures users see appropriate view

#### Access vs Role
- Workspace membership grants **access** to public projects
- Project membership determines **role** (edit vs readonly)
- Project owners always have editor role

### 5. UI Elements for Guest Users

#### What Guests Cannot Do
- **Folder Management**: Create, rename, delete, or reorder folders
- **Project Management**: Create, rename, duplicate, delete projects (unless owner)
- **Workspace Settings**: Change settings, upload logo, invite users
- **Admin Views**: See billing, SAML, plans tabs in settings

#### What Guests Can Do
- View all workspace projects in readonly mode
- Add/remove favorites
- Access profile and preference settings
- Navigate between workspaces

#### Common Implementation Patterns

1. **Role Check Pattern**:
   ```typescript
   const { data: currentUserRole } = useStatusQuery(api.currentUser.queries.currentUserRole);
   const isGuest = currentUserRole?.role === UserRoleEnum.GUEST;
   ```

2. **Conditional Disable**:
   ```typescript
   <Button disabled={userRole === UserRoleEnum.GUEST}>Action</Button>
   ```

3. **Conditional Rendering**:
   ```typescript
   {userRole !== UserRoleEnum.GUEST && <Button>Action</Button>}
   ```

## Code References

### Role Definitions
- `shared/constants/index.ts:28-32` - UserRoleEnum definition
- `shared/types/user.ts:21-25` - UserRoleType definition
- `src/lib/projects/@types.ts:117-120` - ProjectRole enum

### Routing Logic
- `src/components/dashboard/home/project-block.tsx:322-324` - Project block URL generation
- `src/components/dashboard/projects/projects-list/project-block-with-context-menu.tsx:103-105` - Context menu URL generation
- `src/components/sidebars/dashboard/dashboard-sidebar/favorite-project-row.tsx:95-118` - Sidebar favorite routing
- `src/app/(with-migration)/projects/[id]/page.tsx:62-64` - Server-side guest redirect

### Workspace Invite Flow
- `src/app/(with-migration)/join-workspace/[invitationCode]/page.tsx:38` - Post-invite redirect
- `src/app/(dashboard)/projects/page.tsx:28` - Onboarding redirect trigger
- `src/components/projects/onboarding-redirect.tsx:17` - Client-side redirect to /new
- `src/app/(with-migration)/new/route.ts:72` - Project creation without role check

### Permission Checks
- `convex/projects/helpers.ts:11-62` - userHasAccessToProject helper
- `convex/projects/queries.ts:116-127` - Project role determination
- `convex/currentUser/mutations.ts:246-251` - Workspace membership creation

### UI Components
- `src/components/projects/project-context-menu.tsx:85-101` - Context menu permissions
- `src/components/dashboard/account/settings-panel.tsx:43-52` - Settings tab visibility
- `src/components/sidebars/dashboard/dashboard-sidebar/workspace-section.tsx:99-120` - Folder button disabling
- `src/lib/workspace-memberships/helpers.ts:3-12` - Permission helper functions

## Architecture Documentation

### Permission Hierarchy
1. **Workspace Membership** determines if user can see workspace projects
2. **Project Membership** determines how user interacts with specific projects
3. **Project Ownership** grants full editor permissions
4. Guest workspace members can have different project-level roles

### Routing Architecture
- Client-side components generate role-appropriate URLs
- Server-side pages validate and redirect based on actual permissions
- Middleware handles authentication but not role-based routing
- All role decisions happen in page components after data fetching

### Data Flow Patterns
1. User action → Role check → Generate appropriate URL → Navigation
2. Page load → Fetch permissions → Validate role → Redirect if needed
3. Workspace invite → Create membership → Switch workspace → Redirect to projects

## Historical Context

The current implementation shows evolution from a simpler permission model to a two-tier system. The duplication of routing logic suggests rapid feature development without centralization. The workspace invite flow appears designed for regular users and wasn't updated for guest restrictions.

## Related Research

This research specifically addresses PR 2025 review comments. Related areas for future investigation:
- Centralized routing logic implementation
- Workspace invite flow redesign for role-based onboarding
- UI component standardization for role-based rendering

## Open Questions

1. Should `UserRoleType` and `UserRoleEnum` be unified into a single definition?
2. Is the two-tier permission system (workspace + project) the intended architecture?
3. Should guests be prevented from accessing the `/new` route entirely?
4. What specific UI elements were "marked in red or pink" in the review video?
5. Should routing logic be centralized into a single helper function?