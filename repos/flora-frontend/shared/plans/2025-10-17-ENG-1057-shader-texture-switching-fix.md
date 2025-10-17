# Shader Texture Switching Fix Implementation Plan

## Overview
Fix the video material component's texture switching logic to properly transition from thumbnail to video texture. The current implementation has a race condition and improper state initialization that prevents the shader from switching textures correctly.

## Current State Analysis

The `VideoMaterial` component at `src/components/r3f/blocks/video/video-material.tsx` manages three texture states:
- `thumbnailTexture`: The static thumbnail image texture
- `texture`: The video texture from worker/fallback loading
- `displayTexture`: What's actually rendered (initialized with thumbnailTexture's value)

### Key Discoveries:
- `displayTexture` is initialized with `thumbnailTexture`'s value at mount time (line 260), which is `null`
- The thumbnail cleanup function (line 110-112) captures stale `thumbnailTexture` from closure
- Multiple disposal points without coordination cause potential double-disposal
- React batches state updates causing render before effect execution

## Desired End State

The video material should:
1. Display the thumbnail texture immediately when available
2. Seamlessly switch to video texture when loaded
3. Properly dispose of textures without double-disposal errors
4. Handle all edge cases (unmount during load, URL changes, etc.)

### Verification:
- Thumbnail displays immediately on component mount
- Video texture replaces thumbnail when loaded (no flicker or blank frames)
- No console errors about texture disposal
- Memory properly freed (no texture leaks)

## What We're NOT Doing

- Not refactoring the entire video loading system
- Not changing the worker/fallback loading architecture
- Not modifying the shader code or uniforms
- Not changing the parent component integration
- Not altering the video playback control logic

## Implementation Approach

Fix the texture switching by:
1. Properly initializing `displayTexture` state to handle null case
2. Using refs to track texture disposal state and prevent double-disposal
3. Fixing effect dependencies to avoid stale closures
4. Ensuring atomic texture switching with proper timing

## Phase 1: Fix State Initialization and Dependencies

### Overview
Fix the `displayTexture` initialization and ensure proper dependency tracking in effects.

### Changes Required:

#### 1. Fix displayTexture State Initialization
**File**: `src/components/r3f/blocks/video/video-material.tsx`
**Changes**: Update line 260 to properly initialize displayTexture

```typescript
// Line 260 - CHANGE FROM:
const [displayTexture, setDisplayTexture] = useState(thumbnailTexture);

// CHANGE TO:
const [displayTexture, setDisplayTexture] = useState<THREE.Texture | THREE.VideoTexture | null>(null);
```

#### 2. Fix Thumbnail Effect Cleanup Dependencies
**File**: `src/components/r3f/blocks/video/video-material.tsx`
**Changes**: Use a ref to track the current thumbnail texture for cleanup

```typescript
// Add after line 75 (after thumbnailLoaderRef declaration):
const thumbnailTextureRef = useRef<THREE.Texture | null>(null);

// Update the thumbnail loading effect (lines 85-114):
useEffect(() => {
  if (!thumbnailUrl || !thumbnailLoaderRef.current) {
    setThumbnailTexture(null);
    thumbnailTextureRef.current = null;
    return;
  }

  // Load thumbnail texture
  thumbnailLoaderRef.current.load(
    thumbnailUrl,
    (loadedTexture) => {
      if (mounted.current) {
        configureTexture(loadedTexture);
        setThumbnailTexture(loadedTexture);
        thumbnailTextureRef.current = loadedTexture;
      }
    },
    undefined,
    (error) => {
      log.warn("Failed to load video thumbnail", {
        error: error instanceof Error ? error.message : String(error),
        thumbnailUrl,
      });
    }
  );

  return () => {
    // Use ref instead of state for cleanup
    if (thumbnailTextureRef.current) {
      thumbnailTextureRef.current.dispose();
      thumbnailTextureRef.current = null;
    }
  };
}, [thumbnailUrl]); // Remove thumbnailLoaderRef.current from dependencies
```

### Success Criteria:

#### Automated Verification:
- [ ] Type checking passes: `pnpm typecheck`
- [ ] Linting passes: `pnpm lint`
- [ ] Build succeeds: `pnpm build`

#### Manual Verification:
- [ ] Component mounts without errors
- [ ] No console warnings about missing dependencies
- [ ] Thumbnail cleanup works on URL change

---

## Phase 2: Implement Proper Disposal Coordination

### Overview
Add disposal tracking to prevent double-disposal of textures.

### Changes Required:

#### 1. Add Disposal Tracking Ref
**File**: `src/components/r3f/blocks/video/video-material.tsx`
**Changes**: Add a ref to track which textures have been disposed

```typescript
// Add after line 75 (after other refs):
const disposedTextures = useRef<WeakSet<THREE.Texture>>(new WeakSet());
```

#### 2. Update Display Texture Effect with Safe Disposal
**File**: `src/components/r3f/blocks/video/video-material.tsx`
**Changes**: Update the display texture switching effect (lines 261-271)

