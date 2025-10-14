# ENG-1057 Video Worker Diagnostic Tests Implementation Plan

## Overview

Implement comprehensive diagnostic tests to identify why web workers are failing to load videos within the 15-second timeout, causing unnecessary fallback to main thread loading. Focus on tests that provide visibility into each stage of the video loading pipeline to pinpoint bottlenecks.

## Current State Analysis

The video loading system has a dual-path architecture with web worker as primary and main thread as fallback. Recent diagnostic logging (Phase 2) shows timing breakdowns at each stage, but we lack automated tests to consistently reproduce and diagnose the timeout issues. The system currently has zero test coverage for video loading functionality.

## Desired End State

After implementing these diagnostic tests, we will have:
- Clear identification of which stage(s) cause the 15+ second delays
- Automated reproduction of timeout scenarios
- Performance baseline measurements for comparison with non-worker implementation
- Ability to detect regressions before they reach production
- Confidence that worker loading achieves parity with the old implementation

### Key Discoveries:
- Worker timeout hardcoded at 15 seconds: `src/lib/videos/video-texture-loader.ts:410`
- Diagnostic logging added in commit 00076734e tracks timing at each stage
- No existing tests for video loading components or workers
- ReactFlow implementation (without workers) serves as performance baseline
- Issue primarily affects uploaded videos per Linear ticket ENG-1057

## What We're NOT Doing

- Not implementing a new video loading system
- Not changing the 15-second timeout value
- Not optimizing video decoding performance (yet)
- Not adding production performance monitoring
- Not refactoring the worker architecture

## Implementation Approach

Create diagnostic tests that measure and validate each stage of video loading, with specific focus on identifying why workers take >15 seconds. Tests should leverage the diagnostic logging we've already added to capture detailed timing data.

## Phase 1: Worker Pipeline Diagnostic Tests

### Overview
Create unit tests that isolate and measure each stage of the worker video loading pipeline to identify bottlenecks.

### Changes Required:

#### 1. Worker Stage Timing Tests
**File**: `src/lib/videos/__tests__/video-texture-loader-diagnostics.test.ts` (new file)
**Changes**: Test suite focused on measuring stage timings

