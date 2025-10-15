---
date: 2025-10-15 11:30:40 EDT
researcher: matanshavit
git_commit: 0a212379dd23cade8bb7211802c1b3c8cf3298c4
branch: eng-1082
repository: eng-1082
topic: "ENG-1082: NEXT_REDIRECT error when visiting view-only project links"
tags: [research, codebase, project-sharing, authentication, view-only, next-redirect, guest-permissions]
status: complete
last_updated: 2025-10-15
last_updated_by: matanshavit
---

# Research: ENG-1082 - NEXT_REDIRECT Error When Visiting View-Only Project Links

**Date**: 2025-10-15 11:30:40 EDT
**Researcher**: matanshavit
**Git Commit**: 0a212379dd23cade8bb7211802c1b3c8cf3298c4
**Branch**: eng-1082
**Repository**: eng-1082

## Research Question
When sharing a view-only link to a project (a generated join-project link), visitors on different accounts get an error page that says NEXT_REDIRECT. The intended behavior is that they see the project in view-only mode.

## Summary
The codebase implements a comprehensive project sharing system with view-only mode, but does not explicitly handle NEXT_REDIRECT errors. The issue likely stems from Next.js's internal redirect mechanism when `redirect()` is called in server components. The system uses role-based access control where users with `ProjectRole.GUEST` should be redirected to `/projects/readonly/[id]`, while the join-project invitation flow accepts invitations and determines appropriate access levels. The NEXT_REDIRECT error appears when Next.js's redirect function throws its internal error that should be caught automatically but is being exposed to users instead.

## Detailed Findings

### Project Sharing System Architecture

#### Join-Project Flow
The join-project invitation system is implemented at `/src/app/(with-migration)/join-project/[invitationCode]/page.tsx:8-36`. When a user visits a join-project link:

1. The server component immediately processes the invitation code
2. Calls `acceptProjectInvitation` mutation at line 17-21
3. On success, redirects to the project at line 23
4. On error, shows `GenericErrorBlock` with error message (lines 24-34)

The backend invitation processing at `/convex/projects/mutations.ts:523-614` handles:
- Invitation lookup by code (lines 532-535)
- Email validation if invitation is email-specific (lines 542-546)
- Expiration checking (lines 549-551)
- Creating project membership with appropriate role (lines 584-590)
- Cleaning up single-use email invitations (lines 593-595)

#### Authentication and Authorization Flow

The middleware at `/src/middleware.ts:38-94` implements a multi-layer authentication system:

1. **Protected Route Matching** (lines 48-50): Defines protected routes including `/join-project/*`
2. **Authentication Check** (lines 52-53): Uses Clerk to verify user authentication
3. **Unauthenticated Handling** (lines 70-73): Calls `redirectToSignIn({ returnBackUrl: request.url })` for protected pages
4. **Convex User Sync** (lines 76-90): Ensures authenticated users exist in Convex database

The project access authorization at `/src/app/(with-migration)/projects/[id]/page.tsx:40-68`:
1. Validates project ID format (lines 43-46)
2. Fetches project with user's role (line 52)
3. Redirects guests to readonly view (lines 62-64)
4. Redirects unauthorized users to access denied (lines 66-68)

#### View-Only Mode Implementation

The view-only system has multiple entry points:

1. **Community Projects** (`/src/app/(with-migration)/view/[id]/page.tsx`): Public view for community projects
2. **Read-Only Projects** (`/src/app/(with-migration)/projects/readonly/[id]/page.tsx`): Authenticated guest access
3. **Role-Based Redirection**: Editors/owners cannot access readonly view (lines 61-66)

The implementation enforces read-only restrictions through:
- `ProjectContextProvider` with `viewOnly={true}` prop (`/src/components/workspace/viewonly/viewonly-workspace.tsx:56`)
- `GenerationService` blocking generations (`/src/lib/generation/generation-service.ts:95-97`)
- UI components disabling interactions (various toolbar and block components)
- Mock Liveblocks provider for view-only mode (`/src/lib/projects/context.tsx:1791`)

### NEXT_REDIRECT Error Patterns

#### Current Redirect Usage
The codebase uses Next.js's `redirect()` function extensively but **does not explicitly handle NEXT_REDIRECT errors**:

1. **Server Components**: Use `redirect()` from `next/navigation` with return statement
   - Example: `/src/app/(with-migration)/projects/[id]/page.tsx:45,63,67`
   - Pattern: `return redirect(getErrorRedirect(error))`

2. **Route Handlers**: Use `NextResponse.redirect()`
   - Example: `/src/app/(with-migration)/new/route.ts:85`
   - Pattern: `return NextResponse.redirect(new URL(path, request.url))`

3. **Middleware**: Uses `NextResponse.redirect()` or Clerk's `redirectToSignIn()`
   - Example: `/src/middleware.ts:45,73`

#### Error Redirect Helper
The `getErrorRedirect()` function at `/src/lib/helpers.ts:178-186` creates redirect URLs with error parameters:
- Encodes error title and description as query parameters
- Default redirect path is `/access-denied`
- Used with both `redirect()` and `NextResponse.redirect()`

