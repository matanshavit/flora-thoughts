---
date: 2025-10-13T23:28:23+0000
researcher: Matan Shavit
git_commit: 0f79457f17630520df002d87e6d945128e057577
branch: eng-976-guest-permissions
repository: eng-976-guest-permissions
topic: "Evaluate if Phase 3 is needed - guests should not be able to access the editor canvas anymore, only the readonly view"
tags: [research, codebase, guest-permissions, readonly-view, phase3, eng-976]
status: complete
last_updated: 2025-10-13
last_updated_by: Matan Shavit
---

# Research: Evaluate if Phase 3 is needed - guests should not be able to access the editor canvas anymore, only the readonly view

**Date**: 2025-10-13T23:28:23+0000
**Researcher**: Matan Shavit
**Git Commit**: 0f79457f17630520df002d87e6d945128e057577
**Branch**: eng-976-guest-permissions
**Repository**: eng-976-guest-permissions

## Research Question

Evaluate if Phase 3 of the guest permissions implementation plan is needed, given that guests should not be able to access the editor canvas anymore, only the readonly view.

## Summary

**Phase 3 is NOT needed.** The codebase already enforces strict routing that prevents guest users from accessing the editor canvas. Guest users are automatically redirected to a completely separate readonly view (`/projects/readonly/[id]`) that does not contain any of the UI elements targeted by Phase 3 (ContributorsList, CreditsIndicator, ProjectMenuButton). The readonly view has its own dedicated UI components that provide a view-only experience without any editing or management capabilities.

## Detailed Findings

### Guest Routing System

The application implements a robust server-side enforcement system that prevents guests from accessing the editor:

1. **Automatic URL Routing** (`src/lib/routing/role-helpers.ts:5-12`)
   - The `getProjectUrl()` function returns different URLs based on user role
   - Guests get `/projects/readonly/{id}` while editors get `/projects/{id}`
   - This function is used consistently throughout the application for generating project links

2. **Server-Side Route Guards**
   - **Editor Route** (`src/app/(with-migration)/projects/[id]/page.tsx:62-64`): Redirects non-editors to readonly view
   - **Readonly Route** (`src/app/(with-migration)/projects/readonly/[id]/page.tsx:61-66`): Redirects editors back to editor view
   - These are server-side checks that cannot be bypassed by client-side navigation

3. **Project Creation Prevention** (`src/app/(with-migration)/new/route.ts:105-111`)
   - Guest users attempting to create projects are redirected to the shared projects page
   - Prevents guests from accessing project creation flows entirely

### Readonly View Architecture

The readonly view is a completely separate implementation from the editor view:

1. **Separate Component Tree**
   - **Readonly**: Uses `ViewonlyWorkspace`, `ViewonlyContainer`, `ViewonlySidebar`, `ViewonlyFooter`
   - **Editor**: Uses `ProjectWorkspace`, `WebGLMultiplayerProjectWorkspaceWrapper`, `ProjectToolsSidebar`, `ParametersSidebar`
   - No shared UI components between the two views for the targeted elements

2. **Phase 3 Target Components Not Present in Readonly**
   - ❌ **ContributorsList** - Not rendered in readonly view
   - ❌ **CreditsIndicator** - Not rendered in readonly view
   - ❌ **ProjectMenuButton** - Not rendered in readonly view
   - ❌ **"New project" menu item** - Does not exist in readonly view

3. **What Readonly View Actually Contains**
   - Project title and author information
   - Project preview thumbnail
   - Clone button (requires authentication)
   - Share link button
   - Canvas in view-only mode
   - No editing tools or management features

### Current Access Control Flow

1. **User clicks project link** → `getProjectUrl()` generates appropriate URL based on role
2. **User navigates to project** → Server component fetches project with role information
3. **Server enforces access** → Redirects based on `currentProjectRole`:
   - Guests to `/projects/readonly/[id]`
   - Editors to `/projects/[id]`
4. **Appropriate view renders** → Completely different UI based on route

### Phase 3 Implementation Analysis