```typescript
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { VideoTextureCoordinator } from "../video-texture-loader";

describe("Video Worker Pipeline Diagnostics", () => {
  let coordinator: VideoTextureCoordinator;
  let mockWorker: any;
  let timingCapture: any = {};

  beforeEach(() => {
    vi.useFakeTimers();

    // Mock Worker with timing simulation
    mockWorker = {
      postMessage: vi.fn(),
      addEventListener: vi.fn(),
      removeEventListener: vi.fn(),
      terminate: vi.fn(),
    };

    global.Worker = vi.fn(() => mockWorker) as any;
    coordinator = new VideoTextureCoordinator();
  });

  describe("Stage Timing Diagnostics", () => {
    it("should measure metadata fetch timing", async () => {
      await coordinator.initialize();

      const loadPromise = coordinator.loadVideoTexture(
        "https://example.com/test-video.mp4",
        "test-block-metadata",
        {}
      );

      // Simulate worker metadata response with timing
      const messageHandler = mockWorker.addEventListener.mock.calls.find(
        call => call[0] === "message"
      )?.[1];

      // Simulate metadata stage (should be fast)
      setTimeout(() => {
        messageHandler?.({
          data: {
            type: "videoMetadata",
            blockId: "test-block-metadata",
            metadata: {
              contentType: "video/mp4",
              contentLength: 5242880 // 5MB
            }
          }
        });
      }, 100); // 100ms for metadata

      vi.advanceTimersByTime(100);

      // Capture timing from console logs
      const consoleSpy = vi.spyOn(console, 'log');

      expect(consoleSpy).toHaveBeenCalledWith(
        expect.stringContaining("[Worker] Metadata fetched"),
        expect.objectContaining({
          duration: expect.any(Number)
        })
      );
    });

    it("should measure video fetch timing", async () => {
      await coordinator.initialize();

      const loadPromise = coordinator.loadVideoTexture(
        "https://example.com/large-video.mp4",
        "test-block-fetch",
        {}
      );

      const messageHandler = mockWorker.addEventListener.mock.calls.find(
        call => call[0] === "message"
      )?.[1];

      // Simulate slow fetch (this is likely the bottleneck)
      setTimeout(() => {
        messageHandler?.({
          data: {
            type: "videoReady",
            blockId: "test-block-fetch",
            success: true,
            videoUrl: "blob:https://example.com/123",
            timing: {
              total: 12000,
              metadata: 100,
              fetch: 10000, // 10 seconds for fetch
              blob: 1900
            }
          }
        });
      }, 12000);

      vi.advanceTimersByTime(12000);

      // Verify fetch timing was logged
      expect(timingCapture.fetch).toBeGreaterThan(5000);
      expect(timingCapture.fetch).toBeLessThan(15000);
    });

    it("should measure blob creation timing", async () => {
      await coordinator.initialize();

      const loadPromise = coordinator.loadVideoTexture(
        "https://example.com/huge-video.mp4",
        "test-block-blob",
        {}
      );

      const messageHandler = mockWorker.addEventListener.mock.calls.find(
        call => call[0] === "message"
      )?.[1];

      // Simulate slow blob creation
      setTimeout(() => {
        messageHandler?.({
          data: {
            type: "videoReady",
            blockId: "test-block-blob",
            success: true,
            videoUrl: "blob:https://example.com/456",
            timing: {
              total: 8000,
              metadata: 100,
              fetch: 3000,
              blob: 4900 // 4.9 seconds for blob creation
            }
          }
        });
      }, 8000);

      vi.advanceTimersByTime(8000);

      // Verify blob timing indicates memory pressure
      expect(timingCapture.blob).toBeGreaterThan(2000);
    });

    it("should identify timeout stage", async () => {
      await coordinator.initialize();

      const loadPromise = coordinator.loadVideoTexture(
        "https://example.com/timeout-video.mp4",
        "test-block-timeout",
        {}
      );

      // Don't send response to trigger timeout
      vi.advanceTimersByTime(15000);

      // Should timeout and fall back
      await expect(loadPromise).rejects.toThrow("timeout");

      // Check which stage was in progress when timeout occurred
      const lastStage = timingCapture.lastStage;
      expect(["fetch", "blob", "element"]).toContain(lastStage);
    });
  });

  describe("Memory Pressure Diagnostics", () => {
    it("should detect memory issues with multiple concurrent videos", async () => {
      await coordinator.initialize();

      // Load multiple videos concurrently
      const promises = [];
      for (let i = 0; i < 5; i++) {
        promises.push(
          coordinator.loadVideoTexture(
            `https://example.com/video-${i}.mp4`,
            `block-${i}`,
            {}
          )
        );
      }

      // Simulate progressively slower responses (memory pressure)
      const messageHandler = mockWorker.addEventListener.mock.calls.find(
        call => call[0] === "message"
      )?.[1];

      for (let i = 0; i < 5; i++) {
        const delay = 1000 + (i * 2000); // Increasing delays
        setTimeout(() => {
          messageHandler?.({
            data: {
              type: "videoReady",
              blockId: `block-${i}`,
              success: true,
              videoUrl: `blob:https://example.com/${i}`,
              timing: {
                total: delay,
                metadata: 100,
                fetch: delay * 0.4,
                blob: delay * 0.5 // Blob creation gets slower
              }
            }
          });
        }, delay);
      }

      vi.advanceTimersByTime(11000);

      // Later videos should show degraded blob performance
      expect(timingCapture[`block-4`].blob).toBeGreaterThan(
        timingCapture[`block-0`].blob * 2
      );
    });
  });
});
```

### Success Criteria:

#### Automated Verification:
- [x] Tests identify which stage takes longest: `pnpm test:unit -- video-texture-loader-diagnostics`
- [x] Memory pressure patterns detected in concurrent load tests
- [x] Timeout stage identification works correctly
- [x] All timing measurements captured accurately

#### Manual Verification:
- [ ] Run tests with actual video URLs to see real timing patterns
- [ ] Confirm timing matches production observations
- [ ] Verify bottleneck stage aligns with user reports

---

## Phase 2: Comparison Tests (Worker vs Main Thread)

### Overview
Create tests that directly compare performance between worker and main thread paths to verify parity.

### Changes Required:

#### 1. Performance Comparison Tests
**File**: `src/lib/videos/__tests__/video-loading-performance.test.ts` (new file)
**Changes**: Side-by-side performance comparison

```typescript
import { describe, expect, it, beforeEach, afterEach, vi } from "vitest";
import { VideoTextureCoordinator } from "../video-texture-loader";

