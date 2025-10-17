---
date: 2025-10-17 14:22:12 EDT
researcher: Research Agent
git_commit: 538eb92ba35d0277659295cb77364ff9d8a0f693
branch: eng-1084
repository: eng-1084
topic: "R3F Canvas Temporary Connection Line Rendering and Z-Index Management"
tags: [research, codebase, r3f, three-js, connection-line, z-index, rendering-order, canvas-3d]
status: complete
last_updated: 2025-10-17
last_updated_by: Research Agent
---

# Research: R3F Canvas Temporary Connection Line Rendering and Z-Index Management

**Date**: 2025-10-17 14:22:12 EDT
**Researcher**: Research Agent
**Git Commit**: 538eb92ba35d0277659295cb77364ff9d8a0f693
**Branch**: eng-1084
**Repository**: eng-1084

## Research Question

How is the temporary connection line currently rendered in the R3F (React Three Fiber) canvas implementation, specifically focusing on z-index/layering management to understand why it might not appear on top of all nodes while dragging?

## Summary

The R3F canvas uses a 3D z-position based layering system rather than traditional z-index. The temporary connection line (`TempConnectionLine` component) renders during handle dragging using a bezier curve with ribbon geometry. Currently, the temporary line has no explicit z-position set in its mesh, relying on the z-values from the connection points (z=10 from `use-connection-manager.tsx:85`). This z-position of 10 places it above regular blocks (z=0 to 400) but the rendering behavior depends on Three.js depth buffer and transparency blending. The issue described in ENG-1084 suggests the line may not always appear on top due to missing explicit z-position management or render order configuration.

## Detailed Findings

### Component Architecture

#### Core Components Location (`src/components/r3f/`)

The R3F canvas implementation consists of 100+ files in the r3f directory with the following key components for connection handling:

- **TempConnectionLine Component** (`src/components/r3f/temp-connection-line.tsx:24`): Renders the temporary curved line during connection dragging
- **ConnectionHandle Component** (`src/components/r3f/connection-handle.tsx:39`): Interactive handles on nodes for starting connections
- **Edge Component** (`src/components/r3f/Edge.tsx`): Permanent connection lines between nodes
- **ConnectionManagerContext** (`src/components/r3f/connection-manager-context.tsx`): Provides connection state to components
- **useConnectionManager Hook** (`src/hooks/use-connection-manager.tsx:18`): Core logic for connection state management

#### Canvas Integration

- **FloraCanvasContent** (`src/components/workspace/multiplayer/flora-canvas-content.tsx:740`): Main canvas that renders `<TempConnectionLine />` in the scene hierarchy
- **FloraCanvas** (`src/components/workspace/multiplayer/flora-canvas.tsx`): R3F Canvas wrapper with GL configuration

### Current Implementation

#### Connection State Management

The connection system tracks state through `useConnectionManager` hook:

```typescript
// use-connection-manager.tsx:8-16
type ConnectionRef = {
  active: boolean;
  start: THREE.Vector3;
  end: THREE.Vector3;
  sourceId?: string;
  sourceSide?: "left" | "right";
  hoveredId?: string;
  hoveredSide?: "left" | "right";
};
```

When a connection starts (`use-connection-manager.tsx:77-101`):
- Handle position calculated based on node position and width
- **Critical**: Start and end positions set with `z=10` (`use-connection-manager.tsx:85`)
- This z-position is much higher than regular edges (z=-0.5) and blocks (z=0-400)

#### Temporary Line Rendering

The `TempConnectionLine` component (`temp-connection-line.tsx:63-114`):
1. Uses `RibbonGeometryBuilder` to create curved bezier line geometry
2. Updates every frame via `useFrame` hook
3. Material configuration (`temp-connection-line.tsx:30-37`):
   ```typescript
   color: 0xffffff
   transparent: true
   opacity: 0.7
   side: THREE.DoubleSide
   ```
4. **No explicit z-position or renderOrder set on the mesh**
5. Relies on z-values from the curve points (start.z = 10, end.z = 10)

#### Z-Position Hierarchy

The codebase uses a sophisticated z-position system (`constants.ts:106-171`):

```typescript
// Z-position ranges in 3D space
- Infinite interaction plane: z = -1
- Edges: z = -0.5 (EDGE_Z_POSITION)
- Block bodies: z = 0 to 400 (MAX_BLOCK_BODY_Z)
- Temp connection line points: z = 10 (hardcoded)
- Selection group: z = 600 (SELECTION_GROUP_Z)
```

