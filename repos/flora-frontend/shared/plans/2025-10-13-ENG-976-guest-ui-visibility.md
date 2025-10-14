# ENG-976 Guest UI Elements Visibility Implementation Plan

## Overview

Hide all creation, management, and billing-related UI elements from guest users on both the /projects dashboard and React Flow canvas to ensure guests have a read-only experience without access to features they cannot use.

## Current State Analysis

The codebase has a robust permission system with `UserRoleEnum` (ADMIN, EDITOR, GUEST) defined in `shared/constants/index.ts`. Components fetch user roles via `useStatusQuery(api.currentUser.queries.currentUserRole)`. However, the identified UI elements are currently rendered for all users without role-based visibility controls. The `/new` route already redirects guests to the shared projects page, but the UI buttons remain visible.

### Key Discoveries:
- `isGuestUser()` helper exists at `src/lib/routing/role-helpers.ts:14-16`
- Established patterns for conditional rendering with `{!isGuestUser(role) && <Component />}`
- Components already query user roles consistently
- Server-side guest redirect already implemented in `/new` route

## Desired End State

After implementation, guest users should:
- Not see any buttons for creating projects or folders
- Not see workspace management features (invite, members)
- Not see billing/credits information
- Not see private project sections
- Have a clean, read-only interface focused on viewing shared content

### Verification:
- Log in as a guest user and verify all specified elements are hidden
- Log in as an editor/admin and verify elements remain visible
- Check both dashboard and canvas views

## What We're NOT Doing

- Not changing the underlying permission logic or backend mutations
- Not modifying the server-side redirects (already working)
- Not changing the readonly project view experience
- Not hiding elements from editors or admins
- Not modifying any context menus or disabled states (keeping current patterns)

## Implementation Approach

Use the existing `isGuestUser()` helper and conditional rendering patterns to hide elements from guest users. Query the user role at the component level and conditionally render based on the result.

---

## Phase 1: Dashboard Header Elements

### Overview

Hide the New Project button, Invite button, and Members counter from guest users in the dashboard header.

### Changes Required:

#### 1. Dashboard Header Component

**File**: `src/components/dashboard/projects/header/dashboard-header.tsx`
**Changes**: Add role check and conditional rendering for action buttons

First, import the helper function at the top of the file:
```typescript
import { isGuestUser } from "@/lib/routing/role-helpers";
```

Then fetch the user role (add after line 22):
```typescript
const { data: currentUserRole } = useStatusQuery(api.currentUser.queries.currentUserRole);
```

Modify the action buttons section (lines 66-100) to conditionally render:
```typescript
{/* Members counter - hide from guests */}
{!isGuestUser(currentUserRole) && (
  <WithTooltip tooltip="Members">
    <Button
      variant="transparent"
      size="sm"
      className="hidden gap-spacing-100 px-spacing-200 text-text-light-secondary sm:flex"
      onClick={() => {
        openSettings();
        setTimeout(() => {
          const peopleTab = document.querySelector('[value="people"]');
          if (peopleTab instanceof HTMLElement) {
            peopleTab.click();
          }
        }, 0);
      }}
    >
      <UsersIcon
        width={18}
        height={18}
        className="text-icon-light-secondary"
      />
      {currentWorkspace?.membershipsCount}
    </Button>
  </WithTooltip>
)}

{/* Invite button - hide from guests */}
{!isGuestUser(currentUserRole) && (
  <InviteUserDialog>
    {(openDialog) => (
      <Button
        variant="blur"
        size="sm"
        onClick={openDialog}
      >
        Invite
      </Button>
    )}
  </InviteUserDialog>
)}

{/* New Project button - hide from guests */}
{!isGuestUser(currentUserRole) && <CreateProjectButton />}
```

### Success Criteria:

#### Automated Verification:
- [x] Type checking passes: `pnpm typecheck`
- [x] Linting passes: `pnpm lint`
- [x] Build completes successfully: `pnpm build`

#### Manual Verification:
- [ ] Guest users don't see Members counter, Invite button, or New Project button in header
- [ ] Editor users still see all three elements
- [ ] Admin users still see all three elements
- [ ] No layout shift or spacing issues when elements are hidden

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation from the human that the manual testing was successful before proceeding to the next phase.

