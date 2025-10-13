# Video WebWorker Diagnostics and Testing Implementation Plan

## Overview

Implement comprehensive logging diagnostics and automated tests to identify why the web worker fails to load videos within the 15-second timeout, fix the misleading status reporting that incorrectly shows "Video loaded using Web Worker" when main thread fallback was used, and prevent future regressions through automated testing.

## Current State Analysis

The video loading system has a race condition where `isUsingWorker()` only checks if a worker instance exists (`!!this.worker`) rather than tracking which path actually loaded the video. When the worker times out after 15 seconds and falls back to main thread loading, the system incorrectly reports "Video loaded using Web Worker" because the worker reference still exists.

## Desired End State

After implementation:
- Accurate logging that correctly identifies whether video was loaded via worker or main thread
- Comprehensive diagnostic logging to identify bottlenecks in the worker loading process
- Automated tests that verify timeout behavior and fallback mechanisms
- Clear visibility into video loading performance and failure points
- No misleading logs that confuse debugging efforts

### Key Discoveries:
- Worker timeout is hardcoded at 15 seconds at `src/lib/videos/video-texture-loader.ts:409`
- The `isUsingWorker()` method at line 625 only returns `!!this.worker`, not actual loading path
- Worker is not terminated on timeout, only on error or disposal
- The worker loads entire video into memory before creating blob URL
- Logging uses next-axiom with environment-aware patterns (`!isProdEnv` for console logs)

## What We're NOT Doing

