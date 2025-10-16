---
date: 2025-10-16T14:40:43-04:00
researcher: matan
git_commit: e92ba42a902ce1151a284473f1c3ceea05dd6b17
branch: eng-1057-video-img
repository: eng-1057-video
topic: "Video Thumbnail Loading Behavior - Hover Dependency Issue"
tags: [research, codebase, video-thumbnails, lazy-loading, r3f-canvas, ENG-1057]
status: complete
last_updated: 2025-10-16
last_updated_by: matan
---

# Research: Video Thumbnail Loading Behavior - Hover Dependency Issue

**Date**: 2025-10-16T14:40:43-04:00
**Researcher**: matan
**Git Commit**: e92ba42a902ce1151a284473f1c3ceea05dd6b17
**Branch**: eng-1057-video-img
**Repository**: eng-1057-video

## Research Question

Why does the video thumbnail only load on hover instead of loading immediately when the component mounts, and how is the current implementation structured?

## Summary

The video thumbnail system is designed to load thumbnails immediately and independently from video loading, but the current implementation shows thumbnails only appearing on hover. The thumbnail loading effect in VideoMaterial depends solely on `videoUrl` being present and should trigger on mount. The implementation uses a two-tier approach: thumbnails should load unconditionally while videos load only after user interaction (hover/selection). The thumbnail URL is generated using ImageKit's `/ik-thumbnail.jpg` suffix pattern, and the texture is managed separately from the video texture.

## Detailed Findings

### Video Material Component - Thumbnail Loading