---

## Phase 2: Dashboard Sidebar Elements

### Overview

Hide the Credits indicator, Trash button, and Private section from guest users in the dashboard sidebar.

### Changes Required:

#### 1. Dashboard Sidebar Component

**File**: `src/components/sidebars/dashboard/dashboard-sidebar/index.tsx`
**Changes**: Add role check and conditional rendering for sidebar elements

First, import the helper function at the top:
```typescript
import { isGuestUser } from "@/lib/routing/role-helpers";
```

Add role query (after line 44):
```typescript
const { data: currentUserRole } = useStatusQuery(api.currentUser.queries.currentUserRole);
```

Modify the Private section rendering (lines 175-178):
```typescript
{/* Private section - hide from guests */}
{!isGuestUser(currentUserRole) && (
  <PrivateSection
    dashboardState={dashboardState}
    onOpenFolderDialog={openFolderDialog}
  />
)}
```

Modify the bottom controls section (lines 187-222):
```typescript
<div className="mt-auto flex flex-col">
  <Separator className="border-border-transparent-secondary" />
  <div className="flex flex-row items-center gap-2 px-4 py-2">
    {/* Credits indicator - hide from guests */}
    {!isGuestUser(currentUserRole) && (
      <CreditsIndicator
        openSettingsModal={() => {
          openSettings();
          setTimeout(() => {
            const plansTab = document.querySelector('[value="plans"]');
            if (plansTab instanceof HTMLElement) {
              plansTab.click();
            }
          }, 0);
        }}
      />
    )}

    {/* Trash button - hide from guests */}
    {!isGuestUser(currentUserRole) && (
      <WithTooltip tooltip="Trash">
        <Link href={appRoutes.trash}>
          <Button
            variant="transparent"
            size="icon_sm"
            className={cn(
              "h-fit w-fit",
              dashboardState.viewState === DashboardViewState.Trash &&
                "bg-background-transparent-white-hover",
            )}
          >
            <TrashIcon
              width={16}
              height={16}
              className={cn(
                "text-icon-light-secondary hover:text-icon-light",
                dashboardState.viewState === DashboardViewState.Trash &&
                  "text-icon-light-default",
              )}
            />
          </Button>
        </Link>
      </WithTooltip>
    )}

    {/* Settings button remains visible to all users */}
    <WithTooltip tooltip="Settings">
      <Button
        onClick={() => openSettings()}
        variant="transparent"
        size="icon_sm"
        className="h-fit w-fit"
      >
        <Settings
          size={16}
          className="text-icon-light-secondary hover:text-icon-light"
        />
      </Button>
    </WithTooltip>

    <Link
      href="https://flora.support"
      target="_blank"
    >
      <Button
        variant="transparent"
        size="icon_sm"
        className="h-fit w-fit"
      >
        <CircleHelp
          size={16}
          className="text-icon-light-secondary hover:text-icon-light"
        />
      </Button>
    </Link>
  </div>
</div>
```

### Success Criteria:

#### Automated Verification:
- [x] Type checking passes: `pnpm typecheck`
- [x] Linting passes: `pnpm lint`
- [x] Build completes successfully: `pnpm build`

#### Manual Verification:
- [ ] Guest users don't see Private section in sidebar
- [ ] Guest users don't see Credits indicator in bottom controls
- [ ] Guest users don't see Trash button in bottom controls
- [ ] Settings and Help buttons remain visible to guests
- [ ] No layout issues when elements are hidden
- [ ] Sidebar bottom controls maintain proper alignment

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation from the human that the manual testing was successful before proceeding to the next phase.

---

## [WILL NOT DO] Phase 3: Canvas UI Elements

Note, this phase is not needed because guest users should only ever see the read-only canvas, not the canvas editor.

### Overview

Hide the upper right contributor/invite buttons and lower left credits indicator from guest users in the React Flow canvas workspace.

### Changes Required:

#### 1. Parameters Sidebar (Upper Right)

**File**: `src/components/sidebars/parametersSidebar.tsx`
**Changes**: Conditionally render contributors list based on role