Internal block components use tiny z-offsets (0.00001 to 0.00008) relative to their parent block.

### Rendering Order Patterns

#### Pattern 1: Dynamic Z-Position Management
- Blocks use `useActiveNodeZIndex` hook to dynamically adjust z-position
- Dragging/connecting adds MAX_BLOCK_BODY_Z (400) offset to bring to front
- Updated via `useFrame` by setting `group.position.setZ()`

#### Pattern 2: depthWrite Configuration
- Many transparent overlays use `depthWrite: false` to prevent occlusion
- Edges, connection handles, selection boxes all disable depth writing
- **TempConnectionLine does NOT set depthWrite** - uses default (true)

#### Pattern 3: RenderOrder Property
- Some components use `renderOrder` for explicit draw order
- Marquee selection uses `renderOrder: 2`
- **TempConnectionLine does NOT set renderOrder**

### Issue Analysis

The temporary connection line may not always render on top because:

1. **No explicit mesh z-position**: While curve points have z=10, the mesh itself has no position.z set
2. **depthWrite not disabled**: Default depthWrite=true may cause depth buffer conflicts
3. **No renderOrder**: Missing explicit render queue ordering
4. **Transparency sorting**: Three.js sorts transparent objects back-to-front, but without explicit controls

### Comparison with Static Edges

Static edges (`Edge.tsx:228,365`) use:
```typescript
depthWrite: false
transparent: true
position.z = EDGE_Z_POSITION (-0.5)
```

TempConnectionLine differs by not setting depthWrite=false, which could cause rendering inconsistencies.

## Architecture Documentation

### Connection Creation Flow

1. **Initiation**: User clicks ConnectionHandle → `onPointerDown` event
2. **State Setup**: `useConnectionManager.begin()` sets start position with z=10
3. **Dragging**: Canvas `onPointerMove` updates end position via `drag()`
4. **Frame Updates**: `useFrame` in TempConnectionLine recalculates bezier curve
5. **Rendering**: RibbonGeometry updated in-place with new curve points
6. **Completion**: `onPointerUp` → `complete()` → creates Edge or shows add menu

### Geometry Management

The system uses object pooling and in-place updates:
- `RibbonGeometryBuilder` pre-allocates buffers for maximum curve size
- Updates geometry without creating new objects each frame
- Reuses Vector3 instances to minimize garbage collection

### Material System

Components follow consistent material patterns:
- Transparent overlays use `transparent: true` with reduced opacity
- Interactive elements often use `depthWrite: false`
- `DoubleSide` rendering for visibility from both angles

## Historical Context (from thoughts/)

### Related Tickets
- **ENG-1084** (`thoughts/shared/tickets/ENG-1084.md`): Current ticket about TempConnectionLine rendering
- **ENG-929** (`thoughts/shared/tickets/ENG-929.md`): Undo-redo implementation for R3F
- **ENG-1057** (`thoughts/shared/tickets/ENG-1057.md`): Video loading issues in R3F canvas

### Previous Research
- Multiple research documents about video loading and shader implementation in R3F
- Analysis of opacity mixing patterns and dual texture blending
- Performance and reliability analysis of R3F canvas rendering

The research shows the R3F canvas has evolved from ReactFlow, with ongoing work to optimize rendering patterns and fix edge cases like the temporary connection line visibility.

## Related Research

- `thoughts/shared/research/2025-10-14-ENG-1057-video-loading-r3f-canvas.md` - R3F canvas architecture overview
- `thoughts/shared/research/2025-10-17-ENG-1057-opacity-mixing-patterns.md` - Material opacity patterns
- `thoughts/shared/research/2025-10-16-codebase-coupling-and-decoupling-opportunities.md` - R3F architecture analysis

## Open Questions

1. Should TempConnectionLine use `depthWrite: false` like other transparent overlays?
2. Would setting explicit `position.z` on the mesh improve consistency?
3. Should renderOrder be used in addition to z-position?
4. Are there edge cases where blocks with z > 10 could occlude the connection line?

## Code References

Key files for implementing the fix:
- `src/components/r3f/temp-connection-line.tsx:30-37` - Material configuration
- `src/components/r3f/temp-connection-line.tsx:115` - Mesh rendering
- `src/hooks/use-connection-manager.tsx:85` - Z-position setting
- `src/components/r3f/constants.ts:106-171` - Z-position constants