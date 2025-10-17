# Layered Materials Loading Implementation Plan

## Overview

Implement a visual loading indicator for video blocks that displays both thumbnail and skeleton overlays simultaneously using a layered materials approach, allowing smooth opacity transitions between loading states.

## Current State Analysis

The video loading system currently uses a three-stage sequential pattern:
1. **Thumbnail stage**: Displays static thumbnail image
2. **Skeleton stage**: Shows pulsing gray placeholder when loading starts
3. **Video stage**: Shows the loaded video texture

Each stage completely replaces the previous material, with no visual transition between states. The research document (thoughts/shared/research/2025-10-17-ENG-1057-opacity-mixing-patterns.md) identified Option 2 (Layered Materials) as the preferred approach for creating a mixed loading state.

### Key Discoveries:
- Current implementation uses conditional returns to swap materials (`video-material.tsx:215-240`)
- All materials share the same shader structure with rounded corners (`video-and-image-shaders.ts`)
- FadeTransitionGroup pattern exists for opacity animations (`fade-transition-group.tsx`)
- Z-layering pattern is established with BLOCK_Z constants (`constants.ts`)
- Material isolation pattern exists to avoid mutating shared instances

## Desired End State

After implementation, the video loading experience will:
- Display the thumbnail as a base layer that remains visible
- Overlay a semi-transparent skeleton animation when loading begins
- Smoothly fade out the skeleton overlay as the video loads
- Provide visual continuity throughout the loading process

### Verification:
- Thumbnail remains visible during loading
- Skeleton overlay appears with smooth fade-in animation
- Both layers display correctly with proper transparency
- Skeleton fades out smoothly when video is ready
- No z-fighting or rendering artifacts

## What We're NOT Doing

- NOT modifying the shader code to handle multiple textures
- NOT changing the existing three-stage state management
- NOT affecting the video playback functionality
- NOT changing how thumbnails are loaded
- NOT modifying the skeleton pulsing animation
- NOT implementing cross-fade between thumbnail and video (video replaces entire group)

## Implementation Approach

Use a layered mesh approach where the thumbnail and skeleton are rendered as separate meshes at different z-positions within a group. This allows independent opacity control for each layer while maintaining the existing material structure.

## Phase 1: Create Layered Video Material Component

### Overview
Refactor VideoMaterial to render thumbnail and skeleton as overlapping layers instead of sequential replacements.

### Changes Required:

#### 1. Update VideoMaterial Component Structure
**File**: `src/components/r3f/blocks/video/video-material.tsx`
**Changes**: Replace conditional returns with layered group structure

```tsx
// Replace lines 215-240 with:
return (
  <group>
    {/* Base layer: Thumbnail or Video */}
    {(shouldDisplayThumbnail || (!texture && thumbnailUrl)) && (
      <mesh>
        <planeGeometry args={[displayWidth, displayHeight]} />
        <ThumbnailImageMaterial
          thumbnailUrl={thumbnailUrl}
          displayWidth={displayWidth}
          displayHeight={displayHeight}
        />
      </mesh>
    )}

    {/* Skeleton overlay layer */}
    {isLoading && !texture && (
      <mesh position-z={0.001}>
        <planeGeometry args={[displayWidth, displayHeight]} />
        <SkeletonMaterial
          width={displayWidth}
          height={displayHeight}
          radiusX={BLOCK_BORDER_RADIUS}
          radiusY={BLOCK_BORDER_RADIUS}
          opacity={skeletonOpacity}
        />
      </mesh>
    )}

    {/* Video layer (replaces entire group when loaded) */}
    {texture && !isLoading && (
      <mesh>
        <planeGeometry args={[displayWidth, displayHeight]} />
        <VideoShaderMaterial
          texture={texture}
          displayWidth={displayWidth}
          displayHeight={displayHeight}
        />
      </mesh>
    )}
  </group>
);
```

#### 2. Add Skeleton Opacity State Management
**File**: `src/components/r3f/blocks/video/video-material.tsx`
**Changes**: Add opacity state and animation logic after line 43

```tsx
// Add new state for skeleton opacity
const [skeletonOpacity, setSkeletonOpacity] = useState(0);
const skeletonOpacityRef = useRef(0);

// Add useFrame for smooth opacity transitions
useFrame((_, delta) => {
  const targetOpacity = isLoading && !texture ? 0.5 : 0;
  const lambda = Math.log(20) / OPACITY_LERP_T; // Use existing constant

  skeletonOpacityRef.current = THREE.MathUtils.damp(
    skeletonOpacityRef.current,
    targetOpacity,
    lambda,
    delta
  );

  // Only update state if changed significantly
  if (Math.abs(skeletonOpacityRef.current - skeletonOpacity) > 0.01) {
    setSkeletonOpacity(skeletonOpacityRef.current);
  }
});
```

#### 3. Import Required Dependencies
**File**: `src/components/r3f/blocks/video/video-material.tsx`
**Changes**: Add imports at top of file

```tsx
import { useFrame } from "@react-three/fiber";
import * as THREE from "three";
import { OPACITY_LERP_T } from "@/components/r3f/constants";
```

