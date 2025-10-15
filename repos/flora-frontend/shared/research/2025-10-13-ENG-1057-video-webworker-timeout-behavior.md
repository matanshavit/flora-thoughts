---
date: 2025-01-13T15:07:53-0400
researcher: Matan Shavit
git_commit: 7c38aa71cb4132dcab34509ef345d032bb7e2e12
branch: eng-1057-video-webworker
repository: eng-1057-video-webworker
topic: "Video WebWorker timeout after 15 seconds, main thread fallback, and confusing logging behavior"
tags:
  [
    research,
    codebase,
    video-loading,
    webworker,
    timeout,
    main-thread-fallback,
    performance,
    eng-1057,
  ]
status: complete
last_updated: 2025-01-13
last_updated_by: Matan Shavit
---

# Research: Video WebWorker Timeout and Confusing Loading Status

**Date**: 2025-01-13T15:07:53-0400
**Researcher**: Matan Shavit
**Git Commit**: 7c38aa71cb4132dcab34509ef345d032bb7e2e12
**Branch**: eng-1057-video-webworker
**Repository**: eng-1057-video-webworker

## Research Question

Why is the webworker timing out after 15 seconds and failing, why does it say the main thread is being used and why does it then say the web worker did the loading? The point of the web worker was to get the video loading off the main thread to stop blocking it.

Console logs showing the confusing behavior:

- Worker initializes successfully
- Video load times out after 15 seconds
- System falls back to main thread
- Final log says "Video loaded using Web Worker" despite main thread fallback

## Summary

The video loading system has a **race condition in the status reporting logic** that causes misleading log messages. When a worker-based load fails and falls back to main thread loading, the system can incorrectly report "Video loaded using Web Worker" because the worker reference (`this.worker`) still exists even though the actual loading was performed on the main thread. The 15-second timeout is a hardcoded value designed to prevent indefinite waiting, but the worker status check happens after the video loads successfully via the main thread fallback, leading to incorrect logging.

## Detailed Findings

### Video Loading Architecture

The video loading system (`src/lib/videos/video-texture-loader.ts`) implements a dual-path strategy:

1. **Primary Path**: Web Worker-based loading to avoid blocking the main thread
2. **Fallback Path**: Main thread loading when workers fail or timeout

The `VideoTextureCoordinator` class manages this with aggressive retry logic:

- Initial attempt + 3 retries for worker initialization
- Exponential backoff between retries (1s, 2s, 4s)
- 5-second ping timeout to verify worker health
- 15-second timeout for actual video loading

### The 15-Second Timeout

The timeout is hardcoded at line 409 in `video-texture-loader.ts`:

```typescript
const workerTimeout = setTimeout(() => {
  this.workerCallbacks.delete(blockId);
  const workerStatus = this.getWorkerStatus();
  reject(new Error(
    `Worker video load timeout after 15s. ` +
    `Block ID: ${blockId}, Video URL: ${videoUrl}, ` +
    // ... diagnostic information
  ));
}, 15000); // 15 second timeout for video loading
```

When this timeout fires:

1. The promise is rejected with a detailed error
2. The catch block at line 368 catches the error
3. If `this.worker` still exists, it retries with `loadInMainThread()` at line 373
4. The main thread successfully loads the video

### The Confusing "Web Worker" Message

The misleading log occurs due to a **timing issue** in the status check at `video-material.tsx:126-139`:

```typescript
const status = videoCoordinator.getWorkerStatus();
if (videoCoordinator.isUsingWorker()) {  // Line 127
  log.info("Video loaded using Web Worker", {...});
} else {
  log.info("Video loaded on main thread (after worker retry attempts)", {...});
}
```

The `isUsingWorker()` method simply returns `!!this.worker` (line 625). The problem sequence:

1. **T=0**: Worker starts loading video
2. **T=15000ms**: Worker times out, promise rejected
3. **T=15001ms**: Catch block executes, calls `loadInMainThread()`
4. **T=30477ms**: Main thread successfully loads video
5. **T=30478ms**: Component checks `isUsingWorker()`
6. **Result**: `this.worker` still exists (wasn't terminated), so it logs "Web Worker" incorrectly

### Why the Worker Reference Persists

The worker is only terminated in specific scenarios:

- During `handleWorkerError()` when the worker process crashes (line 289-292)
- During explicit `dispose()` or `disposeAll()` calls (lines 612-615)
- During recovery attempts after worker errors (line 290)

However, when a video load times out, this is **not a worker error**—it's a timeout in the main thread waiting for the worker. The worker itself may still be functioning, just slow. Therefore:

- The worker reference remains intact
- The `initialized` flag stays `true`
- The `isUsingWorker()` check returns `true`

### Main Thread Fallback Mechanism

When the worker fails, the system transparently falls back to main thread loading (`loadInMainThread()` at lines 507-587):

1. Uses `requestIdleCallback` when available (not Safari) with 2-second timeout
2. Falls back to `setTimeout(10ms)` in Safari
3. Creates a standard HTMLVideoElement
4. Loads the video directly without worker involvement
5. Returns the same result structure as worker loading

This fallback is **transparent to the caller**—the component doesn't know which path succeeded.

### Performance Tracking

The `VideoPerformanceMonitor` tracks load times and identifies slow loads:

- Threshold for "slow": 2000ms
- Your video took 15001ms (worker timeout) + 15476ms (main thread) = 30477ms total
- The monitor correctly logs this as a slow load
- Performance tracking happens in both paths but doesn't distinguish between them

## Code References

- `src/lib/videos/video-texture-loader.ts:409` - 15-second timeout configuration
- `src/lib/videos/video-texture-loader.ts:368-378` - Fallback logic on worker failure
- `src/lib/videos/video-texture-loader.ts:624-626` - `isUsingWorker()` status check
- `src/components/r3f/blocks/video/video-material.tsx:126-139` - Logging logic with race condition
- `src/lib/videos/video-texture-loader.ts:507-587` - Main thread loading implementation
- `public/workers/video-processor.worker.js:265-267` - Worker message handling

## Architecture Documentation

The system uses these patterns:

1. **Singleton Coordinator**: `videoCoordinator` manages all video loading
2. **Promise-based Loading**: Both paths return the same promise interface
3. **Texture Caching**: Prevents duplicate loads for the same video
4. **Worker Pooling**: Single worker handles multiple concurrent videos
5. **Graceful Degradation**: Automatic fallback maintains functionality

Current retry configuration:

- Worker initialization: 3 retries, 1s base delay, exponential backoff
- Worker recovery after error: 1 retry, 500ms delay
- No retries for individual video timeouts (immediate fallback)

## Historical Context

From the Linear ticket (ENG-1057):

- Issue identified on 2025-10-10
- Affects WebGL branch specifically (not ReactFlow implementation)
- Happens consistently, especially for uploaded videos
- ReactFlow implementation serves as performance baseline

## Related Research

- `thoughts/shared/tickets/ENG-1057.md` - Original issue ticket

## Current Implementation Behavior

The actual loading flow in your case:

1. **Worker initialized successfully** - Worker created and responding to pings
2. **Worker attempts video load** - Sends fetch request for video
3. **15-second timeout fires** - Main thread stops waiting for worker
4. **Main thread fallback** - Loads video directly on main thread
5. **Main thread succeeds after ~15 seconds** - Video loaded successfully
6. **Status check shows worker exists** - Leading to incorrect "Web Worker" message

The system is **working as designed** in terms of fallback behavior—videos do eventually load. The issue is the **misleading diagnostic logging** that makes debugging difficult.

## Key Insights

1. **The timeout is working correctly** - It prevents indefinite waiting for slow/stuck workers
2. **The fallback is working correctly** - Videos load successfully via main thread
3. **The logging is incorrect** - Status check doesn't account for which path actually succeeded
4. **Worker may not be failing** - Could just be slow network/processing, hence worker reference persists
5. **Main thread blocking is occurring** - During fallback, defeating the purpose of workers

The fundamental issue: The system can't distinguish between "worker exists and loaded the video" versus "worker exists but main thread loaded the video" because it only checks worker existence, not which path succeeded.

## Open Questions

1. Why is the worker taking >15 seconds to load videos initially?
2. Is the 15-second timeout appropriate for all video sizes/network conditions?
3. Should the worker be terminated after timeout to ensure accurate status reporting?
4. Could the worker status include which path actually loaded each video?