The Phase 3 plan targets hiding UI elements in the editor canvas:
- Parameters sidebar with ContributorsList
- Project tools sidebar with CreditsIndicator
- Project menu with "New project" option

However, these components only exist in the editor view at `/projects/[id]`, which guest users **cannot access** due to server-side route guards. The readonly view at `/projects/readonly/[id]` uses entirely different components that don't include these elements.

## Code References

### Routing and Access Control
- `src/lib/routing/role-helpers.ts:5-12` - `getProjectUrl()` function for role-based routing
- `src/lib/routing/role-helpers.ts:14-16` - `isGuestUser()` helper function
- `src/app/(with-migration)/projects/[id]/page.tsx:62-64` - Editor route guard redirecting non-editors
- `src/app/(with-migration)/projects/readonly/[id]/page.tsx:61-66` - Readonly route guard redirecting editors
- `src/app/(with-migration)/new/route.ts:105-111` - New project route preventing guest creation

### Readonly View Components
- `src/components/workspace/viewonly/viewonly-workspace.tsx` - Main readonly workspace
- `src/components/workspace/viewonly/viewonly-container.tsx` - Readonly container with canvas
- `src/components/workspace/viewonly/viewonly-sidebar.tsx` - Readonly sidebar (no Phase 3 targets)
- `src/components/workspace/viewonly/viewonly-footer.tsx` - Readonly footer

### Editor View Components (Phase 3 Targets)
- `src/components/sidebars/parametersSidebar.tsx:28-32` - ContributorsList location
- `src/components/sidebars/workspace/projectToolsSidebar.tsx:293-303` - CreditsIndicator location
- `src/components/sidebars/workspace/project-menu-button.tsx:83` - "New project" menu item

### Database and Role Management
- `convex/currentUser/helpers.ts:29-48` - `getCurrentUserRole()` function
- `convex/projects/queries.ts:115-127` - Project role determination in `getProject`
- `convex/projects/queries.ts:698-710` - Project role determination in `getReadonlyProject`

## Architecture Documentation

### Role-Based Routing Pattern
The application uses a consistent pattern of role-based URL generation through a single helper function (`getProjectUrl`), ensuring all project links throughout the application respect user permissions. This eliminates the need for permission checks at every link creation point.

### Dual View Architecture
The system maintains two completely separate view implementations:
- **Editor View**: Full React Flow canvas with editing capabilities, project management tools, and real-time collaboration
- **Readonly View**: Simplified view-only canvas with minimal UI for browsing and cloning

### Server-Side Enforcement
Access control is enforced at the server component level, not through client-side checks. This ensures security cannot be bypassed through client-side manipulation or direct URL navigation.

### Permission Hierarchy
1. **Workspace Level**: User role determined by `workspaceMemberships` table
2. **Project Level**: Can be overridden by `projectMemberships` table
3. **Route Level**: Server components enforce access based on computed role

## Historical Context (from thoughts/)

### Original Plan
- `thoughts/shared/plans/2025-10-13-ENG-976-guest-ui-visibility.md` - Phase 3 planned to hide UI elements in canvas view
- Plan assumed guests could access the editor canvas and needed client-side hiding

### Current Implementation Status
- Phase 1 (Dashboard Header) - Completed and verified
- Phase 2 (Dashboard Sidebar) - Completed and verified
- Phase 3 (Canvas UI Elements) - Not needed due to existing route separation

## Related Research

- `thoughts/shared/research/2025-10-13-ENG-976-guest-ui-elements.md` - Initial research identifying UI elements to hide

## Open Questions

None - The research conclusively shows that Phase 3 is unnecessary given the current architecture.

## Recommendation

**Do not implement Phase 3.** The existing server-side route guards and separate readonly view implementation already prevent guest users from accessing any editor UI elements. The Phase 3 changes would add unnecessary code to components that guest users cannot reach.

Instead, the implementation should be considered complete with Phases 1 and 2, which successfully hide creation and management UI elements from the dashboard where guest users do have access.