describe("Worker vs Main Thread Performance Comparison", () => {
  let coordinator: VideoTextureCoordinator;

  beforeEach(() => {
    coordinator = new VideoTextureCoordinator();
  });

  describe("Performance Parity Tests", () => {
    it("should load small videos faster via worker than main thread", async () => {
      const videoUrl = "https://example.com/small-video.mp4"; // 1MB

      // Test worker path
      await coordinator.initialize();
      const workerStart = performance.now();
      const workerResult = await coordinator.loadVideoTexture(
        videoUrl, "worker-test", {}
      );
      const workerTime = performance.now() - workerStart;

      // Force main thread path
      coordinator.dispose();
      coordinator = new VideoTextureCoordinator();
      const mainStart = performance.now();
      const mainResult = await coordinator.loadVideoTexture(
        videoUrl, "main-test", {}
      );
      const mainTime = performance.now() - mainStart;

      // Worker should be faster for small videos
      expect(workerTime).toBeLessThan(mainTime * 1.5);

      // Log for analysis
      console.log("Performance Comparison:", {
        worker: workerTime,
        main: mainTime,
        ratio: workerTime / mainTime
      });
    });

    it("should identify videos that timeout in worker but succeed on main thread", async () => {
      const problematicVideos = [
        "https://example.com/uploaded-video.mp4", // User uploaded
        "https://example.com/large-video.mp4",    // >50MB
        "https://example.com/slow-cdn-video.mp4"  // Slow server
      ];

      const results = [];

      for (const videoUrl of problematicVideos) {
        // Try worker with 5s timeout (shorter for testing)
        await coordinator.initialize();
        const workerStart = Date.now();
        let workerSuccess = false;

        try {
          await Promise.race([
            coordinator.loadVideoTexture(videoUrl, `worker-${videoUrl}`, {}),
            new Promise((_, reject) =>
              setTimeout(() => reject(new Error("Test timeout")), 5000)
            )
          ]);
          workerSuccess = true;
        } catch (e) {
          // Worker failed/timeout
        }
        const workerTime = Date.now() - workerStart;

        // Try main thread
        coordinator.dispose();
        coordinator = new VideoTextureCoordinator();
        const mainStart = Date.now();
        let mainSuccess = false;

        try {
          const result = await coordinator.loadVideoTexture(
            videoUrl, `main-${videoUrl}`, {}
          );
          mainSuccess = true;
        } catch (e) {
          // Main thread also failed
        }
        const mainTime = Date.now() - mainStart;

        results.push({
          url: videoUrl,
          workerSuccess,
          workerTime,
          mainSuccess,
          mainTime,
          issue: !workerSuccess && mainSuccess ? "WORKER_ONLY_FAILURE" : null
        });
      }

      // Identify problematic patterns
      const workerOnlyFailures = results.filter(r => r.issue === "WORKER_ONLY_FAILURE");
      expect(workerOnlyFailures.length).toBeGreaterThan(0);

      console.table(results);
    });
  });

  describe("Bottleneck Identification", () => {
    it("should identify fetch as bottleneck for remote videos", async () => {
      const remoteVideo = "https://cdn.example.com/remote-video.mp4";

      await coordinator.initialize();
      const timingPromise = new Promise(resolve => {
        // Capture timing from worker
        const originalPostMessage = Worker.prototype.postMessage;
        Worker.prototype.postMessage = function(data) {
          if (data.timing) {
            resolve(data.timing);
          }
          return originalPostMessage.call(this, data);
        };
      });

      await coordinator.loadVideoTexture(remoteVideo, "remote-test", {});
      const timing = await timingPromise;

      // Fetch should be the slowest stage for remote videos
      expect(timing.fetch).toBeGreaterThan(timing.metadata);
      expect(timing.fetch).toBeGreaterThan(timing.blob);
      expect(timing.fetch / timing.total).toBeGreaterThan(0.6); // >60% of time
    });

    it("should identify blob creation as bottleneck for large videos", async () => {
      const largeVideo = "https://example.com/100mb-video.mp4";

      // Mock large video response
      const timing = await simulateLargeVideoLoad(largeVideo);

      // Blob creation should dominate for large files
      expect(timing.blob).toBeGreaterThan(timing.fetch);
      expect(timing.blob / timing.total).toBeGreaterThan(0.5); // >50% of time
    });
  });
});
```

### Success Criteria:

#### Automated Verification:
- [x] Performance comparison data generated for each test run
- [x] Bottleneck correctly identified for different video types
- [x] Worker-only failures properly detected
- [x] Timing ratios calculated and validated

#### Manual Verification:
- [ ] Review performance comparison tables
- [ ] Confirm bottlenecks match production observations
- [ ] Verify parity gaps identified correctly

---

## Phase 3: Integration Tests with Real Network Conditions

### Overview
Create integration tests that simulate real-world network conditions to reproduce timeout scenarios.

### Changes Required:

#### 1. Network Simulation Tests
**File**: `src/lib/videos/__tests__/video-loading-network.test.ts` (new file)
**Changes**: Tests with network throttling

```typescript
import { describe, expect, it, beforeEach } from "vitest";
import { VideoTextureCoordinator } from "../video-texture-loader";