First, import the necessary modules:
```typescript
import { isGuestUser } from "@/lib/routing/role-helpers";
import { useStatusQuery } from "@/hooks/convex/use-status-query";
import { api } from "@convex/_generated/api";
```

Add role query (after line 18):
```typescript
const { data: currentUserRole } = useStatusQuery(api.currentUser.queries.currentUserRole);
```

Modify the contributors list rendering (around line 28):
```typescript
{/* Contributors list - hide from guests */}
{!isGuestUser(currentUserRole) && <ContributorsList projectId={projectId} />}
```

#### 2. Project Tools Sidebar (Lower Left Credits)

**File**: `src/components/sidebars/workspace/projectToolsSidebar.tsx`
**Changes**: Conditionally render credits indicator

First, import the helper function:
```typescript
import { isGuestUser } from "@/lib/routing/role-helpers";
```

Add role query (after line 65):
```typescript
const { data: currentUserRole } = useStatusQuery(api.currentUser.queries.currentUserRole);
```

Modify the credits indicator rendering (around line 293):
```typescript
{/* Credits indicator - hide from guests */}
{!isGuestUser(currentUserRole) && (
  <CreditsIndicator
    readonly
    openSettingsModal={openSettingsToPlanTab}
  />
)}
```

#### 3. Project Menu Button

**File**: `src/components/sidebars/workspace/project-menu-button.tsx`
**Changes**: Hide "New project" option from guests

Import the helper:
```typescript
import { isGuestUser } from "@/lib/routing/role-helpers";
```

Add role query (after line 29):
```typescript
const { data: currentUserRole } = useStatusQuery(api.currentUser.queries.currentUserRole);
```

Modify the "New project" menu item (around line 83):
```typescript
{!isGuestUser(currentUserRole) && (
  <>
    <DropdownMenuItem
      asChild
      onClick={() => {
        const url = new URL(appRoutes.newProject, window.location.origin);
        window.open(url.href, "_blank");
      }}
    >
      <Button
        variant="dropdown"
        className="cursor-pointer"
      >
        New project
      </Button>
    </DropdownMenuItem>
    <DropdownMenuSeparator />
  </>
)}
```

### Success Criteria:

#### Automated Verification:
- [ ] Type checking passes: `pnpm typecheck`
- [ ] Linting passes: `pnpm lint`
- [ ] Build completes successfully: `pnpm build`

#### Manual Verification:
- [ ] Guest users don't see contributors list in canvas upper right
- [ ] Guest users don't see invite/share options in canvas
- [ ] Guest users don't see credits indicator in canvas lower left
- [ ] Guest users don't see "New project" in project menu dropdown
- [ ] Canvas tools (add node, assets, history, etc.) remain accessible
- [ ] No layout shifts or alignment issues in canvas

---

## Testing Strategy

### Unit Tests:
- Test `isGuestUser()` helper with various input scenarios
- Mock different user roles and verify component rendering

### Integration Tests:
- Test complete dashboard loading with guest role
- Test canvas loading with guest role
- Verify navigation flows for guests

### Manual Testing Steps:
1. Create a test guest account or use existing guest credentials
2. Log in as guest and navigate to /projects dashboard
3. Verify all specified elements are hidden in dashboard
4. Open a shared project to enter canvas view
5. Verify all specified elements are hidden in canvas
6. Log in as editor and verify elements are visible
7. Log in as admin and verify all elements are visible
8. Test on different screen sizes to ensure responsive behavior

## Performance Considerations

- Additional role queries are minimal overhead (already cached by Convex)
- Conditional rendering reduces DOM elements for guests (slight performance improvement)
- No impact on initial page load or runtime performance

## Migration Notes

No data migration required. These are UI-only changes that take effect immediately upon deployment.

## References

- Original research: `thoughts/shared/research/2025-10-13-ENG-976-guest-ui-elements.md`
- Role helper location: `src/lib/routing/role-helpers.ts:14-16`
- UserRoleEnum definition: `shared/constants/index.ts:28-32`
- Existing patterns: `src/components/dashboard/account/settings-panel.tsx` (admin-only tabs)