### Success Criteria:

#### Automated Verification:
- [ ] Type checking passes: `pnpm typecheck`
- [ ] Build completes successfully: `pnpm build`
- [ ] No linting errors: `pnpm lint`

#### Manual Verification:
- [ ] Thumbnail displays immediately when video block loads
- [ ] Skeleton overlay appears when loading starts
- [ ] Both layers visible simultaneously during loading
- [ ] No z-fighting or flickering between layers
- [ ] Proper transparency rendering (no black backgrounds)

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation from the human that the manual testing was successful before proceeding to the next phase.

---

## Phase 2: Update Skeleton Material for Opacity Control

### Overview
Modify SkeletonMaterial to accept an external opacity prop that works with the pulsing animation.

### Changes Required:

#### 1. Add Opacity Prop to SkeletonMaterial
**File**: `src/components/r3f/shaders/skeleton.tsx`
**Changes**: Update component props interface around line 10

```tsx
interface SkeletonMaterialProps {
  width: number;
  height: number;
  radiusX?: number;
  radiusY?: number;
  opacity?: number; // Add new prop
}

export function SkeletonMaterial({
  width,
  height,
  radiusX = SKELETON_DEFAULT_RADIUS_X,
  radiusY = SKELETON_DEFAULT_RADIUS_Y,
  opacity = 1, // Default to fully opaque
}: SkeletonMaterialProps) {
```

#### 2. Update Shader Uniforms
**File**: `src/components/r3f/shaders/skeleton.tsx`
**Changes**: Add base opacity uniform around line 85

```tsx
uniforms={{
  radiusX: { value: radiusX / width },
  radiusY: { value: radiusY / height },
  backgroundColor: { value: new THREE.Color(0.55, 0.55, 0.55) },
  time: { value: 0 },
  baseOpacity: { value: opacity }, // Add base opacity
}}
```

#### 3. Update Fragment Shader
**File**: `src/components/r3f/shaders/skeleton.tsx`
**Changes**: Modify fragment shader to use base opacity around line 25

```glsl
uniform vec3 backgroundColor;
uniform float time;
uniform float baseOpacity; // Add uniform declaration

void main() {
  // ... existing corner calculation code ...

  // Pulse calculation with base opacity
  float pulse = 0.4 + 0.2 * sin(time * 3.0);
  gl_FragColor = vec4(backgroundColor, pulse * alpha * baseOpacity);
  //                                                    ^^^^^^^^^^^^ multiply by base
}
```

#### 4. Update Opacity in useFrame
**File**: `src/components/r3f/shaders/skeleton.tsx`
**Changes**: Update base opacity in animation loop around line 71

```tsx
useFrame((_, delta) => {
  timeRef.current += delta;
  if (material) {
    material.uniforms.time.value = timeRef.current;
    material.uniforms.baseOpacity.value = opacity; // Update base opacity
    invalidate();
  }
});
```

### Success Criteria:

#### Automated Verification:
- [ ] Type checking passes: `pnpm typecheck`
- [ ] Component tests pass: `pnpm test:unit`
- [ ] No shader compilation errors in browser console

#### Manual Verification:
- [ ] Skeleton material accepts opacity prop
- [ ] Pulsing animation still works with custom opacity
- [ ] Opacity multiplies correctly (base * pulse * alpha)
- [ ] Smooth transitions when opacity changes

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation from the human that the manual testing was successful before proceeding to the next phase.

---

## Phase 3: Integrate FadeTransitionGroup for Smooth Transitions

### Overview
Wrap the skeleton overlay in FadeTransitionGroup for smooth fade-in/out animations.

### Changes Required:

#### 1. Wrap Skeleton Layer with FadeTransitionGroup
**File**: `src/components/r3f/blocks/video/video-material.tsx`
**Changes**: Update the skeleton overlay section

```tsx
import { FadeTransitionGroup } from "@/components/r3f/fade-transition-group";

// In the return statement, replace skeleton mesh with:
{isLoading && !texture && (
  <FadeTransitionGroup
    fadeIn={true}
    duration={0.3}
    isolateMaterials={false}
  >
    <mesh position-z={0.001}>
      <planeGeometry args={[displayWidth, displayHeight]} />
      <SkeletonMaterial
        width={displayWidth}
        height={displayHeight}
        radiusX={BLOCK_BORDER_RADIUS}
        radiusY={BLOCK_BORDER_RADIUS}
      />
    </mesh>
  </FadeTransitionGroup>
)}
```

#### 2. Remove Manual Opacity Animation
**File**: `src/components/r3f/blocks/video/video-material.tsx`
**Changes**: Remove the useFrame hook and opacity state added in Phase 1

```tsx
// Remove these lines:
// const [skeletonOpacity, setSkeletonOpacity] = useState(0);
// const skeletonOpacityRef = useRef(0);
// useFrame hook for opacity animation
```

### Success Criteria:

#### Automated Verification:
- [ ] Type checking passes: `pnpm typecheck`
- [ ] Build succeeds: `pnpm build`
- [ ] No React warnings about unmounting during animations

