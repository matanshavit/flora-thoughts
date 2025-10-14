---
date: 2025-10-14T10:17:57-04:00
researcher: Claude
git_commit: 6424810f1049f15cb6960bb270c840be3697528e
branch: eng-1057-video-webworker
repository: flora-frontend
topic: "Why Web Workers Are Failing to Load Videos Within 15-Second Timeout"
tags: [research, codebase, video-loading, web-workers, performance, timeout, ENG-1057]
status: complete
last_updated: 2025-10-14
last_updated_by: Claude
---

# Research: Why Web Workers Are Failing to Load Videos Within 15-Second Timeout

**Date**: 2025-10-14T10:17:57-04:00
**Researcher**: Claude
**Git Commit**: 6424810f1049f15cb6960bb270c840be3697528e
**Branch**: eng-1057-video-webworker
**Repository**: flora-frontend

## Research Question

Run tests from the last two commits and read logging from the commit before that described in thoughts/shared/plans/2025-01-13-ENG-1057-video-webworker-diagnostics.md to determine why web workers are failing to load videos.

## Summary

The web worker video loading system is experiencing systematic timeouts due to a combination of blocking operations in the worker pipeline, network bandwidth limitations, and memory pressure during blob creation. The 15-second timeout is being exceeded primarily during the video fetch stage (downloading the complete video file) and blob creation stage (converting the response to a blob object). Test results show that videos as small as 10MB can timeout on connections below 700 kbps, and the worker adds significant overhead compared to main thread loading, with worker loads taking up to 16x longer than main thread for the same video.

## Detailed Findings

### Worker Pipeline Architecture

