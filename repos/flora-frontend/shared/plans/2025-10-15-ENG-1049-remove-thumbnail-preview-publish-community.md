# Remove Thumbnail Preview from Publish to Community Implementation Plan

## Overview

Remove the thumbnail selection UI from the "Publish to Community" dialog to simplify the user experience and avoid showing blank preview thumbnails. The system will automatically select the first available thumbnail from project outputs without giving users the option to change it.

## Current State Analysis

The current implementation in `src/components/projects/publish-to-community-dialog.tsx` provides:
- A preview mockup showing how the thumbnail will appear (lines 173-191)
- A grid of selectable thumbnails from project outputs (lines 192-213)
- An upload button for custom thumbnails (lines 214-233)
- State management for selection (`selectedPreview`) and uploads (`uploadedImageUrl`, `uploading`)
- Existing fallback logic that already selects the first thumbnail if none chosen (line 116)

### Key Discoveries:
- The fallback behavior we want already exists at line 116: `selectedPreview ?? possiblePreviewUrls[0]`
- The dialog is only used in one place: `share-project-dialog.tsx`
- No other components depend on the thumbnail selection functionality
- Backend mutations expect a `previewUrl` string and will continue to work unchanged

## Desired End State

After implementation:
- The dialog will show only the title and description form fields
- Thumbnails will be automatically selected from the first available project output
- No visual preview or selection UI will be shown
- The backend will continue to receive a valid `previewUrl` without changes
- Existing published projects will retain their previously selected thumbnails

### Success Criteria:

#### Automated Verification:
- [ ] Type checking passes: `pnpm typecheck`
- [ ] Build completes successfully: `pnpm build`
- [ ] No linting errors: `pnpm lint`

#### Manual Verification:
- [ ] Dialog opens without thumbnail selection UI
- [ ] Publishing works with automatic thumbnail selection
- [ ] Projects with outputs publish successfully
- [ ] Error shown when project has no outputs
- [ ] Existing published projects retain their thumbnails
- [ ] Unpublishing continues to work

## What We're NOT Doing

- NOT modifying the backend mutations or database schema
- NOT changing how thumbnails are displayed in the community view
- NOT modifying the thumbnail generation logic (`getOutputThumnailUrl`)
- NOT changing the parent `ShareProjectDialog` component
- NOT implementing smart thumbnail selection (just using first available)
- NOT handling the broader sharing overhaul mentioned in the ticket

## Implementation Approach

This is primarily a code removal task. We'll remove the UI components for thumbnail selection while preserving the existing automatic selection logic. The changes are confined to a single file, making this a low-risk modification.

## Phase 1: Remove Thumbnail Selection UI

### Overview
Remove all visual components related to thumbnail preview and selection from the dialog.

### Changes Required:

#### 1. Remove Preview Mockup and Thumbnail Grid
**File**: `src/components/projects/publish-to-community-dialog.tsx`
**Changes**: Delete lines 173-233 (entire preview section)

Remove this entire block:
```tsx
// Lines 173-233 - DELETE ALL OF THIS
<div className="relative mb-4 w-full">
  <Image
    className="mb-4 w-full object-contain"
    src="/editor/share-community-preview.png"
    width={604}
    height={295}
    alt="community page preview"
  />
  {selectedPreview && (
    <ThumbnailImage
      containerClassName="absolute left-[28%] top-[21%] aspect-square w-[21.5%] overflow-hidden rounded-radius-250 object-cover"
      className="aspect-square h-full w-full object-cover"
      src={getOutputThumnailUrl(selectedPreview)}
      alt="output preview"
      width={200}
      height={200}
    />
  )}
</div>
<div className="flex flex-wrap justify-center gap-2">
  {possiblePreviewUrls.map((url, index) => (
    <Button
      key={index}
      variant="plain"
      className={cn(
        "flex aspect-square overflow-hidden rounded-radius-250 border border-border-transparent bg-background-transparent-black-secondary p-0",
        {
          "border-2 border-border-transparent-tertiary": selectedPreview === url,
        },
      )}
      onClick={() => {
        setSelectedPreview(url);
      }}
    >
      <ThumbnailImage
        src={getOutputThumnailUrl(url)}
        alt="output preview"
        className="h-full w-full object-cover"
      />
    </Button>
  ))}
  <>
    <Button
      variant="plain"
      className={cn(
        "flex aspect-square w-[72px] rounded-radius-250 border border-border-transparent bg-background-transparent-black-secondary",
      )}
      onClick={() => inputRef && inputRef.current && inputRef.current.click()}
      disabled={uploading}
    >
      {uploading ? <Loader size={16} /> : <UploadIcon size={16} />}
    </Button>
    <input
      hidden
      ref={inputRef}
      type="file"
      accept="image/*"
      onChange={handleUploadPreview}
    />
  </>
</div>
```

