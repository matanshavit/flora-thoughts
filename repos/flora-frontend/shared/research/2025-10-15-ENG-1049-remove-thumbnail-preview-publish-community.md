---
date: 2025-10-15T11:39:05-04:00
researcher: Matan Shavit
git_commit: 0a212379dd23cade8bb7211802c1b3c8cf3298c4
branch: eng-1049
repository: eng-1049
topic: "ENG-1049: Remove thumbnail previews from Publish to Community"
tags: [research, codebase, community-publishing, thumbnails, publish-dialog, convex]
status: complete
last_updated: 2025-10-15
last_updated_by: Matan Shavit
---

# Research: ENG-1049 Remove Thumbnail Previews from Publish to Community

**Date**: 2025-10-15T11:39:05-04:00
**Researcher**: Matan Shavit
**Git Commit**: 0a212379dd23cade8bb7211802c1b3c8cf3298c4
**Branch**: eng-1049
**Repository**: eng-1049

## Research Question

Research all the relevant files for removing thumbnail previews from the "Publish to Community" feature, allowing the default thumbnail to be picked automatically without the option to change it. This is a simplification effort to avoid showing blank preview thumbnails while waiting for a broader sharing overhaul.

## Summary

The thumbnail preview and selection functionality in the "Publish to Community" feature consists of several interconnected components:

1. **Main UI Component**: `publish-to-community-dialog.tsx` contains all thumbnail selection UI, including the preview grid, upload capability, and selection state management
2. **Backend Storage**: Thumbnails are stored in the `previewUrl` field in both Convex (`communityProjects` table) and PostgreSQL schemas
3. **State Management**: Local React state (`useState`) handles thumbnail selection - no global store involvement
4. **Display Components**: Multiple components display thumbnails (community cards, grids, sliders) that will continue to work with automatically selected thumbnails

The removal of thumbnail selection UI will require:
- Modifying the dialog to remove the thumbnail selection grid and upload functionality
- Updating the submission logic to automatically select the first available thumbnail
- Ensuring the backend mutations continue to receive a valid `previewUrl`
- No changes needed to display components or database schemas

## Detailed Findings

### Primary Implementation File - Publishing Dialog

**`src/components/projects/publish-to-community-dialog.tsx`**
- **Lines 58-62**: State management for thumbnail selection
  - `uploadedImageUrl` - stores custom uploaded image URL
  - `uploading` - upload in progress flag
  - `selectedPreview` - currently selected thumbnail URL
- **Lines 32-46**: Memoized selector extracting project outputs from ProjectStore
- **Lines 98**: `possiblePreviewUrls` array combining project outputs and uploaded images
- **Lines 100-113**: `handleUploadPreview` function for custom image uploads
- **Lines 116-119**: Fallback logic using first available thumbnail if none selected
- **Lines 173-233**: UI components to remove:
  - Preview mockup display (lines 173-191)
  - Thumbnail selection grid (lines 193-213)
  - Upload button (lines 214-233)

### Backend Mutations (Convex)

**`convex/communityProjects/mutations.ts`**
- **Lines 13-77**: `saveCommunityProject` mutation
  - `previewUrl` is a required string parameter (line 16)
  - Updates existing projects (lines 37-49) or creates new ones (lines 50-68)
  - No server-side thumbnail generation - expects URL from client

**`convex/communityProjects/queries.ts`**
- **Lines 43-60**: `getCommunityProjectForProject` query fetches existing community project with `previewUrl`

### Database Schemas

**Convex Schema** (`convex/modelValidators.ts:337-353`)
- `previewUrl: v.optional(v.string())` field in `communityProjectsValidator`

**PostgreSQL Schema** (`src/db/schema.ts:625-673`)
- `previewUrl: text("preview_url")` column in `communityProjects` table

### Automatic Thumbnail Generation Logic

**`convex/generationHistory/helpers.ts:130-138`**
- When a generation completes with `COMPLETED` status and has an image/video URL
- Calls `getOutputThumnailUrl()` to create thumbnail URL
- Updates project's `pictureUrl` field (NOT the community project's `previewUrl`)

**`shared/files/thumbnails.ts:7-12`**
- `getOutputThumnailUrl()` helper creates ImageKit thumbnail URLs
- Adds transformation parameters for resizing
- For videos (.mp4), appends `/ik-thumbnail.jpg` for frame extraction

**`convex/projects/helpers.ts:256-279`**
- `_updateProjectPicture()` manages project thumbnails (separate from community thumbnails)
- Priority system: manual thumbnails override automatic ones

### Display Components (No Changes Needed)

These components will continue to display thumbnails from the `previewUrl` field:

**Community Project Cards**
- `src/components/community/community-project/community-project-card.tsx:80-104`
- Displays thumbnail using ThumbnailImage component
- 346px × 192px dimensions with hover effects

