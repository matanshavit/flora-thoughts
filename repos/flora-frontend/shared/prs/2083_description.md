# Pull Request Description

## Summary

Fix critical data integrity issues in Convex user management by ensuring sanitizedEmail is updated when email changes and adding a unique index on clerkId.

## What problem does this PR solve?

This PR addresses data integrity issues in the Convex user system that could lead to inconsistent user data and potential lookup failures.

[ENG-1063](https://linear.app/florafauna/issue/ENG-1063/convex-user-fixes) - Convex user fixes

The specific problems:
1. When a user updates their email, the `sanitizedEmail` field was not being updated, leading to data inconsistency
2. Missing unique index on `clerkId` could potentially allow duplicate clerk IDs in the database

## What does this PR do?

- Ensures `sanitizedEmail` is automatically updated whenever a user's email is changed
- Adds a database index on `clerkId` for better query performance and data integrity

## Technical Approach

The solution implements automatic synchronization of the `sanitizedEmail` field when users update their email addresses through the patch mutation.

### Key Changes:

- **convex/currentUser/mutations.ts**: Added logic to automatically compute and update `sanitizedEmail` when email is updated
- **convex/schema.ts**: Added `by_clerk_id` index to the users table for efficient lookups and uniqueness enforcement

### Performance Impact:

- Initial load time: No impact
- Runtime performance: Improved query performance for clerk ID lookups due to new index
- Memory usage: Minimal increase due to additional index

## Screen Recording / Screenshots

### Before:
N/A - Backend data integrity fix with no UI changes

### After:
N/A - Backend data integrity fix with no UI changes

## How to verify it

### Automated Verification:

- [ ] Type checking passes: `pnpm typecheck` (Note: Pre-existing type errors unrelated to this PR)
- [ ] Build succeeds: `pnpm build`
- [ ] Unit tests pass: `pnpm test:unit`
- [ ] Linting passes: `pnpm lint` (Command timed out during verification)

### Manual Verification Steps:

1. Update a user's email through the application
2. Verify in the Convex dashboard that both `email` and `sanitizedEmail` fields are updated
3. Check that the `by_clerk_id` index is created in the database schema

### Expected Behavior:

- When a user updates their email, the `sanitizedEmail` field should automatically be updated with the sanitized version
- Database queries using `clerkId` should be faster due to the new index
- No duplicate `clerkId` values should be possible in the users table

## Breaking Changes

- [x] No breaking changes

## Documentation

- Linear ticket: [ENG-1063](https://linear.app/florafauna/issue/ENG-1063/convex-user-fixes)
- Figma designs: N/A
- Architecture doc: N/A
- Related PRs: None

## Deployment Notes

- [x] No special deployment requirements
- [ ] Requires environment variable changes
- [ ] Requires database migration
- [ ] Requires manual intervention

Note: The Convex schema changes will be automatically applied when deployed.

## Checklist

- [x] PR title follows conventional commits format (e.g., `feat:`, `fix:`, `chore:`)
- [x] Code follows project style guidelines
- [ ] Tests have been added/updated for changes (No tests required for database schema changes)
- [ ] Documentation has been updated if needed (No documentation changes required)
- [x] PR has been self-reviewed
- [x] No console.log or debug statements left in code
- [x] Performance implications have been considered
- [x] Security implications have been considered

## PR Labels

- [x] `review now!` - Ready for review
- [ ] `making changes` - Addressing feedback
- [ ] `waiting on another pr` - Has dependencies
- [ ] `changes requested` - Reviewer requested changes

## Changelog Entry

[Database] Fix user email update to properly maintain sanitizedEmail field and add clerkId index for data integrity

## Additional Context

This is a critical data integrity fix that ensures user email updates properly maintain all related fields. The addition of the clerkId index will improve query performance and prevent potential duplicate entries.

---

_[Engineering Standards](https://www.notion.so/Engineering-Standards-223b3414c1b580eb9ceade3d05649e9e?source=copy_link)_