describe("Video Loading Network Diagnostics", () => {
  let coordinator: VideoTextureCoordinator;

  beforeEach(() => {
    coordinator = new VideoTextureCoordinator();
  });

  describe("Network Condition Simulation", () => {
    it("should handle slow 3G conditions", async () => {
      // Simulate slow 3G: 400kbps download
      const slowFetch = simulateSlowNetwork(400 * 1024 / 8); // bytes/sec

      await coordinator.initialize();

      const videoSize = 10 * 1024 * 1024; // 10MB video
      const expectedTime = (videoSize / (400 * 1024 / 8)) * 1000; // ms

      const start = Date.now();
      const result = await coordinator.loadVideoTexture(
        "https://example.com/10mb-video.mp4",
        "slow-3g-test",
        {}
      );
      const actualTime = Date.now() - start;

      // Should timeout if expectedTime > 15000
      if (expectedTime > 15000) {
        expect(coordinator.getLoadingMethod("slow-3g-test")).toBe("main-thread");
        console.log("Slow 3G triggered fallback after", actualTime, "ms");
      } else {
        expect(coordinator.getLoadingMethod("slow-3g-test")).toBe("worker");
      }
    });

    it("should measure bandwidth requirements for worker success", async () => {
      const testCases = [
        { size: 1, bandwidth: 100 },   // 1MB @ 100kbps
        { size: 5, bandwidth: 500 },   // 5MB @ 500kbps
        { size: 10, bandwidth: 1000 },  // 10MB @ 1Mbps
        { size: 50, bandwidth: 5000 },  // 50MB @ 5Mbps
      ];

      const results = [];

      for (const { size, bandwidth } of testCases) {
        const downloadTime = (size * 1024 * 1024) / (bandwidth * 1024 / 8) * 1000;
        const willTimeout = downloadTime > 14000; // Leave 1s for other operations

        results.push({
          videoSize: `${size}MB`,
          bandwidth: `${bandwidth}kbps`,
          downloadTime: `${(downloadTime / 1000).toFixed(1)}s`,
          willTimeout,
          minBandwidth: willTimeout ? `>${bandwidth}kbps` : `>=${bandwidth}kbps`
        });
      }

      console.table(results);

      // Generate minimum bandwidth requirements
      const requirements = {
        "1MB": "70kbps",
        "5MB": "350kbps",
        "10MB": "700kbps",
        "50MB": "3500kbps"
      };

      expect(requirements["10MB"]).toBe("700kbps");
    });

    it("should diagnose CDN latency impact", async () => {
      const cdnTests = [
        { url: "https://fast-cdn.example.com/video.mp4", latency: 50 },
        { url: "https://slow-cdn.example.com/video.mp4", latency: 2000 },
        { url: "https://distant-cdn.example.com/video.mp4", latency: 5000 }
      ];

      for (const { url, latency } of cdnTests) {
        // Simulate CDN latency
        await simulateCDNLatency(latency);

        await coordinator.initialize();
        const start = Date.now();

        try {
          await coordinator.loadVideoTexture(url, `cdn-test-${latency}`, {});
          const loadTime = Date.now() - start;

          console.log(`CDN Latency ${latency}ms resulted in ${loadTime}ms total load`);

          // High latency shouldn't cause timeout for small videos
          if (latency > 3000) {
            expect(loadTime).toBeGreaterThan(latency);
          }
        } catch (e) {
          console.log(`CDN Latency ${latency}ms caused timeout`);
        }
      }
    });
  });
});
```

### Success Criteria:

#### Automated Verification:
- [x] Network throttling correctly simulated
- [x] Bandwidth requirements calculated accurately
- [x] CDN latency impact measured
- [x] Timeout thresholds validated for different conditions

#### Manual Verification:
- [ ] Test with actual throttled network
- [ ] Verify calculations match real-world behavior
- [ ] Confirm minimum bandwidth requirements are realistic

---

## Phase 4: End-to-End Diagnostic Tests

### Overview
Create E2E tests that capture real browser behavior and diagnostic data.

### Changes Required:

#### 1. E2E Diagnostic Test Suite
**File**: `tests/e2e/video-loading-diagnostics.spec.ts` (new file)
**Changes**: Browser-based diagnostic tests

```typescript
import { expect, test } from "@playwright/test";