### Success Criteria:

#### Automated Verification:
- [x] Component compiles without errors
- [x] No TypeScript errors related to removed UI

#### Manual Verification:
- [ ] Dialog opens without preview mockup image
- [ ] No thumbnail selection buttons visible
- [ ] No upload button visible

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation from the human that the manual testing was successful before proceeding to the next phase.

---

## Phase 2: Simplify Thumbnail Logic

### Overview
Remove state management for thumbnail selection and simplify the automatic selection logic.

### Changes Required:

#### 1. Remove Thumbnail Selection State Variables
**File**: `src/components/projects/publish-to-community-dialog.tsx`
**Changes**: Remove state declarations and update logic

Remove these state variables (lines 57-59, 62):
```tsx
// Line 57 - DELETE
const inputRef = useRef<HTMLInputElement>(null);
// Line 58 - DELETE
const [uploadedImageUrl, setUploadedImageUrl] = useState<string | null>(null);
// Line 59 - DELETE
const [uploading, setUploading] = useState(false);
// Line 62 - DELETE
const [selectedPreview, setSelectedPreview] = useState<string | null>(null);
```

#### 2. Update possiblePreviewUrls Array
**File**: `src/components/projects/publish-to-community-dialog.tsx`
**Changes**: Simplify array to only include project outputs

Update line 98 from:
```tsx
const possiblePreviewUrls = [...projectOutputs, uploadedImageUrl].filter(Boolean);
```

To:
```tsx
const possiblePreviewUrls = projectOutputs.filter(Boolean);
```

#### 3. Remove Upload Handler Function
**File**: `src/components/projects/publish-to-community-dialog.tsx`
**Changes**: Delete entire function (lines 100-113)

Remove:
```tsx
// Lines 100-113 - DELETE
const handleUploadPreview = async (e: ChangeEvent<HTMLInputElement>): Promise<void> => {
  const file = e.target.files ? e.target.files[0] : null;
  if (!file) {
    return;
  }

  setUploading(true);
  const { error, url: uploadedImageUrl } = await uploadImageClient(file);
  setUploading(false);
  if (error) {
    return showError(error);
  }
  setUploadedImageUrl(uploadedImageUrl!);
};
```

#### 4. Update Submit Handler
**File**: `src/components/projects/publish-to-community-dialog.tsx`
**Changes**: Simplify preview URL selection

Update line 116 from:
```tsx
const previewUrl = selectedPreview ?? possiblePreviewUrls[0];
```

To:
```tsx
const previewUrl = possiblePreviewUrls[0];
```

#### 5. Update useEffect Hook
**File**: `src/components/projects/publish-to-community-dialog.tsx`
**Changes**: Remove selectedPreview sync

Update lines 86-96 from:
```tsx
useEffect(() => {
  setSelectedPreview(communityProject?.previewUrl ?? null);
  form.reset(
    communityProject
      ? {
          title: communityProject.title,
          description: communityProject.description ?? undefined,
        }
      : { title: "", description: "" },
  );
}, [communityProject, form]);
```

To:
```tsx
useEffect(() => {
  form.reset(
    communityProject
      ? {
          title: communityProject.title,
          description: communityProject.description ?? undefined,
        }
      : { title: "", description: "" },
  );
}, [communityProject, form]);
```

