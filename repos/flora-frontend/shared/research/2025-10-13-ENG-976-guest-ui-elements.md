---
date: 2025-10-13T18:58:03-04:00
researcher: Claude
git_commit: 0dc7eb1efe0089453a8f63fc190ecf2f78c044dc
branch: eng-976-guest-permissions
repository: eng-976-guest-permissions
topic: "ENG-976: Guest UI Elements to Remove from Dashboard and Canvas"
tags: [research, codebase, guest-permissions, ui-components, dashboard, canvas]
status: complete
last_updated: 2025-10-13
last_updated_by: Claude
---

# Research: ENG-976 Guest UI Elements to Remove from Dashboard and Canvas

**Date**: 2025-10-13T18:58:03-04:00
**Researcher**: Claude
**Git Commit**: 0dc7eb1efe0089453a8f63fc190ecf2f78c044dc
**Branch**: eng-976-guest-permissions
**Repository**: eng-976-guest-permissions

## Research Question

Identify and document all UI elements that should be removed from guest UI on the /projects dashboard and in the React Flow (r3f) canvas, specifically:
- All upper right buttons (New Project, Invite, members counter)
- Two lower left buttons (credits amount/button, trash button)
- Private section in left sidebar

## Summary

The codebase contains six main UI elements identified for removal from guest views across two distinct interfaces: the projects dashboard and the canvas workspace. These components are currently rendered without guest-specific visibility controls. The permission system exists with three roles (ADMIN, EDITOR, GUEST) defined in `UserRoleEnum`, and there are established patterns for role-based UI control throughout the codebase, but these patterns are not yet applied to the identified components.

## Detailed Findings

### Dashboard UI Components (/projects page)

#### 1. New Project Button
**Location**: `src/components/ui/createProjectButton.tsx`
**Rendered in**: `src/components/dashboard/projects/header/dashboard-header.tsx:98`
**Current implementation**:
- Component that opens `/new` route in new tab with optional `folderId` and `isPrivate` parameters
- The `/new` route handler (`src/app/(with-migration)/new/route.ts:109-111`) already redirects guests to shared projects page
- However, the button itself is still visible to guests in the UI

#### 2. Invite Button
**Location**: `src/components/dashboard/invite-user-dialog.tsx`
**Rendered in**: `src/components/dashboard/projects/header/dashboard-header.tsx:87-97`
**Current implementation**:
- Dialog wrapper that checks if user is on free plan
- Opens settings modal for free users, invite dialog for paid users
- No guest-specific visibility control currently implemented

#### 3. Members Counter
**Location**: Inline component in `src/components/dashboard/projects/header/dashboard-header.tsx:66-86`
**Current implementation**:
- Button showing `currentWorkspace?.membershipsCount`
- Opens settings panel to "people" tab when clicked
- No role-based visibility control

#### 4. Credits Amount/Button
**Location**: `src/components/billing/credits-indicator.tsx`
**Rendered in**: `src/components/sidebars/dashboard/dashboard-sidebar/index.tsx:189-198`
**Current implementation**:
- Shows user's available credits with Flower icon
- Positioned in bottom controls section of sidebar
- Opens settings to "plans" tab when clicked

#### 5. Trash Button
**Location**: Inline component in `src/components/sidebars/dashboard/dashboard-sidebar/index.tsx:200-222`
**Current implementation**:
- Links to trash route (`appRoutes.trash`)
- Shows trash icon with tooltip
- Part of bottom controls section

#### 6. Private Section
**Location**: `src/components/sidebars/dashboard/dashboard-sidebar/private-section.tsx`
**Rendered in**: `src/components/sidebars/dashboard/dashboard-sidebar/index.tsx:175-178`
**Current implementation**:
- Section showing private projects and folders
- Includes "New Folder" button and drag-drop functionality
- Currently visible to all users

### Canvas UI Components (React Flow workspace)

#### Upper Right Buttons (Contributors/Invite)
**Main Container**: `src/components/sidebars/parametersSidebar.tsx:23`
**Components**:
- Contributors list: `src/components/multiplayer/contributors-list/index.tsx:28`
- Share/Invite dialog: `src/components/projects/share-project-dialog.tsx`
- Project menu: `src/components/sidebars/workspace/project-menu-button.tsx` (includes "New project" option)

