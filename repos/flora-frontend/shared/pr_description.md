# Pull Request Description Template

## Summary

<!-- Provide a brief, high-level summary of the changes in 1-2 sentences -->

## What problem does this PR solve?

<!-- Describe the problem, bug, or feature request that motivated this PR -->
<!-- Include Linear ticket link: [ENG-XXXX](https://linear.app/...) -->

## What does this PR do?

## <!-- List the key changes and improvements -->

-
-

## Technical Approach

<!-- Describe the technical implementation approach -->
<!-- Include any important architectural decisions or trade-offs -->

### Key Changes:

<!-- List the main files/components modified and why -->

- **Component/File**: Description of changes
- **Component/File**: Description of changes

### Performance Impact:

<!-- Describe any performance implications -->

- Initial load time:
- Runtime performance:
- Memory usage:

## Screen Recording / Screenshots

<!-- Required for UI changes - show before/after with FPS indicator and INP monitor visible -->

### Before:

<!-- Attach screenshot/recording or note "N/A - no UI changes" -->

### After:

<!-- Attach screenshot/recording or note "N/A - no UI changes" -->

## How to verify it

<!-- Provide step-by-step instructions to test the changes -->

### Automated Verification:

- [ ] Type checking passes: `pnpm typecheck`
- [ ] Build succeeds: `pnpm build`
- [ ] Unit tests pass: `pnpm test:unit`
- [ ] Linting passes: `pnpm lint`

### Manual Verification Steps:

1.
2.
3.

### Expected Behavior:

## <!-- Describe what should happen when testing -->

-

## Breaking Changes

<!-- List any breaking changes or migrations required -->

- [ ] No breaking changes
- [ ] ## Breaking changes (describe below):

## Documentation

<!-- Links to related documentation -->

- Linear ticket: [ENG-XXXX](https://linear.app/...)
- Figma designs:
- Architecture doc:
- Related PRs:

## Deployment Notes

<!-- Any special deployment considerations -->

- [ ] No special deployment requirements
- [ ] Requires environment variable changes
- [ ] Requires database migration
- [ ] Requires manual intervention

## Checklist

<!-- Ensure all items are addressed before requesting review -->

- [ ] PR title follows conventional commits format (e.g., `feat:`, `fix:`, `chore:`)
- [ ] Code follows project style guidelines
- [ ] Tests have been added/updated for changes
- [ ] Documentation has been updated if needed
- [ ] PR has been self-reviewed
- [ ] No console.log or debug statements left in code
- [ ] Performance implications have been considered
- [ ] Security implications have been considered

## PR Labels

<!-- Current PR status - update as needed -->

- [ ] `review now!` - Ready for review
- [ ] `making changes` - Addressing feedback
- [ ] `waiting on another pr` - Has dependencies
- [ ] `changes requested` - Reviewer requested changes

## Changelog Entry

<!-- One-line description for the changelog -->
<!-- Format: [Component] Brief description of user-facing change -->

## Additional Context

<!-- Any other relevant information for reviewers -->

---

_[Engineering Standards](https://www.notion.so/Engineering-Standards-223b3414c1b580eb9ceade3d05649e9e?source=copy_link)_