### Success Criteria:

#### Automated Verification:
- [ ] No unused variable warnings
- [ ] TypeScript compilation succeeds
- [ ] No references to removed state variables

#### Manual Verification:
- [ ] Form still resets when community project loads
- [ ] Automatic thumbnail selection works
- [ ] Error shown when no thumbnails available

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation from the human that the manual testing was successful before proceeding to the next phase.

---

## Phase 3: Clean Up Unused Code

### Overview
Remove unused imports and clean up any remaining references.

### Changes Required:

#### 1. Clean Up Imports
**File**: `src/components/projects/publish-to-community-dialog.tsx`
**Changes**: Remove unused imports

Remove from imports:
- `UploadIcon` from lucide-react (line 11)
- `uploadImageClient` from upload helpers (line 6)
- `ThumbnailImage` component (line 15) - only if not used elsewhere
- `Image` from next/image (line 18)
- `ChangeEvent, useRef` from react (line 19)
- `cn` utility (line 8) - only if not used elsewhere

Updated imports section:
```tsx
import { useShowError } from "@/components/nodes/useShowError";
import { Button } from "@/components/ui/button";
import { Dialog, DialogContent, DialogTitle } from "@/components/ui/dialog";
import { FormInputField } from "@/components/ui/form-input-field";
import { createExpensiveSelector, ProjectState, useProjectStore } from "@/lib/projects/context";
import { zodResolver } from "@hookform/resolvers/zod";
import { useMutation, useQuery } from "convex/react";
import { GlobeIcon, Loader } from "lucide-react";

import { FloraNodeOutput, ImageUrlOutput, VideoUrlOutput } from "@/components/nodes/types";
import { Form } from "@/components/ui/form";

import { getFloraError } from "@/lib/helpers";
import { useEffect, useState } from "react";
import { useForm } from "react-hook-form";
import { z } from "zod";
import { api } from "../../../convex/_generated/api";
import { Id } from "../../../convex/_generated/dataModel";

import { getOutputThumnailUrl } from "@shared/files/thumbnails";
```

Note: Keep `getOutputThumnailUrl` import as it may still be needed for future display purposes.

### Success Criteria:

#### Automated Verification:
- [ ] No unused import warnings: `pnpm typecheck`
- [ ] Build succeeds: `pnpm build`
- [ ] Linting passes: `pnpm lint`

#### Manual Verification:
- [ ] Complete end-to-end publish flow works
- [ ] Update existing publication works
- [ ] Unpublish functionality works
- [ ] No console errors during operation

---

## Testing Strategy

### Unit Tests:
- Verify dialog renders without thumbnail UI
- Test form submission with automatic thumbnail selection
- Test error handling when no outputs available

### Integration Tests:
- Publish new project to community
- Update existing community project
- Unpublish project
- Test with projects containing various output types (images, videos)

### Manual Testing Steps:
1. Open a project with image/video outputs
2. Click "Share" → "Community" tab → "Publish to Community"
3. Verify no thumbnail selection UI appears
4. Fill in title and description
5. Click "Publish to Community"
6. Verify project appears in community with first output as thumbnail
7. Re-open dialog and verify it shows "Publish updates"
8. Test unpublishing
9. Test with a project that has no outputs (should show error)

## Performance Considerations

This change actually improves performance by:
- Reducing the number of rendered components
- Eliminating state updates for selection
- Removing image upload functionality
- Simplifying the rendering logic

## Migration Notes

No migration required. Existing community projects retain their `previewUrl` values. The change only affects the UI for future publishing actions.

## References

- Original ticket: ENG-1049 "Publish to community: filter by empty (thumbnail)"
- Research document: `thoughts/shared/research/2025-10-15-ENG-1049-remove-thumbnail-preview-publish-community.md`
- Current implementation: `src/components/projects/publish-to-community-dialog.tsx`
- Parent component: `src/components/projects/share-project-dialog.tsx`