#### Lower Left Buttons (Credits)
**Location**: `src/components/sidebars/workspace/projectToolsSidebar.tsx:293`
**Current implementation**:
- Same `CreditsIndicator` component used in dashboard
- Positioned absolutely at `left-4 top-1/2`

### Permission System Architecture

#### Role Definitions
**Core Types**: `shared/constants/index.ts`
```typescript
UserRoleEnum = {
  ADMIN: "admin",
  EDITOR: "editor",
  GUEST: "guest"
}
```

#### Helper Functions
**Role Checking**: `src/lib/routing/role-helpers.ts`
- `isGuestUser()`: Returns true if user role is GUEST
- `getProjectUrl()`: Returns readonly URL for guest users

#### Permission Patterns Currently Used

The codebase has established patterns for role-based UI control:

1. **Conditional Rendering**: Components wrapped in `{!isGuestUser(role) && ...}`
2. **Button Disabling**: `disabled={isGuestUser(role)}`
3. **Route Redirects**: Server-side checks redirecting guests
4. **Context Menu Items**: Disabled based on role and ownership
5. **Computed Permissions**: Objects defining allowed actions per role

Examples found in:
- `src/components/dashboard/account/settings-tabs/people-tab.tsx`: Admin-only features
- `src/components/sidebars/dashboard/dashboard-sidebar/workspace-section.tsx:105`: Editor role restrictions
- `src/components/projects/project-context-menu.tsx:85-101`: Role-based menu items

### Data Flow

#### Query Dependencies
1. **currentUserRole**: Used to get user's role in workspace
2. **currentWorkspace**: Provides workspace data including memberships
3. **currentCustomer**: Used for billing/credit information

#### Context Providers
- `DashboardProjectsProvider`: Dashboard state management
- `ProjectStore`: Canvas/workspace state management
- `SettingsContext`: Settings modal management

## Code References

### Components to Modify
- `src/components/dashboard/projects/header/dashboard-header.tsx:66-100` - Header buttons section
- `src/components/sidebars/dashboard/dashboard-sidebar/index.tsx:175-222` - Sidebar sections
- `src/components/sidebars/parametersSidebar.tsx:23-50` - Canvas upper right area
- `src/components/sidebars/workspace/projectToolsSidebar.tsx:293` - Canvas credits indicator

### Permission Check Locations
- `src/lib/routing/role-helpers.ts:14-16` - isGuestUser helper
- `convex/currentUser/queries.ts:72-91` - currentUserRole query
- `src/app/(with-migration)/new/route.ts:105-111` - Existing guest redirect example

## Architecture Documentation

### Current Permission Check Flow
1. Client components query `api.currentUser.queries.currentUserRole`
2. Role is compared against `UserRoleEnum` constants
3. UI elements are conditionally rendered or disabled based on role
4. Server-side routes validate permissions before operations

### Dashboard View State Management
The dashboard uses `DashboardViewState` enum to control which UI elements appear:
- Components already check `viewState` for conditional rendering
- Pattern can be extended with role checks: `viewState !== Trash && !isGuestUser(role)`

## Historical Context (from thoughts/)

### Related Documents Found
- `thoughts/shared/plans/2025-01-13-ENG-976-guest-permission-redirect.md` - Guest Permission Redirect Implementation Plan
- `thoughts/shared/plans/2025-10-13-ENG-976-guest-permissions-fixes.md` - Guest Permissions Fixes Implementation Plan
- `thoughts/shared/research/research-2025-10-13-eng-976-guest-permissions.md` - Previous research on UserRoleEnum definitions
- `thoughts/shared/prs/2063_description.md` - PR with security fixes and code organization

These documents indicate ongoing work to implement guest permissions, with focus on security and proper role-based access control.

## Related Research

- Previous research documented UserRoleEnum definition locations and usage patterns
- Implementation plans outline the desired end state for guest permissions
- PR 2063 addressed initial security concerns

## Open Questions

1. Should the entire bottom controls section be hidden from guests, or just specific buttons?
2. Are there any canvas-specific tools in `projectToolsSidebar` that guests should retain access to?
3. Should view-only mode components (`src/components/workspace/viewonly/`) be leveraged for guest users?
4. How should the empty state appear when all action buttons are hidden from guests?