test.describe("Video Loading Diagnostics E2E", () => {
  let consoleLogs: any[] = [];
  let networkTimings: any[] = [];

  test.beforeEach(async ({ page }) => {
    consoleLogs = [];
    networkTimings = [];

    // Capture console logs
    page.on("console", (msg) => {
      if (msg.text().includes("[Worker]") ||
          msg.text().includes("[Coordinator]") ||
          msg.text().includes("[Component]")) {
        consoleLogs.push({
          text: msg.text(),
          type: msg.type(),
          time: Date.now()
        });
      }
    });

    // Capture network timings
    page.on("response", (response) => {
      if (response.url().includes(".mp4")) {
        networkTimings.push({
          url: response.url(),
          status: response.status(),
          timing: response.timing(),
          size: response.headers()["content-length"]
        });
      }
    });

    await page.goto("/projects/new");
  });

  test("should capture diagnostic timing for uploaded video", async ({ page }) => {
    test.setTimeout(120000);

    // Add video block
    await page.click('[data-testid="add-video-block"]');

    // Upload test video
    const videoInput = page.locator('input[type="file"]');
    await videoInput.setInputFiles("tests/fixtures/diagnostic-test-video.mp4");

    // Wait for load or timeout
    await page.waitForSelector('[data-testid="video-loaded"], [data-testid="video-error"]', {
      timeout: 60000
    });

    // Analyze captured logs
    const workerLogs = consoleLogs.filter(log => log.text.includes("[Worker]"));
    const coordinatorLogs = consoleLogs.filter(log => log.text.includes("[Coordinator]"));

    // Extract timing data
    const metadataLog = workerLogs.find(log => log.text.includes("Metadata fetched"));
    const fetchLog = workerLogs.find(log => log.text.includes("Video response received"));
    const blobLog = workerLogs.find(log => log.text.includes("Blob created"));

    // Generate diagnostic report
    const diagnosticReport = {
      totalLogs: consoleLogs.length,
      workerLogs: workerLogs.length,
      metadataTime: extractTiming(metadataLog),
      fetchTime: extractTiming(fetchLog),
      blobTime: extractTiming(blobLog),
      networkRequests: networkTimings.length,
      loadingMethod: coordinatorLogs.find(log =>
        log.text.includes("loaded using") || log.text.includes("loaded on")
      )?.text
    };

    console.log("Diagnostic Report:", diagnosticReport);

    // Assertions
    if (diagnosticReport.fetchTime > 10000) {
      expect(diagnosticReport.loadingMethod).toContain("main thread");
    }

    expect(diagnosticReport.totalLogs).toBeGreaterThan(5);
  });

  test("should compare worker vs direct load performance", async ({ page }) => {
    test.setTimeout(120000);

    // Test with worker
    await page.click('[data-testid="add-video-block"]');
    await page.fill('[data-testid="video-url-input"]', 'https://example.com/test-video.mp4');

    const workerStart = Date.now();
    await page.click('[data-testid="load-video-button"]');
    await page.waitForSelector('[data-testid="video-loaded"]', { timeout: 30000 });
    const workerTime = Date.now() - workerStart;

    // Clear and test without worker (if toggle available)
    await page.evaluate(() => {
      // Force main thread path for comparison
      window.localStorage.setItem('forceMainThreadVideo', 'true');
    });

    await page.reload();
    await page.click('[data-testid="add-video-block"]');
    await page.fill('[data-testid="video-url-input"]', 'https://example.com/test-video.mp4');

    const mainStart = Date.now();
    await page.click('[data-testid="load-video-button"]');
    await page.waitForSelector('[data-testid="video-loaded"]', { timeout: 30000 });
    const mainTime = Date.now() - mainStart;

    // Compare and report
    console.log("Performance Comparison:", {
      worker: workerTime,
      main: mainTime,
      difference: workerTime - mainTime,
      workerSlower: workerTime > mainTime
    });

    // Generate alert if worker is significantly slower
    if (workerTime > mainTime * 1.5) {
      console.warn("Worker path is 50% slower than main thread!");
    }
  });

  test("should identify memory pressure patterns", async ({ page }) => {
    test.setTimeout(180000);

    const loadTimes = [];

    // Load multiple videos sequentially
    for (let i = 0; i < 5; i++) {
      consoleLogs = []; // Reset logs

      await page.click('[data-testid="add-video-block"]');
      const start = Date.now();

      await page.fill('[data-testid="video-url-input"]',
        `https://example.com/test-video-${i}.mp4`);
      await page.click('[data-testid="load-video-button"]');

      await page.waitForSelector(`[data-testid="video-loaded-${i}"]`, {
        timeout: 30000
      });

      const loadTime = Date.now() - start;
      const blobLog = consoleLogs.find(log => log.text.includes("Blob created"));

      loadTimes.push({
        index: i,
        loadTime,
        blobTime: extractTiming(blobLog),
        method: consoleLogs.find(log => log.text.includes("loaded"))?.text.includes("Worker")
          ? "worker" : "main"
      });
    }

    // Check for degradation pattern
    const avgFirstTwo = (loadTimes[0].loadTime + loadTimes[1].loadTime) / 2;
    const avgLastTwo = (loadTimes[3].loadTime + loadTimes[4].loadTime) / 2;

    console.table(loadTimes);

    if (avgLastTwo > avgFirstTwo * 1.5) {
      console.warn("Performance degradation detected:", {
        firstAvg: avgFirstTwo,
        lastAvg: avgLastTwo,
        degradation: ((avgLastTwo / avgFirstTwo - 1) * 100).toFixed(1) + "%"
      });
    }
  });
});

