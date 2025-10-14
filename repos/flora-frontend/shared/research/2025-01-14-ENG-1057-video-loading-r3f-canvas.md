# ENG-1057: Video Loading Analysis in R3F Canvas

## Ticket Summary

**ENG-1057**: "Video slow loading, video load failed on many canvases"

## Problem Statement

Videos in the r3f canvas are causing performance issues when multiple video blocks are present. The application becomes slow and unresponsive, with high bandwidth consumption on page load.

## Current Behavior Analysis

### Video Loading Architecture

#### 1. Component Hierarchy

```
VideoBlockResultR3F
  └── VideoMaterial
       ├── useBackgroundVideoTexture (fallback)
       └── VideoTextureCoordinator (primary)
            └── Web Worker or Main Thread
```

#### 2. Key Files Identified

**Video Block Components:**

- `/src/components/r3f/video-block-result.tsx` - Main video result renderer

  - Line 66: Play trigger logic (`selected || isHovered`)
  - Renders VideoMaterial inside Suspense boundary
  - Manages hover and selection states

- `/src/components/r3f/blocks/video/video-material.tsx` - Video material component

  - Lines 62-175: Main loading effect that triggers on mount
  - Attempts Web Worker loading first, falls back to main thread
  - Creates THREE.VideoTexture for WebGL rendering

- `/src/components/r3f/static-video-block-r3f.tsx` - Static video block variant
  - Similar pattern to VideoBlockResultR3F
  - Uses same VideoMaterial component

**Video Loading Infrastructure:**

- `/src/lib/videos/video-texture-loader.ts` - VideoTextureCoordinator class

  - Line 98: Constructor immediately calls `preloadWorker()`
  - Line 523: Main thread fallback uses `preload="auto"`
  - Manages Web Worker lifecycle and fallback logic
  - Caches loaded videos by blockId

- `/public/workers/video-processor.worker.js` - Web Worker implementation
  - Line 94: Fetches entire video blob on load request
  - Processes video in background thread
  - Returns blob URL for main thread consumption

**Supporting Hooks:**

- `/src/components/r3f/hooks/use-bg-video-texture-loader.ts` - Background loading hook
  - Used as fallback when Web Worker unavailable
  - Creates video element directly on main thread

### Current Loading Behavior

1. **On Component Mount:**

   - VideoMaterial component mounts when VideoBlockResultR3F renders
   - Immediately starts loading video via `useEffect` (line 62)
   - VideoTextureCoordinator preloads Web Worker on first instantiation

2. **Loading Process:**

   ```javascript
   // Current flow in VideoMaterial
   useEffect(() => {
     if (!useWorker) {
       // Use main thread fallback
       return;
     }
     // Immediately start loading
     loadVideo();
   }, [videoUrl, ...deps]);
   ```

3. **Video Element Creation:**

   - Web Worker path: Downloads entire video → creates blob URL → video element
   - Main thread path: Creates video element with `preload="auto"` → downloads entire file

4. **Playback Control:**
   - Correctly implemented: Videos only _play_ on hover/selection
   - Problem: Videos _load_ immediately regardless of interaction

### Performance Impact

**Observed Issues:**

1. **Bandwidth Consumption:** All videos download simultaneously on page load
2. **Memory Usage:** All video data held in memory even if never played
3. **CPU Usage:** Multiple video decoders initialized simultaneously
4. **Network Congestion:** Parallel video downloads compete for bandwidth

**Example Scenario:**

- Canvas with 10 video blocks (10MB each)
- Current: 100MB downloaded immediately on page load
- Expected: 0MB initially, 10MB per video on interaction

### Existing Patterns Found

1. **Lazy Loading Context** (`/src/components/r3f/lazy-media-loader.tsx`)

   - Exists but not used by video blocks
   - Implements viewport-based loading for other media types

2. **Hover-based Playback**

   - Already implemented correctly
   - Videos pause when not hovered/selected
   - Good UX pattern to extend for loading

3. **Interaction Tracking**
   - `useBlockEvents` hook handles hover/selection
   - Can be leveraged for lazy loading triggers

## Root Cause

The root cause is that video loading is **decoupled from user interaction**. While playback correctly waits for interaction, the expensive download operation happens immediately on mount.

Key problematic patterns:

1. `VideoTextureCoordinator` constructor preloads worker
2. `VideoMaterial` effect runs on mount with videoUrl
3. Main thread fallback uses `preload="auto"`
4. No conditional loading based on interaction state

## Solution Approach

### Recommended Fix: Interaction-Based Lazy Loading

Add a simple interaction tracking layer that delays video loading until first hover or selection:

1. **Track Interaction State:**

   ```typescript
   const [hasInteracted, setHasInteracted] = useState(false);
   const shouldLoad = hasInteracted || selected || isHovered;
   ```

2. **Conditional URL Passing:**

   ```typescript
   <VideoMaterial
     videoUrl={shouldLoad ? videoUrl : undefined}
     // ... other props
   />
   ```

3. **Handle Undefined URLs:**
   - Update VideoMaterial to gracefully handle undefined URLs
   - Show skeleton state when URL is undefined
   - Start loading when URL becomes defined

### Why This Approach?

**Advantages:**

- Minimal code changes (4 files)
- Preserves existing architecture
- No breaking changes
- Immediate performance improvement

**Alternatives Considered:**

1. **Viewport-based loading:** More complex, requires scroll tracking
2. **Progressive loading:** Requires video streaming infrastructure
3. **Thumbnail preview:** Additional asset generation needed
4. **Manual load button:** Poor UX

## Expected Improvements

### Performance Metrics

- **Initial Load Time:** 80-90% reduction with multiple videos
- **Bandwidth Usage:** Only loaded videos consume bandwidth
- **Memory Usage:** Gradual increase vs immediate spike
- **Time to Interactive:** Significantly faster

### User Experience

- Videos load on hover (natural interaction)
- No visual difference for single videos
- Smooth playback once loaded
- No re-downloading on subsequent hovers

## Implementation Checklist

- [x] Identify all video loading components
- [x] Trace loading flow from mount to playback
- [x] Find interaction event handlers
- [x] Design lazy loading approach
- [x] Implement interaction tracking
- [x] Update VideoMaterial for undefined URLs
- [x] Apply to static video blocks
- [x] Optimize worker initialization

## References

- Ticket: ENG-1057
- Implementation Plan: `thoughts/shared/plans/2025-01-14-ENG-1057-video-lazy-loading.md`
- Similar pattern: `src/components/r3f/lazy-media-loader.tsx` (viewport-based)
- Interaction hooks: `src/components/r3f/use-block-events.ts`