#### No NEXT_REDIRECT Error Handling
Critical finding: The codebase does not catch or handle NEXT_REDIRECT errors anywhere:
- Global error handler (`/src/app/global-error.tsx`) logs all errors but doesn't filter NEXT_REDIRECT
- No try-catch blocks around `redirect()` calls
- Client components incorrectly use `redirect()` in some places (`viewonly-sidebar.tsx:115`)

### Authorization Helpers

The `userHasAccessToProject()` function at `/convex/projects/helpers.ts:11-62` performs a 4-level access check:
1. Owner check (lines 22-24)
2. Explicit sharing via `sharedProjects` table (lines 27-35)
3. Project membership via `projectMemberships` table (lines 37-46)
4. Workspace membership for non-private projects (lines 48-59)

Role determination happens in `/convex/projects/queries.ts:116-127`:
- Project owners get "editor" role
- Other users get role from `projectMemberships` table
- Returns undefined if no membership exists

### Database Schema

The project sharing system uses three main tables:
- **projectInvitations**: Stores pending invitations with codes, roles, and expiration
- **projectMemberships**: Active project members with userId, projectId, and role
- **sharedProjects**: Legacy direct project shares

## Historical Context (from thoughts/)

### Related Research and Plans

1. **Guest Permission Redirect Plan** (`thoughts/shared/plans/2025-01-13-ENG-976-guest-permission-redirect.md`):
   - Documented NEXT_REDIRECT error handling for guest permissions
   - Identified similar issues with guest role redirects

2. **Guest Permissions Fixes** (`thoughts/shared/plans/2025-10-13-ENG-976-guest-permissions-fixes.md`):
   - Comprehensive plan for fixing workspace invite flow for guests
   - Addressed routing logic duplication issues

3. **PR 2025 Guest Permissions** (`thoughts/shared/research/research-2025-10-13-eng-976-guest-permissions.md`):
   - Analysis of UserRoleEnum and routing logic problems
   - Identified workspace invite flow issues

4. **Current Ticket** (`thoughts/shared/tickets/ENG-1082.md`):
   - Documents the specific issue with view-only project links showing NEXT_REDIRECT error

## Code References

### Core Components
- `/src/app/(with-migration)/join-project/[invitationCode]/page.tsx:17` - Invitation acceptance entry point
- `/src/middleware.ts:73` - Authentication redirect for protected routes
- `/src/app/(with-migration)/projects/[id]/page.tsx:62-64` - Guest role redirect to readonly
- `/src/app/(with-migration)/projects/readonly/[id]/page.tsx:61-66` - Role validation for readonly access
- `/convex/projects/mutations.ts:532-595` - Backend invitation processing
- `/convex/projects/helpers.ts:11-62` - User access verification logic
- `/src/lib/helpers.ts:178-186` - Error redirect URL construction

### View-Only Implementation
- `/src/components/workspace/viewonly/viewonly-workspace.tsx:56` - Context setup with viewOnly flag
- `/src/lib/generation/generation-service.ts:95-97` - Generation blocking for view-only
- `/src/lib/projects/context.tsx:1791` - Mock Liveblocks for view-only mode
- `/src/components/workspace/viewonly/viewonly-container.tsx:179` - Disabled connections in canvas

### Invitation System
- `/convex/projects/sharing.ts:253-260` - Create new project invite link
- `/src/lib/projects/membership.ts` - Project membership management functions
- `/src/components/projects/share-project-dialog.tsx` - Share dialog UI component
- `/src/db/schema.ts:515` - projectInvitations table definition

## Architecture Documentation

### Authentication Flow for Join-Project Links
1. User visits `/join-project/[invitationCode]`
2. Middleware checks if route is protected (it is)
3. If unauthenticated, redirects to sign-in with return URL
4. After auth, page component accepts invitation
5. Creates project membership with appropriate role
6. Redirects to project or readonly view based on role

### Role-Based Access Control
- **ProjectRole.EDITOR**: Full project editing access
- **ProjectRole.GUEST**: Read-only access via `/projects/readonly/[id]`
- **No role**: Redirected to access denied page

### Redirect Patterns
- Server components use `redirect()` with return statement
- Middleware uses `NextResponse.redirect()` or Clerk's `redirectToSignIn()`
- Error pages use `GenericErrorBlock` with client-side timed redirects
- All authorization failures redirect to `/access-denied` with error parameters

## Related Research
- `thoughts/shared/research/2025-10-13-ENG-976-phase3-necessity.md` - Guest routing system analysis
- `thoughts/shared/research/2025-10-13-ENG-976-guest-ui-elements.md` - UI elements for guest users
- `thoughts/shared/research/2025-10-14-ENG-982-workspace-invitation-deletion.md` - Invitation management

## Open Questions
1. Why is the NEXT_REDIRECT error being exposed to users instead of being handled internally by Next.js?
2. Is there a race condition between invitation acceptance and redirect?
3. Are there specific scenarios where the redirect happens before authentication completes?
4. Is the issue related to the immediate redirect in the join-project page component?