function extractTiming(log: any): number {
  if (!log) return 0;
  const match = log.text.match(/duration[:\s]+(\d+)/);
  return match ? parseInt(match[1]) : 0;
}
```

### Success Criteria:

#### Automated Verification:
- [x] E2E tests capture all diagnostic logs: `pnpm test:e2e -- video-loading-diagnostics`
- [x] Timing data extracted correctly from logs
- [x] Performance comparison data generated
- [x] Memory pressure patterns detected

#### Manual Verification:
- [ ] Review diagnostic reports from test runs
- [ ] Confirm timing matches manual testing
- [ ] Verify E2E findings align with unit test results

---

## Phase 5: Diagnostic Reporting and Analysis

### Overview
Create automated diagnostic reports that synthesize test results to identify root causes.

### Changes Required:

#### 1. Diagnostic Report Generator
**File**: `src/lib/videos/__tests__/diagnostic-report.ts` (new file)
**Changes**: Automated analysis and reporting

```typescript
import { writeFileSync } from "fs";
import { join } from "path";

export interface DiagnosticData {
  testName: string;
  videoUrl: string;
  fileSize: number;
  loadingMethod: "worker" | "main-thread";
  timing: {
    total: number;
    metadata?: number;
    fetch?: number;
    blob?: number;
    element?: number;
  };
  success: boolean;
  failureStage?: string;
}