**Community Project Grid**
- `src/components/community/community-project/community-project-grid.tsx`
- Grid layout for project cards

**Community Project Slider**
- `src/components/community/community-project/community-projects-slider.tsx`
- Carousel view of projects

**Template Blocks**
- `src/components/community/template-list/template-block.tsx:15-23`
- Uses `getOutputThumnailUrl()` helper with MEDIUM size

### Supporting UI Components

**`src/components/ui/thumbnail-image.tsx`**
- Reusable thumbnail component with loading states and retry logic
- Will continue to work with automatically selected thumbnails

### State Management Analysis

**Current Implementation**:
- Local React state in dialog component
- No global state management (Zustand/Redux)
- Selection state: `useState<string | null>(null)`
- Available thumbnails derived from ProjectStore nodes

**After Removal**:
- No selection state needed
- Automatic selection of first available thumbnail
- Continue deriving thumbnails from ProjectStore

### Parent Components and Entry Points

**`src/components/projects/share-project-dialog.tsx`**
- Parent dialog that embeds PublishToCommunityDialog
- Shows published status and sharing options

**`src/components/projects/project-context-menu.tsx`**
- Context menu entry point (currently commented out at lines 22, 57)

### API and Data Flow

**Current Flow**:
1. User opens dialog → queries existing community project
2. Extracts thumbnails from project outputs via ProjectStore
3. User selects thumbnail or uploads custom image
4. Submits with `previewUrl` to `saveCommunityProject` mutation
5. Mutation persists to Convex database

**Simplified Flow**:
1. User opens dialog → queries existing community project
2. Extracts thumbnails from project outputs via ProjectStore
3. Automatically selects first available thumbnail
4. Submits with `previewUrl` to `saveCommunityProject` mutation
5. Mutation persists to Convex database

## Code References

### Files to Modify
- `src/components/projects/publish-to-community-dialog.tsx` - Remove thumbnail selection UI and state

### Key Functions
- `src/components/projects/publish-to-community-dialog.tsx:116` - Fallback logic already exists for auto-selection
- `shared/files/thumbnails.ts:7-12` - Thumbnail URL generation helper
- `convex/communityProjects/mutations.ts:13-77` - Backend mutation expecting `previewUrl`

### Constants
- `shared/files/thumbnails.ts:1-5` - Thumbnail size constants (SMALL=96, MEDIUM=350, LARGE=500)

## Architecture Documentation

### Current Patterns
- **Separation of Concerns**: Thumbnail selection is isolated in the dialog component
- **State Management**: Local component state for UI interactions
- **Data Derivation**: Thumbnails extracted from project outputs via ProjectStore
- **Fallback Logic**: Already implements first-thumbnail fallback

### Integration Points
- **ProjectStore**: Source of project outputs for thumbnail candidates
- **Convex Mutations**: Backend expects `previewUrl` parameter
- **Display Components**: Read `previewUrl` from community project data
- **No Coupling**: Display components are decoupled from selection logic

## Historical Context (from thoughts/)

- **ENG-1049 Ticket**: "Publish to community: filter by empty (thumbnail)" - Current issue addressing empty thumbnails
- **No Historical Documentation**: No prior documentation found about thumbnail feature development or decisions
- **Related Research**: ENG-1082 research covers project sharing architecture but not thumbnail specifics

## Related Research

- `thoughts/shared/research/2025-10-15-ENG-1082-view-only-project-sharing-redirect-error.md` - Project sharing architecture
- `thoughts/shared/research/research-2025-10-13-eng-976-guest-permissions.md` - Guest permissions and view-only mode

## Open Questions

1. **Default Thumbnail Selection Algorithm**: Should we use the first output, most recent, or implement smart selection?
2. **Empty State Handling**: What happens if a project has no outputs to use as thumbnails?
3. **Existing Projects**: How to handle projects already published with custom thumbnails?
4. **User Communication**: Should we notify users about the simplified thumbnail behavior?

## Implementation Recommendations

Based on the research, the implementation should focus on:

1. **Minimal Changes**: Only modify `publish-to-community-dialog.tsx`
2. **Preserve Backend**: Keep `previewUrl` field and mutations unchanged
3. **Automatic Selection**: Use existing fallback logic (first available thumbnail)
4. **Remove UI Elements**:
   - Lines 58-59: Remove upload-related state
   - Lines 62: Remove selection state
   - Lines 100-113: Remove upload handler
   - Lines 173-233: Remove entire preview/selection UI section
5. **Simplify Logic**:
   - Line 116: Always use `possiblePreviewUrls[0]` without user selection
   - Remove `selectedPreview` references throughout

This approach ensures backward compatibility while simplifying the user experience and preventing blank thumbnails in the community view.