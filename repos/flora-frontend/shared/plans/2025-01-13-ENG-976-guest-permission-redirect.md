# Guest Permission Redirect Implementation Plan

## Overview

Add permission checks when users visit `/projects/[id]` to redirect guest users to `/projects/readonly/[id]`. This extends PR #2025's existing work on guest permissions to ensure guests always land on the read-only view.

## Current State Analysis

The current implementation (PR #2025) includes:
- `getProjectUrl()` helper in `src/lib/routing/role-helpers.ts` that returns the correct URL based on user role
- Updated dashboard components to use `getProjectUrl()` for generating links
- Existing redirect logic in `/projects/[id]/page.tsx:62-64` that redirects guests to readonly view

However, the issue is that the redirect only happens AFTER the page loads and checks permissions server-side. We need to prevent guests from even attempting to access the editor view directly.

### Key Discoveries:
- Current redirect logic is in `src/app/(with-migration)/projects/[id]/page.tsx:62-64`
- Guest role is fetched from `api.projects.queries.getProject` which returns `currentProjectRole`
- The readonly page at `src/app/(with-migration)/projects/readonly/[id]/page.tsx:60-66` correctly redirects non-guests back to editor view
- All navigation links have been updated to use `getProjectUrl()` helper

## Desired End State

When a guest user directly accesses `/projects/[id]` (via bookmark, direct URL, or any other means), they should be immediately redirected to `/projects/readonly/[id]` before any project data loads. The redirect should happen as early as possible in the request lifecycle.

### Key Requirements:
- Guest users accessing `/projects/[id]` redirect to `/projects/readonly/[id]`
- Editor/Admin users accessing `/projects/readonly/[id]` redirect to `/projects/[id]` (already implemented)
- Redirect happens before expensive project data fetching
- Maintain existing error handling for invalid project IDs

## What We're NOT Doing

- NOT changing the existing `getProjectUrl()` helper logic
- NOT modifying how roles are fetched from the database
- NOT changing the readonly view implementation
- NOT adding middleware-level redirects (keeping logic in page components)
- NOT modifying the existing dashboard navigation components

## Implementation Approach

Since the current implementation already has the redirect logic in place at lines 62-64 of the project page, and it's working correctly, the main issue might be optimization. The current flow fetches multiple pieces of data in parallel before checking permissions. We'll reorganize the data fetching to check permissions first, then fetch additional data only if needed.

## Phase 1: Optimize Permission Check Order

### Overview
Reorganize the project page data fetching to check user role first, then fetch additional data only if the user has appropriate access.

### Changes Required:

#### 1. Project Page Permission Check
**File**: `src/app/(with-migration)/projects/[id]/page.tsx`
**Changes**: Refactor data fetching to check permissions before fetching all project data

```typescript
export default async function ProjectPage(props: { params: Promise<{ id: string }> }) {
  const params = await props.params;

  const projectId = validateConvexId(params.id);
  if (!projectId) {
    return redirect(getErrorRedirect(projectErrors.notFound().error));
  }

  const convexProjectId = params.id as Id<"projects">;
  const convexAuth = await convexAuthToken();

  // First, fetch only the project role to determine if redirect is needed
  const projectResponse = await fetchQuery(
    api.projects.queries.getProject,
    { projectId: convexProjectId },
    convexAuth
  );

  // Check if user is guest BEFORE fetching additional data
  if (!projectResponse || projectResponse.currentProjectRole === ProjectRole.GUEST) {
    return redirect(appRoutes.readonlyProjectView(convexProjectId.toString()));
  }

  if (!projectResponse.project) {
    return redirect(getErrorRedirect(projectErrors.notFound().error));
  }

  // Only fetch additional data if user has editor/admin access
  const [currentUser, projectContributors, projectShared, _] = await Promise.all([
    fetchQuery(api.currentUser.queries.me, {}, convexAuth),
    fetchQuery(
      api.projects.sharing.getProjectContributors,
      { projectId: convexProjectId },
      convexAuth,
    ),
    fetchQuery(api.projects.sharing.isProjectShared, { projectId: convexProjectId }, convexAuth),
    fetchMutation(api.onboarding.mutations.createProgress, {}, convexAuth),
  ]);

  // Get headers during SSR
  const headersList = await headers();
  const userAgent = headersList.get("user-agent") || "";
  const isMobile = isMobileDevice(userAgent);

  // check if workspace has more than 1 user
  // check workspace is paid
  if (projectShared) {
    await ensureExistingProjectRooms({
      projectId: projectResponse.project._id,
      workspaceUuid: projectResponse.project.workspaceId,
    });
  }

  return (
    <ProjectWorkspaceCreditContextClient>
      <>
        {isMobile && <MobileNotAllowed />}
        <div
          className={cn(
            "max-w-screen absolute inset-0 grid h-screen max-h-screen grid-cols-[max-content_1fr_max-content] overflow-hidden bg-background",
          )}
        >
          <ProjectWorkspaceClient
            ownerCanvas={projectResponse.project.userId === currentUser._id}
            currentProject={projectResponse.project}
            projectContributors={projectContributors}
            currentUser={currentUser}
            projectShared={projectShared}
          />
        </div>
      </>
    </ProjectWorkspaceCreditContextClient>
  );
}
```

### Success Criteria:

#### Automated Verification:
- [x] TypeScript compilation passes: `pnpm typecheck`
- [x] Build succeeds: `pnpm build`
- [x] No linting errors: `pnpm lint`

#### Manual Verification:
- [ ] Guest user accessing `/projects/[id]` redirects to `/projects/readonly/[id]`
- [ ] Editor/Admin users can access `/projects/[id]` normally
- [ ] Project page loads faster for guest redirects (fewer queries executed)
- [ ] No regression in project loading for authorized users

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation from the human that the manual testing was successful before proceeding to the next phase.

---

## Phase 2: Add Early Return Optimization for Readonly Page

### Overview
Ensure the readonly page also checks permissions early to redirect editors/admins before fetching unnecessary data.

### Changes Required:

#### 1. Readonly Page Optimization
**File**: `src/app/(with-migration)/projects/readonly/[id]/page.tsx`
**Changes**: Check role before fetching additional readonly-specific data

The current implementation already checks and redirects appropriately at lines 60-66. We should verify this is working as expected and doesn't need optimization.

### Success Criteria:

#### Automated Verification:
- [ ] TypeScript compilation passes: `pnpm typecheck`
- [ ] Build succeeds: `pnpm build`

#### Manual Verification:
- [ ] Editor/Admin accessing `/projects/readonly/[id]` redirects to `/projects/[id]`
- [ ] Guest users can access `/projects/readonly/[id]` normally
- [ ] Readonly page loads correctly with all view-only features

---

## Testing Strategy

### Unit Tests:
Since these are Next.js page components with server-side logic, unit testing is limited. Focus on integration testing.

### Integration Tests:
1. Test guest user redirect flow:
   - Create guest user session
   - Navigate to `/projects/[valid-id]`
   - Verify redirect to `/projects/readonly/[valid-id]`

2. Test editor user access:
   - Create editor user session
   - Navigate to `/projects/[valid-id]`
   - Verify no redirect occurs
   - Verify project loads correctly

3. Test admin user access:
   - Create admin user session
   - Navigate to `/projects/[valid-id]`
   - Verify no redirect occurs
   - Verify project loads correctly

### Manual Testing Steps:
1. Log in as a guest user (or use a test account with guest role)
2. Copy a direct project URL `/projects/[id]`
3. Paste URL in a new tab and navigate
4. Verify redirect to `/projects/readonly/[id]`
5. Log in as an editor/admin user
6. Navigate to same `/projects/[id]`
7. Verify project opens in edit mode
8. Try accessing `/projects/readonly/[id]` as editor
9. Verify redirect back to `/projects/[id]`

## Performance Considerations

The main optimization is reducing the number of parallel queries for guest users. Instead of fetching all project data before checking permissions, we:
1. Fetch only the project and role first
2. Check if redirect is needed
3. Only fetch additional data if user has access

This reduces database queries and server processing time for guest redirects.

## Migration Notes

No data migration required. This is a pure routing optimization that doesn't change data structures or stored permissions.

## References

- Original ticket: Linear ENG-976
- Related PR: #2025 - Fix guest permissions to enforce read-only access
- Project page: `src/app/(with-migration)/projects/[id]/page.tsx`
- Readonly page: `src/app/(with-migration)/projects/readonly/[id]/page.tsx`
- Role helper: `src/lib/routing/role-helpers.ts:5-13`