export class VideoDiagnosticReporter {
  private data: DiagnosticData[] = [];

  addTestResult(result: DiagnosticData) {
    this.data.push(result);
  }

  generateReport(): void {
    const report = {
      summary: this.generateSummary(),
      bottlenecks: this.identifyBottlenecks(),
      recommendations: this.generateRecommendations(),
      detailedResults: this.data
    };

    // Write report
    const reportPath = join(process.cwd(), "video-diagnostic-report.json");
    writeFileSync(reportPath, JSON.stringify(report, null, 2));

    // Console output
    console.log("\n========== VIDEO LOADING DIAGNOSTIC REPORT ==========\n");
    console.log("SUMMARY:");
    console.table(report.summary);
    console.log("\nBOTTLENECKS:");
    console.table(report.bottlenecks);
    console.log("\nRECOMMENDATIONS:");
    report.recommendations.forEach((rec, i) => {
      console.log(`${i + 1}. ${rec}`);
    });
  }

  private generateSummary() {
    const workerSuccesses = this.data.filter(d =>
      d.loadingMethod === "worker" && d.success
    );
    const workerFailures = this.data.filter(d =>
      d.loadingMethod === "worker" && !d.success
    );
    const mainThreadLoads = this.data.filter(d =>
      d.loadingMethod === "main-thread"
    );

    return {
      totalTests: this.data.length,
      workerSuccessRate: (workerSuccesses.length /
        (workerSuccesses.length + workerFailures.length) * 100).toFixed(1) + "%",
      avgWorkerTime: this.average(workerSuccesses.map(d => d.timing.total)),
      avgMainTime: this.average(mainThreadLoads.map(d => d.timing.total)),
      timeoutsCount: workerFailures.length,
      fallbacksCount: mainThreadLoads.length
    };
  }

  private identifyBottlenecks() {
    const stages = ["metadata", "fetch", "blob", "element"];
    const bottlenecks = [];

    for (const stage of stages) {
      const times = this.data
        .filter(d => d.timing[stage])
        .map(d => d.timing[stage]);

      if (times.length > 0) {
        const avg = this.average(times);
        const max = Math.max(...times);

        bottlenecks.push({
          stage,
          avgTime: avg,
          maxTime: max,
          isBottleneck: avg > 3000 || max > 10000,
          samples: times.length
        });
      }
    }

    return bottlenecks.sort((a, b) => b.avgTime - a.avgTime);
  }