- Not changing the 15-second timeout duration (that's a separate optimization task)
- Not refactoring the entire video loading architecture
- Not implementing streaming/chunked video loading in the worker
- Not changing the worker retry logic or initialization process
- Not modifying production logging verbosity (only adding development diagnostics)

## Implementation Approach

Fix the immediate logging issue first, then add comprehensive diagnostics to understand the timeout cause, and finally implement tests to prevent regressions. Each phase builds on the previous one and can be tested independently.

## Phase 1: Fix Misleading Status Reporting

### Overview
Track which loading path actually succeeded to fix the incorrect "Video loaded using Web Worker" message when main thread fallback was used.

### Changes Required:

#### 1. VideoTextureCoordinator - Add Loading Method Tracking
**File**: `src/lib/videos/video-texture-loader.ts`
**Changes**: Add a Map to track which method successfully loaded each video

```typescript
// Add after line 95 (after textures Map)
private loadingMethods: Map<string, 'worker' | 'main-thread'> = new Map();

// Update loadInWorker success path (around line 468)
// After: this.textures.set(blockId, result);
this.loadingMethods.set(blockId, 'worker');

// Update loadInMainThread success path (around line 533)
// After: this.textures.set(blockId, result);
this.loadingMethods.set(blockId, 'main-thread');

// Add new method after isUsingWorker (after line 626)
getLoadingMethod(blockId: string): 'worker' | 'main-thread' | null {
  return this.loadingMethods.get(blockId) || null;
}

// Update dispose method to clear the new Map (around line 615)
// After: this.textures.clear();
this.loadingMethods.clear();
```

#### 2. VideoMaterial Component - Use Accurate Status
**File**: `src/components/r3f/blocks/video/video-material.tsx`
**Changes**: Update logging to use actual loading method instead of worker existence check

```typescript
// Replace lines 126-139 with:
const status = videoCoordinator.getWorkerStatus();
const loadingMethod = videoCoordinator.getLoadingMethod(blockId);

if (loadingMethod === 'worker') {
  log.info("Video loaded using Web Worker", {
    blockId,
    videoUrl,
    workerStatus: status,
    loadingMethod,
  });
} else if (loadingMethod === 'main-thread') {
  log.info("Video loaded on main thread (after worker retry attempts)", {
    blockId,
    videoUrl,
    workerStatus: status,
    loadingMethod,
    workerWasAvailable: status.available,
  });
} else {
  log.warn("Video loaded but method unknown", {
    blockId,
    videoUrl,
    workerStatus: status,
  });
}
```

### Success Criteria:

#### Automated Verification:
- [x] TypeScript compilation passes: `pnpm typecheck`
- [x] Existing tests pass: `pnpm test:unit`
- [x] No linting errors: `pnpm lint`

#### Manual Verification:
- [ ] Load a video and verify correct "main thread" message appears after timeout
- [ ] Load a small video that completes within 15s and verify "Web Worker" message
- [ ] Verify no "unknown method" warnings appear
- [ ] Check that loading method persists correctly for cached videos

---

## Phase 2: Add Enhanced Diagnostic Logging

### Overview
Add detailed timing and progress logs throughout the video loading pipeline to identify bottlenecks.

### Changes Required:

#### 1. Worker Loading Diagnostics
**File**: `public/workers/video-processor.worker.js`
**Changes**: Add timing logs at each stage of video processing

```javascript
// Update processVideoStream function (starting at line 77)
async function processVideoStream(videoUrl, blockId, options = {}) {
  const startTime = Date.now();

  try {
    // Add before line 79 (HEAD request)
    console.log(`[Worker] Starting video load for ${blockId}`, {
      url: videoUrl,
      timestamp: startTime
    });

    // Add after line 91 (after HEAD response)
    const metadataTime = Date.now() - startTime;
    console.log(`[Worker] Metadata fetched for ${blockId}`, {
      duration: metadataTime,
      contentLength: metadata.contentLength
    });

    // Add before line 94 (GET request)
    const fetchStartTime = Date.now();
    console.log(`[Worker] Starting video fetch for ${blockId}`);

    // Add after line 97 (after response.ok check)
    const fetchTime = Date.now() - fetchStartTime;
    console.log(`[Worker] Video response received for ${blockId}`, {
      duration: fetchTime,
      status: videoResponse.status
    });

    // Add before line 99 (blob conversion)
    const blobStartTime = Date.now();

    // Add after line 99 (after blob creation)
    const blobTime = Date.now() - blobStartTime;
    console.log(`[Worker] Blob created for ${blockId}`, {
      duration: blobTime,
      blobSize: videoBlob.size
    });

    // Update the success message at line 111-117 to include timing
    self.postMessage({
      type: 'videoReady',
      blockId,
      success: true,
      videoUrl: objectUrl,
      originalUrl: videoUrl,
      timing: {
        total: Date.now() - startTime,
        metadata: metadataTime,
        fetch: fetchTime,
        blob: blobTime
      }
    });
```

#### 2. Coordinator Callback Timing
**File**: `src/lib/videos/video-texture-loader.ts`
**Changes**: Add logs to track callback registration and response times

```typescript
// Add after line 411 (callback registration)
const callbackRegisteredAt = Date.now();
if (!isProdEnv) {
  console.log(`[Coordinator] Registered callback for ${blockId}`, {
    timestamp: callbackRegisteredAt,
    workerStatus: this.getWorkerStatus(),
  });
}

// Add after line 412 (timeout cleared)
const responseTime = Date.now() - callbackRegisteredAt;
if (!isProdEnv) {
  console.log(`[Coordinator] Worker responded for ${blockId}`, {
    responseTime,
    messageType: message.type,
    success: message.success,
    timing: message.timing, // New timing data from worker
  });
}

// Add warning for near-timeout responses (after clearing timeout)
if (responseTime > 10000) {
  log.warn("Video load approaching timeout", {
    blockId,
    videoUrl,
    responseTime,
    timeoutThreshold: 15000,
    timing: message.timing,
  });
}
```

#### 3. Video Element Loading Timing
**File**: `src/lib/videos/video-texture-loader.ts`
**Changes**: Add logs for video element creation and loading

```typescript
// Add after line 423 (setting video.src from worker URL)
if (!isProdEnv) {
  console.log(`[Coordinator] Setting video src from worker for ${blockId}`, {
    url: message.videoUrl?.substring(0, 50) + '...',
    timestamp: Date.now(),
  });
}

// Add in the canplay handler (around line 426)
if (!isProdEnv) {
  console.log(`[Coordinator] Video element ready for ${blockId}`, {
    videoWidth: video.videoWidth,
    videoHeight: video.videoHeight,
    duration: video.duration,
    readyState: video.readyState,
  });
}

// Add similar logs for main thread path (lines 569 and 525)
```

#### 4. Component-Level Timing
**File**: `src/components/r3f/blocks/video/video-material.tsx`
**Changes**: Add timing at component level

```typescript
// Add at the start of loadVideo function (after line 88)
const componentLoadStartTime = Date.now();
if (!isProdEnv) {
  console.log(`[Component] Starting video load for ${blockId}`, {
    videoUrl,
    timestamp: componentLoadStartTime,
  });
}

// Add after successful load (after line 116)
const totalLoadTime = Date.now() - componentLoadStartTime;
if (!isProdEnv) {
  console.log(`[Component] Video load completed for ${blockId}`, {
    totalTime: totalLoadTime,
    loadingMethod: videoCoordinator.getLoadingMethod(blockId),
  });
}

// Add structured log for slow loads
if (totalLoadTime > 10000) {
  log.warn("Slow video load detected at component level", {
    blockId,
    videoUrl,
    totalTime: totalLoadTime,
    loadingMethod: videoCoordinator.getLoadingMethod(blockId),
  });
}
```

### Success Criteria:

#### Automated Verification:
- [x] TypeScript compilation passes: `pnpm typecheck`
- [x] No console errors in development mode
- [x] Logs only appear in development (`!isProdEnv`)

#### Manual Verification:
- [ ] Console shows detailed timing breakdown for video loads
- [ ] Worker fetch/blob timing is visible
- [ ] Near-timeout warnings appear for loads >10 seconds
- [ ] Can identify which stage is slowest (fetch vs blob vs element load)

---

## Phase 3: Implement Unit Tests

### Overview
Create comprehensive unit tests for timeout behavior, fallback mechanism, and status reporting.

### Changes Required:

#### 1. VideoTextureCoordinator Tests
**File**: `src/lib/videos/__tests__/video-texture-loader.test.ts` (new file)
**Changes**: Create test suite for coordinator

```typescript
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { VideoTextureCoordinator } from "../video-texture-loader";

// Mock dependencies
vi.mock("three", () => ({
  VideoTexture: class {
    constructor(video: any) {
      this.video = video;
    }
  },
}));

vi.mock("next-axiom", () => ({
  Logger: class {
    info = vi.fn();
    warn = vi.fn();
    error = vi.fn();
  },
}));

vi.mock("@/env/helpers", () => ({
  isProdEnv: false,
}));

describe("VideoTextureCoordinator", () => {
  let coordinator: VideoTextureCoordinator;
  let mockWorker: any;

  beforeEach(() => {
    vi.useFakeTimers();

    // Mock Worker constructor
    mockWorker = {
      postMessage: vi.fn(),
      addEventListener: vi.fn(),
      removeEventListener: vi.fn(),
      terminate: vi.fn(),
    };

    global.Worker = vi.fn(() => mockWorker) as any;

    coordinator = new VideoTextureCoordinator();
  });

  afterEach(() => {
    vi.useRealTimers();
    vi.clearAllMocks();
    coordinator.dispose();
  });

  describe("Worker Timeout and Fallback", () => {
    it("should timeout after 15 seconds and fallback to main thread", async () => {
      await coordinator.initialize();

      const loadPromise = coordinator.loadVideoTexture(
        "https://example.com/video.mp4",
        "test-block-id",
        {}
      );

      // Verify worker message was sent
      expect(mockWorker.postMessage).toHaveBeenCalledWith(
        expect.objectContaining({
          type: "loadVideo",
          videoUrl: "https://example.com/video.mp4",
          blockId: "test-block-id",
        })
      );

      // Advance time to trigger timeout
      vi.advanceTimersByTime(15000);

      // Mock successful main thread load
      const mockVideo = {
        src: "",
        addEventListener: vi.fn((event, handler) => {
          if (event === "canplay") {
            setTimeout(() => handler(), 100);
          }
        }),
        load: vi.fn(),
        play: vi.fn(),
      };

      global.document.createElement = vi.fn(() => mockVideo) as any;

      vi.advanceTimersByTime(100);

      const result = await loadPromise;

      expect(result).toBeDefined();
      expect(coordinator.getLoadingMethod("test-block-id")).toBe("main-thread");
    });

    it("should track loading method correctly for successful worker load", async () => {
      await coordinator.initialize();

      const loadPromise = coordinator.loadVideoTexture(
        "https://example.com/video.mp4",
        "test-block-id",
        {}
      );

      // Simulate worker response
      const messageHandler = mockWorker.addEventListener.mock.calls
        .find(call => call[0] === "message")?.[1];

      messageHandler?.({
        data: {
          type: "videoReady",
          blockId: "test-block-id",
          success: true,
          videoUrl: "blob:https://example.com/123",
        },
      });

      // Mock video element for worker path
      const mockVideo = {
        src: "",
        addEventListener: vi.fn((event, handler) => {
          if (event === "canplay") handler();
        }),
        load: vi.fn(),
      };

      global.document.createElement = vi.fn(() => mockVideo) as any;

      await loadPromise;

      expect(coordinator.getLoadingMethod("test-block-id")).toBe("worker");
    });

    it("should clear callbacks on timeout", async () => {
      await coordinator.initialize();

      const loadPromise = coordinator.loadVideoTexture(
        "https://example.com/video.mp4",
        "test-block-id",
        {}
      );

      // Check callback is registered
      expect((coordinator as any).workerCallbacks.has("test-block-id")).toBe(true);

      // Trigger timeout
      vi.advanceTimersByTime(15000);

      // Check callback is cleared
      expect((coordinator as any).workerCallbacks.has("test-block-id")).toBe(false);
    });
  });

  describe("Worker Initialization", () => {
    it("should retry initialization with exponential backoff", async () => {
      let attemptCount = 0;

      mockWorker.postMessage = vi.fn(() => {
        attemptCount++;
        if (attemptCount < 3) {
          throw new Error("Worker init failed");
        }
      });

      const initPromise = coordinator.initialize({ maxRetries: 3 });

      // First attempt fails immediately
      await vi.runAllTimersAsync();
      expect(attemptCount).toBe(1);

      // Second attempt after 1000ms
      vi.advanceTimersByTime(1000);
      await vi.runAllTimersAsync();
      expect(attemptCount).toBe(2);

      // Third attempt after 2000ms (exponential backoff)
      vi.advanceTimersByTime(2000);

      // Mock pong response for success
      const messageHandler = mockWorker.addEventListener.mock.calls
        .find(call => call[0] === "message")?.[1];
      messageHandler?.({ data: { type: "pong" } });

      const result = await initPromise;
      expect(result).toBe(true);
      expect(attemptCount).toBe(3);
    });

    it("should handle ping timeout", async () => {
      const initPromise = coordinator.initialize({ pingTimeout: 1000 });

      // Don't send pong response
      vi.advanceTimersByTime(1000);

      await expect(initPromise).resolves.toBe(false);
      expect(coordinator.isUsingWorker()).toBe(false);
    });
  });

  describe("Status Reporting", () => {
    it("should report correct worker status", async () => {
      const status = coordinator.getWorkerStatus();

      expect(status).toEqual({
        available: false,
        initialized: false,
        retryCount: 0,
        lastError: null,
        preloaded: false,
      });

      await coordinator.initialize();

      const updatedStatus = coordinator.getWorkerStatus();
      expect(updatedStatus.available).toBe(true);
      expect(updatedStatus.initialized).toBe(true);
    });
  });
});
```

#### 2. VideoMaterial Component Tests
**File**: `src/components/r3f/blocks/video/__tests__/video-material.test.tsx` (new file)
**Changes**: Test component behavior with different loading scenarios

```typescript
import { render, waitFor } from "@testing-library/react";
import { describe, expect, it, vi, beforeEach, afterEach } from "vitest";
import { VideoMaterial } from "../video-material";
import { videoCoordinator } from "@/lib/videos/video-texture-loader";

// Mock videoCoordinator
vi.mock("@/lib/videos/video-texture-loader", () => ({
  videoCoordinator: {
    loadVideoTexture: vi.fn(),
    getWorkerStatus: vi.fn(),
    getLoadingMethod: vi.fn(),
    isUsingWorker: vi.fn(),
  },
}));

// Mock logger
vi.mock("next-axiom", () => ({
  Logger: class {
    info = vi.fn();
    warn = vi.fn();
    error = vi.fn();
  },
}));

describe("VideoMaterial", () => {
  beforeEach(() => {
    vi.clearAllMocks();
  });

  it("should log correct message for worker-loaded video", async () => {
    const mockTexture = { needsUpdate: true };
    const mockVideo = { play: vi.fn() };

    (videoCoordinator.loadVideoTexture as any).mockResolvedValue({
      texture: mockTexture,
      video: mockVideo,
    });

    (videoCoordinator.getLoadingMethod as any).mockReturnValue("worker");
    (videoCoordinator.getWorkerStatus as any).mockReturnValue({
      available: true,
      initialized: true,
    });

    const { container } = render(
      <VideoMaterial
        blockId="test-block"
        videoUrl="https://example.com/video.mp4"
        shouldPlay={true}
      />
    );

    await waitFor(() => {
      expect(videoCoordinator.loadVideoTexture).toHaveBeenCalled();
    });

    // Verify correct loading method was retrieved
    expect(videoCoordinator.getLoadingMethod).toHaveBeenCalledWith("test-block");
  });

  it("should log correct message for main-thread-loaded video", async () => {
    const mockTexture = { needsUpdate: true };
    const mockVideo = { play: vi.fn() };

    (videoCoordinator.loadVideoTexture as any).mockResolvedValue({
      texture: mockTexture,
      video: mockVideo,
    });

    (videoCoordinator.getLoadingMethod as any).mockReturnValue("main-thread");
    (videoCoordinator.getWorkerStatus as any).mockReturnValue({
      available: true,
      initialized: true,
    });

    const { container } = render(
      <VideoMaterial
        blockId="test-block"
        videoUrl="https://example.com/video.mp4"
        shouldPlay={true}
      />
    );

    await waitFor(() => {
      expect(videoCoordinator.getLoadingMethod).toHaveBeenCalledWith("test-block");
    });
  });
});
```

### Success Criteria:

#### Automated Verification:
- [ ] All unit tests pass: `pnpm test:unit`
- [ ] Test coverage for video-texture-loader.ts > 80%
- [ ] Test coverage for video-material.tsx > 70%
- [ ] Tests run in < 10 seconds

#### Manual Verification:
- [ ] Tests accurately simulate timeout scenarios
- [ ] Fallback behavior is properly tested
- [ ] Loading method tracking is verified
- [ ] Worker initialization retry logic is tested

---

## Phase 4: Add E2E Tests

### Overview
Create end-to-end tests to verify video loading works correctly in real browser environment.

### Changes Required:

#### 1. Video Loading E2E Test
**File**: `tests/e2e/video-loading.spec.ts` (new file)
**Changes**: Test actual video loading scenarios

```typescript
import { expect, test } from "@playwright/test";

test.describe("Video Loading with Web Worker", () => {
  test.beforeEach(async ({ page }) => {
    // Setup similar to existing E2E tests
    await page.goto("/projects");
    // ... setup project
  });

  test("should load video successfully via worker", async ({ page }) => {
    test.setTimeout(60000); // 60 second timeout for video operations

    // Add video block to canvas
    await page.click('[data-testid="add-video-block"]');

    // Upload or select a small test video
    const videoInput = page.locator('input[type="file"]');
    await videoInput.setInputFiles("tests/fixtures/small-test-video.mp4");

    // Wait for video to load
    await page.waitForSelector('[data-testid="video-loaded"]', { timeout: 20000 });

    // Check console logs for correct loading method
    const consoleLogs: string[] = [];
    page.on("console", msg => {
      if (msg.text().includes("Video loaded")) {
        consoleLogs.push(msg.text());
      }
    });

    // Verify worker was used for small video
    expect(consoleLogs.some(log => log.includes("Video loaded using Web Worker")))
      .toBeTruthy();
  });

  test("should fallback to main thread on timeout", async ({ page }) => {
    test.setTimeout(120000); // 2 minute timeout

    // Mock slow network to trigger timeout
    await page.route("**/large-test-video.mp4", async route => {
      // Delay response to trigger 15-second timeout
      await new Promise(resolve => setTimeout(resolve, 16000));
      await route.continue();
    });

    // Add video block
    await page.click('[data-testid="add-video-block"]');

    // Try to load large/slow video
    const videoInput = page.locator('input[type="file"]');
    await videoInput.setInputFiles("tests/fixtures/large-test-video.mp4");

    // Should eventually load via main thread
    await page.waitForSelector('[data-testid="video-loaded"]', { timeout: 40000 });

    // Check console logs
    const consoleLogs: string[] = [];
    page.on("console", msg => {
      if (msg.text().includes("Video loaded")) {
        consoleLogs.push(msg.text());
      }
    });

    // Verify main thread was used after timeout
    expect(consoleLogs.some(log =>
      log.includes("Video loaded on main thread (after worker retry attempts)")
    )).toBeTruthy();
  });

  test("should show loading progress", async ({ page }) => {
    // Add video block
    await page.click('[data-testid="add-video-block"]');

    // Check loading indicator appears
    await expect(page.locator('[data-testid="video-loading"]')).toBeVisible();

    // Upload video
    const videoInput = page.locator('input[type="file"]');
    await videoInput.setInputFiles("tests/fixtures/small-test-video.mp4");

    // Loading indicator should disappear
    await expect(page.locator('[data-testid="video-loading"]')).not.toBeVisible();

    // Video should be visible
    await expect(page.locator('[data-testid="video-loaded"]')).toBeVisible();
  });

  test("should handle video load errors gracefully", async ({ page }) => {
    // Add video block
    await page.click('[data-testid="add-video-block"]');

    // Try to load invalid video URL
    await page.fill('[data-testid="video-url-input"]', "https://invalid-url/video.mp4");
    await page.click('[data-testid="load-video-button"]');

    // Should show error state
    await expect(page.locator('[data-testid="video-error"]')).toBeVisible();

    // Should not crash the application
    await expect(page.locator('[data-testid="canvas"]')).toBeVisible();
  });
});
```

#### 2. Test Fixtures
**File**: `tests/fixtures/` (new directory)
**Changes**: Add test video files

```bash
# Create test fixtures
tests/fixtures/
  small-test-video.mp4  # < 1MB video for quick loading
  large-test-video.mp4  # > 10MB video for timeout testing
```

### Success Criteria:

#### Automated Verification:
- [ ] E2E tests pass: `pnpm test:e2e`
- [ ] Tests run in CI/CD pipeline
- [ ] No flaky tests (run 5 times successfully)

#### Manual Verification:
- [ ] E2E tests accurately simulate real user interactions
- [ ] Timeout scenario is properly tested
- [ ] Error handling is verified
- [ ] Loading states are tested

---

## Phase 5: Add Worker Progress Tracking

### Overview
Enhance the worker to report loading progress for better visibility into what stage is taking time.

### Changes Required:

#### 1. Worker Progress Messages
**File**: `public/workers/video-processor.worker.js`
**Changes**: Add progress reporting during fetch

```javascript
// Update processVideoStream function to report progress
async function processVideoStream(videoUrl, blockId, options = {}) {
  const startTime = Date.now();

  try {
    // Report start
    self.postMessage({
      type: 'videoProgress',
      blockId,
      stage: 'started',
      progress: 0,
      timestamp: Date.now()
    });

    // ... existing metadata fetch ...

    self.postMessage({
      type: 'videoProgress',
      blockId,
      stage: 'metadata-fetched',
      progress: 10,
      metadata: {
        contentLength: metadata.contentLength,
        contentType: metadata.contentType
      },
      timestamp: Date.now()
    });

    // Fetch with progress tracking
    const videoResponse = await fetch(videoUrl);
    const reader = videoResponse.body.getReader();
    const contentLength = +videoResponse.headers.get('Content-Length');

    let receivedLength = 0;
    let chunks = [];
    let lastProgressUpdate = Date.now();

    while (true) {
      const { done, value } = await reader.read();

      if (done) break;

      chunks.push(value);
      receivedLength += value.length;

      // Report progress every 500ms or 10% increment
      const progress = Math.round((receivedLength / contentLength) * 100);
      const now = Date.now();

      if (now - lastProgressUpdate > 500 || progress % 10 === 0) {
        self.postMessage({
          type: 'videoProgress',
          blockId,
          stage: 'downloading',
          progress: 10 + (progress * 0.7), // 10-80% for download
          bytesReceived: receivedLength,
          bytesTotal: contentLength,
          timestamp: now
        });
        lastProgressUpdate = now;
      }
    }

    // Report blob creation start
    self.postMessage({
      type: 'videoProgress',
      blockId,
      stage: 'creating-blob',
      progress: 80,
      timestamp: Date.now()
    });

    // Create blob from chunks
    const videoBlob = new Blob(chunks);

    // ... existing blob URL creation ...

    self.postMessage({
      type: 'videoProgress',
      blockId,
      stage: 'completed',
      progress: 100,
      timestamp: Date.now()
    });

    // ... existing success message ...
  } catch (error) {
    self.postMessage({
      type: 'videoProgress',
      blockId,
      stage: 'error',
      error: error.message,
      timestamp: Date.now()
    });

    // ... existing error handling ...
  }
}
```

#### 2. Coordinator Progress Handling
**File**: `src/lib/videos/video-texture-loader.ts`
**Changes**: Handle progress messages from worker

```typescript
// Add progress callback type after line 88
type ProgressCallback = (progress: {
  stage: string;
  progress: number;
  bytesReceived?: number;
  bytesTotal?: number;
}) => void;

// Add progress tracking Map after loadingMethods Map
private progressCallbacks: Map<string, ProgressCallback> = new Map();

// Update handleWorkerMessage to handle progress (around line 309)
private handleWorkerMessage(event: MessageEvent): void {
  const message = event.data;

  // Add progress handling
  if (message.type === "videoProgress") {
    const progressCallback = this.progressCallbacks.get(message.blockId);
    if (progressCallback) {
      progressCallback({
        stage: message.stage,
        progress: message.progress,
        bytesReceived: message.bytesReceived,
        bytesTotal: message.bytesTotal,
      });

      // Log slow stages
      if (!isProdEnv && message.stage === 'downloading' && message.bytesReceived) {
        const mbReceived = (message.bytesReceived / 1024 / 1024).toFixed(2);
        const mbTotal = (message.bytesTotal / 1024 / 1024).toFixed(2);
        console.log(`[Worker Progress] ${message.blockId}: ${mbReceived}/${mbTotal} MB (${message.progress}%)`);
      }
    }
    return;
  }

  // ... existing message handling ...
}

// Add method to register progress callbacks
registerProgressCallback(blockId: string, callback: ProgressCallback): void {
  this.progressCallbacks.set(blockId, callback);
}

// Clear progress callback in dispose
// Update around line 615
this.progressCallbacks.clear();

// Clear progress callback when load completes
// Add in finally block of loadVideoTexture (around line 380)
this.progressCallbacks.delete(blockId);
```

#### 3. Component Progress Display
**File**: `src/components/r3f/blocks/video/video-material.tsx`
**Changes**: Display loading progress

```typescript
// Add progress state after line 85
const [loadingProgress, setLoadingProgress] = useState<{
  stage: string;
  progress: number;
}>({ stage: 'initializing', progress: 0 });

// Register progress callback before loading (around line 98)
videoCoordinator.registerProgressCallback(blockId, (progress) => {
  setLoadingProgress(progress);

  if (!isProdEnv) {
    console.log(`[Component Progress] ${blockId}: ${progress.stage} (${progress.progress}%)`);
  }
});

// Add progress to loading state UI (update skeleton material section)
// This would depend on your UI framework, but conceptually:
if (isLoading) {
  return (
    <>
      <SkeletonMaterial />
      {loadingProgress.progress > 0 && (
        <div>Loading: {loadingProgress.stage} ({loadingProgress.progress}%)</div>
      )}
    </>
  );
}
```

### Success Criteria:

#### Automated Verification:
- [ ] TypeScript compilation passes: `pnpm typecheck`
- [ ] Worker continues to function with progress messages
- [ ] No performance regression from progress tracking

#### Manual Verification:
- [ ] Progress updates appear in console during video load
- [ ] Can identify which stage is slowest (download vs blob creation)
- [ ] Progress reaches 100% before load completes
- [ ] Error states are properly reported

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation from the human that the manual testing was successful before considering the implementation complete.

---

## Testing Strategy

### Unit Tests:
- Test timeout behavior with fake timers
- Test worker initialization retry logic
- Test loading method tracking accuracy
- Test fallback mechanism triggers correctly
- Mock Worker API and video elements

### Integration Tests:
- Test actual Worker creation and messaging
- Test video element creation and events
- Verify logging output in different scenarios
- Test performance monitoring integration

### Manual Testing Steps:
1. Load a small video (<1MB) and verify worker loading succeeds
2. Load a large video (>50MB) and observe timeout behavior
3. Disable workers in browser and verify main thread loading
4. Check console logs show detailed timing information
5. Verify correct loading method is reported in logs
6. Test with slow network throttling to trigger timeouts
7. Verify progress tracking shows download percentage

## Performance Considerations

- All detailed logging is gated behind `!isProdEnv` checks to avoid production overhead
- Progress messages are throttled to every 500ms to avoid message spam
- Loading method tracking uses minimal memory (one Map entry per video)
- Worker timeout remains at 15 seconds to balance UX and resource usage

## Migration Notes

- No database migrations required
- No breaking API changes
- Backward compatible with existing video loading code
- Existing videos will continue to work without changes

## References

- Original research: `thoughts/searchable/shared/research/2025-01-13-ENG-1057-video-webworker-timeout-behavior.md`
- Video texture loader: `src/lib/videos/video-texture-loader.ts:409`
- Video material component: `src/components/r3f/blocks/video/video-material.tsx:126-139`
- Worker implementation: `public/workers/video-processor.worker.js:77-127`
- Test patterns: `src/hooks/__tests__/use-asset-drag-drop.test.tsx`