The video loading worker follows a sequential pipeline with four stages ([video-processor.worker.js:77-169](public/workers/video-processor.worker.js#L77-L169)):

1. **Metadata Fetch** (HEAD request): Typically 50-200ms
2. **Video Fetch** (GET request): Variable, can exceed 10 seconds for 10MB+ videos
3. **Blob Creation** (Response to Blob): 500ms-10+ seconds depending on size
4. **Object URL Creation**: Negligible (<1ms)

The coordinator enforces a hard 15-second timeout ([video-texture-loader.ts:397-410](src/lib/videos/video-texture-loader.ts#L397-L410)) that starts when the worker is messaged and includes all processing stages.

### Test Results Analysis

#### Diagnostic Test Failures

The `video-texture-loader-diagnostics.test.ts` tests reveal critical timing patterns:

- **Stage Timing Test Failure**: Expected to find worker response logging with timing data but instead found timeout after 15 seconds, with fallback to main thread
- **Timeout Identification Test**: Timed out after 5003ms (test timeout), indicating the worker timeout mechanism is functioning but the worker itself is not responding
- **Memory Pressure Test Success**: Shows blob creation time degrades 3x under concurrent loads

#### Network Test Results

The `video-loading-network.test.ts` tests document bandwidth requirements:

- **Minimum Bandwidth for 10MB Video**: Test expects 700-800 kbps but calculation shows 5859 kbps required
- **Slow 3G Simulation (400 kbps)**: All video sizes timeout and fall back to main thread
- **CDN Latency Impact**: Up to 5 seconds additional latency compounds with download time
- **Network Viability Matrix**: 3G networks fail for all video sizes, 4G fails for videos >5MB

#### Performance Test Results

The `video-loading-performance.test.ts` tests show dramatic performance differences:

- **Worker vs Main Thread**: Worker takes 16000ms vs 1000ms for main thread (16x slower)
- **Size-Based Bottlenecks**:
  - 1MB videos: Metadata fetch dominates (40% of time)
  - 10MB videos: Video fetch dominates (70% of time)
  - 100MB videos: Blob creation dominates (74.5% of time)
- **Worker-Only Failures**: Videos that timeout in worker succeed immediately on main thread

### Root Causes of Worker Timeouts

#### 1. Blocking Network Operations

The worker performs two sequential blocking network operations ([video-processor.worker.js:87-122](public/workers/video-processor.worker.js#L87-L122)):

- **HEAD Request** (line 87): Blocks waiting for server response to get content metadata
- **GET Request** (line 112): Blocks downloading entire video file into memory

These operations cannot be parallelized and must complete sequentially. For a 10MB video on a 400 kbps connection, the GET request alone takes ~204 seconds, far exceeding the 15-second timeout.

#### 2. Memory-Intensive Blob Creation

The blob creation process ([video-processor.worker.js:127](public/workers/video-processor.worker.js#L127)) involves:

- Reading the entire response stream into memory
- Creating a new Blob object with the complete video data
- This operation blocks the worker thread and scales linearly with video size

Test results show blob creation taking 1.9 seconds for a 10MB video, increasing to 10+ seconds for 100MB videos. Under memory pressure with concurrent loads, this degrades 3x.

#### 3. Sequential Processing Without Streaming

The current implementation ([video-processor.worker.js:112-136](public/workers/video-processor.worker.js#L112-L136)):

1. Downloads the complete video file
2. Converts entire response to blob
3. Creates object URL
4. Sends URL back to main thread

This sequential, non-streaming approach means the entire video must be downloaded and processed before any part can be used, creating a bottleneck for large files.

#### 4. Network Bandwidth Constraints

Based on test calculations ([video-loading-network.test.ts:187-194](src/lib/videos/__tests__/video-loading-network.test.ts#L187-L194)):

- 1MB video requires ~586 kbps minimum
- 5MB video requires ~2929 kbps minimum
- 10MB video requires ~5859 kbps minimum
- 50MB video requires ~29297 kbps minimum

These calculations leave only 1 second for metadata fetch, blob creation, and coordinator overhead after the video download.

#### 5. Worker Overhead vs Main Thread

The main thread loading path ([video-texture-loader.ts:559-658](src/lib/videos/video-texture-loader.ts#L559-L658)):

- Sets video.src directly to the URL without downloading first
- Lets the browser's native video element handle streaming and buffering
- Avoids the fetch→blob conversion entirely
- Results in 16x faster loading for the same video

### Diagnostic Logging Implementation

Recent commits added comprehensive logging at multiple levels:

**Worker Level** ([video-processor.worker.js](public/workers/video-processor.worker.js)):

- Logs timing for each stage (metadata, fetch, blob)
- Reports timing breakdown in success message
- All logging gated behind development environment checks

**Coordinator Level** ([video-texture-loader.ts](src/lib/videos/video-texture-loader.ts)):

- Logs callback registration with timestamp
- Measures and logs worker response time
- Warns when response time exceeds 10 seconds
- Tracks which loading method (worker vs main thread) actually succeeded

**Component Level** ([video-material.tsx](src/components/r3f/blocks/video/video-material.tsx)):

- Records total component-level load time
- Warns for loads exceeding 10 seconds
- Uses accurate loading method from coordinator

### Historical Context

Previous research ([2025-01-13-ENG-1057-video-webworker-timeout-behavior.md](thoughts/shared/research/2025-01-13-ENG-1057-video-webworker-timeout-behavior.md)) identified:

- The misleading status reporting issue (fixed in commit 25552391b)
- The 15-second timeout is intentional for reliability
- Worker reference persistence after timeout is by design
- The fallback mechanism works correctly but defeats the purpose of offloading

The diagnostic plan ([2025-01-13-ENG-1057-video-webworker-diagnostics.md](thoughts/shared/plans/2025-01-13-ENG-1057-video-webworker-diagnostics.md)) was implemented across three commits:

1. **25552391b**: Fixed misleading status reporting
2. **00076734e**: Added comprehensive diagnostic logging
3. **1b1e192b2**: Added diagnostic tests
4. **6424810f1**: Added performance and network tests

## Code References

### Core Implementation

- `public/workers/video-processor.worker.js:77-169` - Worker video processing pipeline
- `src/lib/videos/video-texture-loader.ts:385-557` - Worker load coordination
- `src/lib/videos/video-texture-loader.ts:559-658` - Main thread fallback
- `src/lib/videos/video-texture-loader.ts:397-410` - 15-second timeout setup

### Test Files

- `src/lib/videos/__tests__/video-texture-loader-diagnostics.test.ts` - Timing diagnostics
- `src/lib/videos/__tests__/video-loading-network.test.ts` - Network conditions
- `src/lib/videos/__tests__/video-loading-performance.test.ts` - Performance comparison
- `scripts/test-video-loading.mjs` - Direct browser testing script

### Critical Bottlenecks

- `public/workers/video-processor.worker.js:112` - Blocking GET request
- `public/workers/video-processor.worker.js:127` - Blocking blob creation
- `public/workers/video-processor.worker.js:87` - Blocking HEAD request

## Architecture Documentation

### Worker Pipeline Stages

1. **Initialization**: Worker created and pinged for health check
2. **Message Dispatch**: loadVideo message sent with URL and blockId
3. **Metadata Fetch**: HEAD request to get content-type and length
4. **Video Download**: GET request downloads complete video file
5. **Blob Conversion**: Response stream converted to Blob object
6. **URL Creation**: Object URL created from blob
7. **Response**: Success message with URL and timing sent back
8. **Main Thread**: Video element created and loads from blob URL

### Timeout Boundaries

- **15-second hard limit**: Set at video-texture-loader.ts:410
- **10-second warning**: Triggered at video-texture-loader.ts:437
- **14-second practical limit**: Network tests assume 1-second safety margin
- **No timeout on main thread**: Fallback path has no timeout

### Memory Management

- Worker stores blobs in `videoCache` Map
- Object URLs tracked in `objectUrlRegistry` Map
- Cleanup performed on dispose or worker termination
- Multiple concurrent videos multiply memory usage

## Historical Context (from thoughts/)

The ENG-1057 ticket ([thoughts/shared/tickets/ENG-1057.md](thoughts/shared/tickets/ENG-1057.md)) describes:

- Video slow loading issues
- Video load failures on many canvases
- Need for diagnostic tooling

The implementation plan ([thoughts/shared/plans/2025-01-13-ENG-1057-video-webworker-diagnostics.md](thoughts/shared/plans/2025-01-13-ENG-1057-video-webworker-diagnostics.md)) outlined five phases:

1. Fix misleading status reporting (completed)
2. Add diagnostic logging (completed)
3. Implement unit tests (completed)
4. Add E2E tests (in progress)
5. Add worker progress tracking (planned)

## Related Research

- [2025-01-13-ENG-1057-video-webworker-timeout-behavior.md](thoughts/shared/research/2025-01-13-ENG-1057-video-webworker-timeout-behavior.md) - Initial investigation of timeout behavior

## Open Questions

1. Why does the worker use fetch→blob conversion instead of streaming?
2. Could the timeout be made adaptive based on video size?
3. Would chunked/progressive loading eliminate the timeout issue?
4. Should the worker be terminated after timeout to free memory?
5. Can the blob creation be optimized or avoided entirely?
