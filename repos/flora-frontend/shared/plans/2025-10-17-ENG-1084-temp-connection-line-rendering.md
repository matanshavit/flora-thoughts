# TempConnectionLine Z-Index Fix Implementation Plan

## Overview

Fix the temporary connection line rendering issue where it gets occluded by blocks during drag operations by elevating its z-position above the maximum block z-index and adding proper depth write configuration.

## Current State Analysis

The TempConnectionLine component currently renders at z=10 (hardcoded in `use-connection-manager.tsx`), while blocks being dragged are elevated to z=400+ (MAX_BLOCK_BODY_Z). This causes the connection line to appear behind blocks during drag operations, making it difficult to see where connections are being made.

### Key Discoveries:
- Connection line z-position is hardcoded to 10 (`src/hooks/use-connection-manager.tsx:62,85`)
- Dragged blocks are elevated by MAX_BLOCK_BODY_Z (400) during drag/connect operations (`src/components/r3f/hooks/use-z-index-updates.tsx:25`)
- TempConnectionLine material lacks `depthWrite: false` unlike other overlay components (`src/components/r3f/temp-connection-line.tsx:30-37`)
- Successful overlay patterns use high z-position + depthWrite=false (SelectionRenderer at z=600)

## Desired End State

After implementation, the temporary connection line should:
- Always render on top of all blocks, including those being dragged
- Use a z-position that respects the existing z-hierarchy constants
- Follow established rendering patterns for overlay components
- Maintain smooth visual appearance during connection creation

### Verification:
- Drag a block and attempt to create a connection - the line should appear on top
- Connect between blocks at different z-indices - line should always be visible
- Test with multiple overlapping blocks - line should render above all of them

## What We're NOT Doing

- Changing the z-position system for blocks
- Modifying how blocks are elevated during drag operations
- Altering the connection creation flow or interaction patterns
- Changing the visual appearance of the connection line (color, opacity, thickness)

## Implementation Approach

We'll use a dynamic z-position based on MAX_BLOCK_BODY_Z to ensure the connection line always renders above dragged blocks. This approach:
1. Defines a new constant TEMP_CONNECTION_Z = MAX_BLOCK_BODY_Z + 100 (z=500)
2. Updates the hardcoded z-values to use this constant
3. Adds depthWrite=false to follow overlay component patterns
4. Optionally adds renderOrder for additional guarantee

## Phase 1: Define Z-Position Constant

### Overview
Add a new constant for the temporary connection line z-position that dynamically positions it above the maximum block z-index.

### Changes Required:

#### 1. Add Constant to constants.ts
**File**: `src/components/r3f/constants.ts`
**Changes**: Add new constant after MAX_BLOCK_BODY_Z definition

```typescript
export const MAX_BLOCK_BODY_Z = 400;
export const SELECTION_GROUP_Z = 600;
// Add this new constant
export const TEMP_CONNECTION_Z = MAX_BLOCK_BODY_Z + 100; // z=500, above dragged blocks but below selection
```

### Success Criteria:

#### Automated Verification:
- [ ] Type checking passes: `pnpm typecheck`
- [ ] No linting errors: `pnpm lint`

#### Manual Verification:
- [ ] Constant is properly exported and accessible

---

## Phase 2: Update Connection Points Z-Position

### Overview
Replace the hardcoded z=10 values in use-connection-manager.tsx with the new TEMP_CONNECTION_Z constant.

### Changes Required:

#### 1. Import the constant
**File**: `src/hooks/use-connection-manager.tsx`
**Changes**: Add import at the top of the file

```typescript
import { TEMP_CONNECTION_Z } from "@/components/r3f/constants";
```

#### 2. Update snap target position
**File**: `src/hooks/use-connection-manager.tsx`
**Changes**: Update line 62

```typescript
// Old code:
tempVec.set(node.position.x + offset, node.position.y, 10);

// New code:
tempVec.set(node.position.x + offset, node.position.y, TEMP_CONNECTION_Z);
```

#### 3. Update connection start position
**File**: `src/hooks/use-connection-manager.tsx`
**Changes**: Update line 85

```typescript
// Old code:
ref.current.start.set(node.position.x + offset, node.position.y, 10);

// New code:
ref.current.start.set(node.position.x + offset, node.position.y, TEMP_CONNECTION_Z);
```

### Success Criteria:

#### Automated Verification:
- [ ] Type checking passes: `pnpm typecheck`
- [ ] Build succeeds: `pnpm build`
- [ ] Unit tests pass: `pnpm test:unit`

#### Manual Verification:
- [ ] Connection line starts at the correct z-position when dragging begins
- [ ] End position updates correctly during drag

---

## Phase 3: Update Material Properties

### Overview
Add depthWrite=false to the TempConnectionLine material and optionally add renderOrder for additional rendering guarantee.

### Changes Required:

#### 1. Update material configuration
**File**: `src/components/r3f/temp-connection-line.tsx`
**Changes**: Update material creation (lines 30-37)

```typescript
const material = useMemo(() => {
  return new THREE.MeshBasicMaterial({
    color: 0xffffff,
    transparent: true,
    opacity: 0.7,
    side: THREE.DoubleSide,
    depthWrite: false,  // Add this line
  });
}, []);
```

#### 2. Add renderOrder to mesh (optional but recommended)
**File**: `src/components/r3f/temp-connection-line.tsx`
**Changes**: Update mesh component (lines 123-126)

```typescript
return (
  <mesh
    ref={meshRef}
    geometry={geometry}
    renderOrder={500}  // Add this line - ensures render priority
  >
    <primitive
      object={material}
      attach="material"
    />
  </mesh>
);
```

### Success Criteria:

#### Automated Verification:
- [ ] Type checking passes: `pnpm typecheck`
- [ ] Linting passes: `pnpm lint`
- [ ] Build completes successfully: `pnpm build`

#### Manual Verification:
- [ ] Connection line renders on top of all blocks during drag
- [ ] Line remains visible when dragging over multiple overlapping blocks
- [ ] No z-fighting or flickering issues
- [ ] Line transparency blends correctly with blocks below

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation from the human that the manual testing was successful before considering the implementation complete.

---

## Testing Strategy

### Unit Tests:
- Verify TEMP_CONNECTION_Z constant is properly defined and exported
- Test that connection positions use the correct z-value

### Integration Tests:
- Test connection creation flow works as before
- Verify connections complete successfully

### Manual Testing Steps:
1. Start the development server (`pnpm dev`)
2. Create multiple blocks on the canvas
3. Drag one block to overlap others
4. Start creating a connection from the dragged block
5. Verify the connection line appears on top of all blocks
6. Test with blocks at various z-indices
7. Create connections between overlapping blocks
8. Verify no visual artifacts or z-fighting

## Performance Considerations

The changes have minimal performance impact:
- Z-position change doesn't affect rendering performance
- depthWrite=false can slightly improve performance for transparent objects
- renderOrder doesn't add overhead for single mesh

## Migration Notes

No migration needed - this is a rendering-only change that doesn't affect data or saved state.

## References

- Original ticket: `thoughts/shared/tickets/ENG-1084.md`
- Related research: `thoughts/shared/research/2025-10-17-ENG-1084-r3f-temp-connection-line-rendering.md`
- Pattern examples: Edge component at `src/components/r3f/Edge.tsx:365`
- Z-position constants: `src/components/r3f/constants.ts:111-115`