The thumbnail loading system is implemented in the VideoMaterial component ([src/components/r3f/blocks/video/video-material.tsx:78-126](src/components/r3f/blocks/video/video-material.tsx#L78-126)):

**State Management:**
- Thumbnail texture state declared at line 42: `const [thumbnailTexture, setThumbnailTexture] = useState<THREE.Texture | null>(null);`
- Texture loader reference at line 43: `const thumbnailLoaderRef = useRef<THREE.TextureLoader | null>(null);`
- Loader initialized in effect at lines 71-76

**Loading Effect (lines 78-126):**
```typescript
useEffect(() => {
  if (!videoUrl || !thumbnailLoaderRef.current) {
    setThumbnailTexture(null);
    return;
  }
  // Generate thumbnail URL and load texture...
}, [videoUrl]);  // Only depends on videoUrl
```

Key observation: This effect depends ONLY on `[videoUrl]` - it is NOT tied to `shouldLoadVideo`, `hasInteracted`, or any interaction state. It should trigger immediately when `videoUrl` is provided.

### Video Block Result - Lazy Loading Control

The VideoBlockResultR3F component controls when content loads ([src/components/r3f/video-block-result.tsx:48-68](src/components/r3f/video-block-result.tsx#L48-68)):

**State Variables:**
- `isHovered` (line 48): Tracks current hover state
- `hasInteracted` (line 49): Sticky flag that becomes true on first interaction
- `selected` (prop): Node selection state from parent

**Loading Decision (lines 67-68):**
```typescript
const shouldPlay = selected || isHovered;
const shouldLoad = hasInteracted || selected || isHovered;
```

**Prop Passing (line 126):**
```typescript
<VideoMaterial
  videoUrl={videoUrl}
  shouldLoadVideo={shouldLoad}
  shouldPlay={shouldPlay}
  // ... other props
/>
```

The `videoUrl` is passed directly without any conditional logic - it's always passed if available.

### Thumbnail URL Generation

The helper function generates ImageKit thumbnail URLs ([src/lib/videos/helpers.ts:196-214](src/lib/videos/helpers.ts#L196-214)):

```typescript
export function getVideoThumbnailUrl(videoUrl: string | null | undefined): string | null {
  if (!videoUrl) return null;
  if (!videoUrl.startsWith("https://ik.imagekit.io/")) return null;

  const isVideo = videoUrl.match(/\.(mp4|webm|mov)(\?|$)/i);
  if (!isVideo) return null;

  const cleanUrl = videoUrl.split("?")[0];
  return `${cleanUrl}/ik-thumbnail.jpg`;
}
```

### Display Logic and Texture Selection

The VideoMaterial render logic ([src/components/r3f/blocks/video/video-material.tsx:293-305](src/components/r3f/blocks/video/video-material.tsx#L293-305)):

**Skeleton Fallback (lines 293-302):**
```typescript
if (!videoUrl || (isLoading && !thumbnailTexture) || (!texture && !thumbnailTexture)) {
  return <SkeletonMaterial />;
}
```

**Texture Priority (line 305):**
```typescript
const displayTexture = texture || thumbnailTexture;
```

The display logic uses video texture if available, otherwise falls back to thumbnail texture.

### Thumbnail Disposal on Video Load

When video texture loads successfully ([src/components/r3f/blocks/video/video-material.tsx:148-152, 200-204](src/components/r3f/blocks/video/video-material.tsx#L148-152)):

```typescript
// Dispose thumbnail when video loads
if (thumbnailTexture) {
  thumbnailTexture.dispose();
  setThumbnailTexture(null);
}
```

## Code References

- `src/components/r3f/blocks/video/video-material.tsx:78` - Thumbnail loading effect start
- `src/components/r3f/blocks/video/video-material.tsx:86` - Thumbnail URL generation call
- `src/components/r3f/blocks/video/video-material.tsx:95` - TextureLoader.load() for thumbnail
- `src/components/r3f/blocks/video/video-material.tsx:107` - Setting thumbnail texture state
- `src/components/r3f/blocks/video/video-material.tsx:305` - Display texture selection logic
- `src/components/r3f/video-block-result.tsx:68` - shouldLoad calculation
- `src/components/r3f/video-block-result.tsx:126` - VideoMaterial prop passing
- `src/lib/videos/helpers.ts:213` - ImageKit thumbnail suffix appending

## Architecture Documentation

### Two-Tier Loading Pattern

The system implements a progressive loading strategy:

1. **Immediate Tier (Thumbnails):**
   - Loads when `videoUrl` prop is available
   - No interaction required
   - Uses THREE.TextureLoader directly
   - Lightweight JPEG image (~50-200KB)

2. **Lazy Tier (Videos):**
   - Loads only when `shouldLoadVideo=true`
   - Requires user interaction (hover/selection)
   - Uses VideoTextureCoordinator with Web Workers
   - Heavy video files (multiple MB)

### State Flow

1. **Component Mount:**
   - VideoBlockResultR3F receives `videoUrl` from props
   - Passes `videoUrl` to VideoMaterial (always, not conditional)
   - VideoMaterial thumbnail effect triggers on `videoUrl` change

2. **Thumbnail Loading:**
   - Effect at line 78 runs when `videoUrl` is set
   - Generates thumbnail URL via helper function
   - Loads texture via THREE.TextureLoader
   - Sets `thumbnailTexture` state when loaded

3. **User Interaction:**
   - Hover triggers `isHovered=true` and `hasInteracted=true`
   - `shouldLoad` becomes `true`
   - Video loading effect triggers
   - Video loads in background

4. **Texture Transition:**
   - Video texture loads and sets `texture` state
   - Thumbnail disposed to free memory
   - Display switches from thumbnail to video

### Current Behavior vs Expected Behavior

**Expected:**
- Thumbnail loads immediately on component mount
- Video waits for user interaction
- Smooth transition from thumbnail to video

**Observed (per user feedback):**
- Thumbnail only appears on hover
- Both thumbnail and video seem tied to interaction
- No pre-hover visual content

## Historical Context (from thoughts/)

### Implementation Plans

From `thoughts/shared/plans/2025-10-16-ENG-1057-video-thumbnails.md`:
- Phase 2 specifically implements "Thumbnail texture loading in VideoMaterial component using Three.js TextureLoader"
- Design states thumbnails should "load immediately using off-thread image loader"
- Thumbnails are meant to "replace skeleton placeholder before interaction"

### Research Documents

From `thoughts/shared/research/2025-10-15-ENG-1057-video-thumbnail-loading-issue.md`:
- Documents investigation into why thumbnails weren't loading in PR 2075
- Identified that thumbnail loading was initially missing from implementation

From `thoughts/shared/research/2025-10-16-ENG-1057-video-thumbnails-imagekit.md`:
- Confirms ImageKit supports video thumbnails via `/ik-thumbnail.jpg` pattern
- Documents that no server-side changes needed

## Related Research

- `thoughts/shared/research/2025-10-14-ENG-1057-video-loading-r3f-canvas.md` - Initial video loading investigation
- `thoughts/shared/plans/2025-01-14-ENG-1057-video-lazy-loading.md` - Original lazy loading plan (PR 2075)
- `thoughts/shared/prs/2075_description.md` - PR implementing video lazy loading

## Open Questions

1. **Is videoUrl being passed conditionally higher up the component tree?** The VideoBlockResultR3F always passes videoUrl if available, but the parent component might be controlling when videoUrl is provided.

2. **Is there a race condition in the thumbnail loading effect?** The effect checks for `thumbnailLoaderRef.current` which is set in a separate effect - could there be a timing issue?

3. **Is the thumbnail URL generation failing for the specific videos being tested?** The helper returns null for non-ImageKit or non-video URLs.

4. **Is there caching or memoization preventing the effect from running?** The effect dependency array only includes `[videoUrl]`, so it should run when the URL changes.

5. **Could the issue be in the parent component's data flow?** Need to trace how `videoUrl` flows from the node data to VideoBlockResultR3F component.

## Follow-up Research [2025-10-16T14:43:00-04:00]

### Investigation of Parent Component Usage

Examined how VideoBlockResultR3F is used in parent components to verify videoUrl passing:

**In video-block.tsx (line 643-644):**
```typescript
videoUrl ? (
  <VideoBlockResultR3F
    nodeId={id}
    videoUrl={videoUrl}
```

**In static-video-block-r3f.tsx (line 210-211):**
```typescript
{videoUrl && (
  <VideoBlockResultR3F
    nodeId={id}
```

Both parent components conditionally render VideoBlockResultR3F only when videoUrl exists, but once rendered, the videoUrl is always passed. This confirms that the issue is not with conditional videoUrl passing at the component level.

### Current Implementation Status

Based on the code analysis, the thumbnail system is fully implemented with:

1. **Thumbnail URL generation** working via `getVideoThumbnailUrl()` helper
2. **Immediate loading trigger** in the effect that depends only on `[videoUrl]`
3. **Proper display logic** that shows thumbnail when available
4. **Memory cleanup** when video texture loads

### Potential Root Cause

Since the implementation appears correct but the behavior shows thumbnails only loading on hover, the most likely issues are:

1. **Timing Issue with TextureLoader Initialization**: The thumbnailLoaderRef is initialized in a separate effect (lines 71-76) from the loading effect (lines 78-126). If the loading effect runs before initialization completes, it would exit early at line 80.

2. **URL Format Issue**: The videos might not be from ImageKit or might not match the expected format (.mp4, .webm, .mov), causing `getVideoThumbnailUrl()` to return null.

3. **Network/CORS Issue**: The thumbnail might be failing to load due to network issues, but only the warning is logged (lines 111-117) without updating UI state.

### Recommended Debugging Steps

1. Add console logging at key points:
   - After thumbnail URL generation (line 86)
   - In the success callback (line 97)
   - In the error callback (line 111)

2. Check browser DevTools Network tab for:
   - Whether thumbnail requests are being made
   - Response status codes
   - CORS headers

3. Verify the video URLs are ImageKit URLs with proper format

4. Consider combining the TextureLoader initialization into the same effect as the loading logic to eliminate potential race conditions