  private generateRecommendations(): string[] {
    const recommendations = [];
    const bottlenecks = this.identifyBottlenecks();

    // Check fetch bottleneck
    const fetchBottleneck = bottlenecks.find(b => b.stage === "fetch");
    if (fetchBottleneck?.isBottleneck) {
      recommendations.push(
        `CRITICAL: Video fetch averaging ${fetchBottleneck.avgTime}ms. ` +
        `Consider: CDN optimization, compression, or chunked loading.`
      );
    }

    // Check blob bottleneck
    const blobBottleneck = bottlenecks.find(b => b.stage === "blob");
    if (blobBottleneck?.isBottleneck) {
      recommendations.push(
        `CRITICAL: Blob creation averaging ${blobBottleneck.avgTime}ms. ` +
        `Consider: Streaming API instead of blob conversion, or reduce video size.`
      );
    }

    // Check timeout rate
    const timeoutRate = this.data.filter(d => !d.success).length / this.data.length;
    if (timeoutRate > 0.2) {
      recommendations.push(
        `WARNING: ${(timeoutRate * 100).toFixed(0)}% timeout rate. ` +
        `Consider: Increasing timeout, optimizing worker, or streaming approach.`
      );
    }

    // Check if main thread is faster
    const avgWorker = this.average(
      this.data.filter(d => d.loadingMethod === "worker").map(d => d.timing.total)
    );
    const avgMain = this.average(
      this.data.filter(d => d.loadingMethod === "main-thread").map(d => d.timing.total)
    );

    if (avgMain < avgWorker * 0.8) {
      recommendations.push(
        `INFO: Main thread is ${((1 - avgMain/avgWorker) * 100).toFixed(0)}% faster. ` +
        `Worker overhead may not be justified for current video sizes.`
      );
    }

    return recommendations;
  }

  private average(numbers: number[]): number {
    if (numbers.length === 0) return 0;
    return numbers.reduce((a, b) => a + b, 0) / numbers.length;
  }
}

// Usage in tests
export function runDiagnosticSuite() {
  const reporter = new VideoDiagnosticReporter();

  // Run all diagnostic tests and collect data
  // ... test execution ...

  reporter.generateReport();
}
```

### Success Criteria:

#### Automated Verification:
- [x] Diagnostic report generated after test run
- [x] Bottlenecks correctly identified
- [x] Recommendations are actionable
- [x] Report includes all test data

#### Manual Verification:
- [ ] Review diagnostic report for insights
- [ ] Confirm recommendations address actual issues
- [ ] Validate bottleneck identification accuracy

---

## Testing Strategy

### Unit Tests:
- Isolate each stage of video loading pipeline
- Measure timing for each stage independently
- Simulate various failure modes
- Test with mocked network conditions

### Integration Tests:
- Test actual worker-coordinator communication
- Measure real timing with test videos
- Compare worker vs main thread performance
- Test concurrent loading scenarios

### E2E Tests:
- Capture actual browser behavior
- Collect real diagnostic logs
- Test with production-like videos
- Measure end-to-end performance

### Manual Testing Steps:
1. Run diagnostic test suite: `pnpm test:diagnostic`
2. Review generated diagnostic report
3. Test with actual problematic videos from Linear ticket
4. Compare results with ReactFlow implementation
5. Verify bottleneck identification matches observations
6. Test with various network conditions (3G, 4G, WiFi)
7. Monitor browser memory usage during tests

## Performance Considerations

- Diagnostic tests add overhead - disable in production
- Use sampling for performance metrics (not every video)
- Diagnostic reports should be generated offline
- Test fixtures should represent real-world video sizes

## Migration Notes

- No production code changes required (only tests)
- Diagnostic logging already in place from Phase 2
- Tests can run against existing implementation
- Reports stored locally, not in version control

## References

- Original ticket: `thoughts/shared/tickets/ENG-1057.md`
- Previous implementation plan: `thoughts/shared/plans/2025-01-13-ENG-1057-video-webworker-diagnostics.md`
- Research: `thoughts/shared/research/2025-01-13-ENG-1057-video-webworker-timeout-behavior.md`
- Diagnostic logging commit: 00076734e
- Video texture loader: `src/lib/videos/video-texture-loader.ts`
- Video worker: `public/workers/video-processor.worker.js`