#### Manual Verification:
- [ ] Skeleton fades in smoothly over 0.3 seconds
- [ ] Skeleton fades out when video loads
- [ ] No abrupt transitions or flashing
- [ ] FadeTransitionGroup properly manages material properties

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation from the human that the manual testing was successful before proceeding to the next phase.

---

## Phase 4: Handle Edge Cases and Cleanup

### Overview
Address edge cases like missing thumbnails and add proper cleanup for animations.

### Changes Required:

#### 1. Handle Missing Thumbnail Case
**File**: `src/components/r3f/blocks/video/video-material.tsx`
**Changes**: Update base layer condition

```tsx
{/* Base layer: Show skeleton if no thumbnail available */}
{(shouldDisplayThumbnail || (!texture && !thumbnailUrl)) && (
  thumbnailUrl ? (
    <mesh>
      <planeGeometry args={[displayWidth, displayHeight]} />
      <ThumbnailImageMaterial
        thumbnailUrl={thumbnailUrl}
        displayWidth={displayWidth}
        displayHeight={displayHeight}
      />
    </mesh>
  ) : (
    <mesh>
      <planeGeometry args={[displayWidth, displayHeight]} />
      <SkeletonMaterial
        width={displayWidth}
        height={displayHeight}
        radiusX={BLOCK_BORDER_RADIUS}
        radiusY={BLOCK_BORDER_RADIUS}
        opacity={0.3} // Lower opacity for base skeleton
      />
    </mesh>
  )
)}

{/* Loading overlay: Only show if we have a thumbnail */}
{isLoading && !texture && thumbnailUrl && (
  <FadeTransitionGroup fadeIn={true} duration={0.3}>
    <mesh position-z={0.001}>
      <planeGeometry args={[displayWidth, displayHeight]} />
      <SkeletonMaterial
        width={displayWidth}
        height={displayHeight}
        radiusX={BLOCK_BORDER_RADIUS}
        radiusY={BLOCK_BORDER_RADIUS}
        opacity={0.5} // Higher opacity for overlay
      />
    </mesh>
  </FadeTransitionGroup>
)}
```

#### 2. Add Z-Index Constant
**File**: `src/components/r3f/constants.ts`
**Changes**: Add constant for skeleton overlay z-position

```tsx
export const BLOCK_Z = {
  // ... existing constants ...
  SKELETON_OVERLAY: 0.001, // Add new constant
  // ... rest of constants ...
};
```

#### 3. Use Z-Index Constant
**File**: `src/components/r3f/blocks/video/video-material.tsx`
**Changes**: Replace hardcoded z-position

```tsx
import { BLOCK_Z } from "@/components/r3f/constants";

// In skeleton overlay mesh:
<mesh position-z={BLOCK_Z.SKELETON_OVERLAY}>
```

### Success Criteria:

#### Automated Verification:
- [ ] All tests pass: `pnpm test:unit`
- [ ] Type checking passes: `pnpm typecheck`
- [ ] Linting passes: `pnpm lint`
- [ ] Build succeeds: `pnpm build`

#### Manual Verification:
- [ ] Videos without thumbnails show single skeleton
- [ ] Videos with thumbnails show layered loading state
- [ ] No memory leaks from animation loops
- [ ] Proper cleanup when component unmounts
- [ ] Z-ordering is consistent across different viewport sizes

---

## Testing Strategy

### Unit Tests:
- Test VideoMaterial renders correct layers based on state
- Test SkeletonMaterial accepts and applies opacity prop
- Test FadeTransitionGroup integration with skeleton overlay
- Test edge cases (no thumbnail, immediate video load)

### Integration Tests:
- Test full loading sequence from thumbnail to video
- Test interrupting load (navigate away during loading)
- Test multiple video blocks loading simultaneously
- Test with different video sizes and aspect ratios

### Manual Testing Steps:
1. Create a new video block with a thumbnail URL
2. Trigger video loading and observe thumbnail + skeleton overlay
3. Verify skeleton pulses while maintaining base transparency
4. Confirm skeleton fades out when video loads
5. Test with video that has no thumbnail
6. Test with very fast loading video (cached)
7. Test with slow loading video (large file)
8. Check performance with multiple video blocks

## Performance Considerations

- Two mesh layers during loading (minimal overhead)
- FadeTransitionGroup manages its own render invalidation
- Skeleton animation continues but is hidden when opacity is 0
- No additional draw calls after video loads (single mesh)
- Material isolation disabled to reuse shared materials

## Migration Notes

No migration needed as this is an enhancement to the existing loading experience. The change is backward compatible and doesn't affect stored data or API contracts.

## References

- Original research: `thoughts/shared/research/2025-10-17-ENG-1057-opacity-mixing-patterns.md`
- Current video material: `src/components/r3f/blocks/video/video-material.tsx`
- Skeleton material: `src/components/r3f/shaders/skeleton.tsx`
- FadeTransitionGroup: `src/components/r3f/fade-transition-group.tsx:57-201`
- Similar layering pattern: `src/components/r3f/shared-block-body.tsx:264-505`