```typescript
// Replace lines 261-271 with:
useEffect(() => {
  // Handle initial load and texture switching
  if (texture) {
    setDisplayTexture(texture);

    // Safely dispose thumbnail if it exists and hasn't been disposed
    if (thumbnailTexture && !disposedTextures.current.has(thumbnailTexture)) {
      thumbnailTexture.dispose();
      disposedTextures.current.add(thumbnailTexture);
      setThumbnailTexture(null);
      thumbnailTextureRef.current = null;
    }
  } else if (thumbnailTexture) {
    setDisplayTexture(thumbnailTexture);
  } else {
    // Both are null, ensure display is null
    setDisplayTexture(null);
  }
}, [texture, thumbnailTexture]);
```

#### 3. Update Video Loading Effect Disposal
**File**: `src/components/r3f/blocks/video/video-material.tsx`
**Changes**: Update thumbnail disposal in video loading effect (lines 130-133)

```typescript
// Replace lines 130-133 with:
if (thumbnailTexture && !disposedTextures.current.has(thumbnailTexture)) {
  thumbnailTexture.dispose();
  disposedTextures.current.add(thumbnailTexture);
  setThumbnailTexture(null);
  thumbnailTextureRef.current = null;
}
```

### Success Criteria:

#### Automated Verification:
- [ ] Type checking passes: `pnpm typecheck`
- [ ] Unit tests pass: `pnpm test:unit`

#### Manual Verification:
- [ ] No double-disposal errors in console
- [ ] Textures properly disposed on unmount
- [ ] Memory profiler shows no texture leaks

---

## Phase 3: Add Missing Dependencies

### Overview
Add missing dependencies to effects to ensure proper updates.

### Changes Required:

#### 1. Fix Video Loading Effect Dependencies
**File**: `src/components/r3f/blocks/video/video-material.tsx`
**Changes**: Add missing dependencies to video loading effect (lines 221-232)

```typescript
// Update line 221-232 dependencies array:
}, [
  videoUrl,
  blockId,
  quality,
  useWorker,
  onVideoLoaded,
  fallbackTexture,
  fallbackVideo,
  fallbackLoaded,
  fallbackError,
  shouldLoadVideo,
  displayWidth,    // Add this
  displayHeight,   // Add this
  muted,          // Add this
  thumbnailTexture, // Add this
]);
```

### Success Criteria:

#### Automated Verification:
- [ ] ESLint exhaustive-deps rule passes
- [ ] Type checking passes: `pnpm typecheck`

#### Manual Verification:
- [ ] Effect re-runs when dimensions change
- [ ] Effect re-runs when muted state changes
- [ ] Proper cleanup on dependency changes

---

## Phase 4: Testing and Validation

### Overview
Comprehensive testing of the texture switching fix.

### Changes Required:

#### 1. Add Debug Logging (Temporary)
**File**: `src/components/r3f/blocks/video/video-material.tsx`
**Changes**: Add logging to verify texture switching

```typescript
// Add temporary debug logging in displayTexture effect (after line 261):
useEffect(() => {
  log.debug("Display texture effect running", {
    hasTexture: !!texture,
    hasThumbnail: !!thumbnailTexture,
    currentDisplay: !!displayTexture,
  });

  // ... rest of effect
}, [texture, thumbnailTexture]);
```

### Success Criteria:

#### Automated Verification:
- [ ] All build steps pass: `pnpm build`
- [ ] Type checking passes: `pnpm typecheck`
- [ ] Linting passes: `pnpm lint`

#### Manual Verification:
- [ ] Load a video block with thumbnail - thumbnail appears immediately
- [ ] Hover to trigger video load - video replaces thumbnail smoothly
- [ ] No texture flicker or blank frames during transition
- [ ] Check browser console - no disposal errors
- [ ] Change video URL - old textures properly disposed
- [ ] Unmount component during load - no errors
- [ ] Memory profiler - textures properly freed

**Implementation Note**: After completing this phase and all automated verification passes, remove the debug logging before committing.

---

## Testing Strategy

### Unit Tests:
- Mock THREE.TextureLoader and verify load/dispose calls
- Test state transitions: null → thumbnail → video
- Test cleanup on unmount during various loading states
- Test URL change scenarios

### Integration Tests:
- Load video block and verify texture switching visually
- Test with different video formats and sizes
- Test with missing/invalid thumbnail URLs
- Test rapid hovering/unhovering

### Manual Testing Steps:
1. Open a project with video blocks
2. Add a new video block with generation
3. Verify thumbnail loads immediately
4. Hover over video block to trigger video load
5. Verify smooth transition from thumbnail to video
6. Check console for any errors
7. Navigate away and back to test cleanup
8. Test with multiple video blocks simultaneously

## Performance Considerations

- Texture disposal is synchronous and should be fast
- WeakSet for disposal tracking has minimal memory overhead
- No additional re-renders introduced
- Refs used for tracking to avoid render cycles

## Migration Notes

This is a bug fix with no breaking changes. No migration needed for existing projects.

## References

- Original research: `thoughts/shared/research/2025-10-17-ENG-1057-shader-texture-switching-issue.md`
- Video material component: `src/components/r3f/blocks/video/video-material.tsx`
- Similar pattern in image block: `src/components/r3f/image-block-result.tsx:56-169`
- Three.js texture documentation: https://threejs.org/docs/#api/en/textures/Texture
