---
date: 2025-10-17T12:22:52-04:00
researcher: Matan Shavit
git_commit: 6cb69a620aa81155976da65addc49646f6274519
branch: eng-1057-separate-material
repository: eng-1057
topic: "Visual loading indicator with thumbnail and skeleton mixed using opacity"
tags: [research, codebase, r3f, opacity, shaders, material-mixing, loading-states]
status: complete
last_updated: 2025-10-17
last_updated_by: Matan Shavit
---

# Research: Visual Loading Indicator with Thumbnail and Skeleton Mixed Using Opacity

**Date**: 2025-10-17T12:22:52-04:00
**Researcher**: Matan Shavit
**Git Commit**: 6cb69a620aa81155976da65addc49646f6274519
**Branch**: eng-1057-separate-material
**Repository**: eng-1057

## Research Question
How to implement a visual loading indicator when video starts loading but the thumbnail image texture is displayed, showing the thumbnail image and skeleton mixed together with opacity on one or both.

## Summary
The current implementation in the last commit uses a **texture switching pattern** rather than opacity mixing. The video material component transitions from thumbnail → skeleton → video using conditional rendering. To achieve the desired mixed loading state, the codebase provides several patterns for opacity blending that could be adapted.

## Key Findings

### Current Implementation (Last Commit)

#### 1. Three-Stage Loading Pattern
The video material currently implements sequential states:
```typescript
// Stage 1: Show thumbnail
if (shouldDisplayThumbnail) {
  return <ThumbnailImageMaterial ... />
}

// Stage 2: Show skeleton while video loads
if (isLoading || !texture) {
  return <SkeletonMaterial ... />
}

// Stage 3: Show video
return <VideoShaderMaterial texture={texture} ... />
```

All three materials use the same shader (`VideoShaderMaterial`), enabling seamless transitions but not concurrent display.

#### 2. Shader Structure
The current shader accepts a single texture uniform:
```glsl
uniform sampler2D map;
uniform float radiusX;
uniform float radiusY;

void main() {
  // ... rounded corner calculations ...
  vec4 texColor = texture2D(map, uv);
  gl_FragColor = texColor * alpha;
}
```

### Opacity Mixing Patterns Found

#### Pattern 1: Dual Texture Blending in Fragment Shader
**Approach**: Modify the shader to accept two textures and a mix factor
```glsl
uniform sampler2D thumbnailMap;
uniform sampler2D skeletonMap; // Or generate procedurally
uniform float mixFactor; // 0 = thumbnail only, 1 = skeleton only

void main() {
  vec4 thumbnailColor = texture2D(thumbnailMap, uv);

  // Generate skeleton effect procedurally
  float time = uTime; // Add time uniform
  float pulse = 0.4 + 0.2 * sin(time * 3.0);
  vec4 skeletonColor = vec4(0.55, 0.55, 0.55, pulse);

  // Mix the two
  vec4 finalColor = mix(thumbnailColor, skeletonColor, mixFactor);
  gl_FragColor = finalColor * alpha;
}
```

#### Pattern 2: Layered Meshes with Different Opacities
**Found in**: FadeTransitionGroup pattern (`src/components/r3f/fade-transition-group.tsx`)

Create two overlapping meshes:
```tsx
<group>
  {/* Thumbnail mesh */}
  <mesh position-z={0}>
    <planeGeometry args={[width, height]} />
    <shaderMaterial
      uniforms={{ map: { value: thumbnailTexture }, opacity: { value: 0.7 } }}
      transparent
    />
  </mesh>

  {/* Skeleton overlay */}
  <mesh position-z={0.001}>
    <planeGeometry args={[width, height]} />
    <SkeletonMaterial
      width={width}
      height={height}
      opacity={0.3}
    />
  </mesh>
</group>
```

#### Pattern 3: Modified Skeleton Material with Texture Background
**Approach**: Enhance SkeletonMaterial to accept an optional background texture
```glsl
uniform sampler2D backgroundMap; // Optional thumbnail
uniform bool hasBackground;
uniform float backgroundOpacity;
uniform float skeletonOpacity;
uniform float time;

void main() {
  // ... corner calculations ...

  vec4 finalColor = vec4(0.0);

  if (hasBackground) {
    vec4 bgColor = texture2D(backgroundMap, uv);
    finalColor = bgColor * backgroundOpacity;
  }

  // Add skeleton pulse on top
  float pulse = 0.4 + 0.2 * sin(time * 3.0);
  vec3 skeletonColor = vec3(0.55, 0.55, 0.55);

  // Blend skeleton over background
  finalColor = mix(
    finalColor,
    vec4(skeletonColor, 1.0),
    pulse * skeletonOpacity
  );

  gl_FragColor = finalColor * alpha;
}
```

### Existing Patterns for Reference

#### Opacity Animation Pattern (from FadeTransitionGroup)
```typescript
useFrame((_, delta) => {
  const lambda = duration > 0 ? Math.log(20) / duration : Infinity;
  const newOpacity = THREE.MathUtils.damp(current, targetOpacity, lambda, delta);
  material.opacity = newOpacity;
});
```

#### Material State Tracking Pattern
```typescript
const [displayState, setDisplayState] = useState<'thumbnail' | 'loading' | 'video'>('thumbnail');
const [loadingOpacity, setLoadingOpacity] = useState(0);

useEffect(() => {
  if (texture) {
    setDisplayState('video');
  } else if (shouldLoadVideo) {
    setDisplayState('loading');
  }
}, [texture, shouldLoadVideo]);

useFrame((_, delta) => {
  if (displayState === 'loading') {
    // Fade in skeleton overlay
    setLoadingOpacity(THREE.MathUtils.damp(loadingOpacity, 0.5, 3, delta));
  } else {
    // Fade out skeleton overlay
    setLoadingOpacity(THREE.MathUtils.damp(loadingOpacity, 0, 3, delta));
  }
});
```

## Recommended Implementation Approach

### Option 1: Enhanced VideoShaderMaterial (Recommended)
Modify the existing video shader to support dual texture/effect blending:

1. **Update shader uniforms**:
```typescript
uniforms={{
  map: { value: displayTexture },
  showSkeleton: { value: isLoading ? 1.0 : 0.0 },
  skeletonOpacity: { value: 0.5 },
  time: { value: 0 },
  radiusX: { value: BLOCK_BORDER_RADIUS / displayWidth },
  radiusY: { value: BLOCK_BORDER_RADIUS / displayHeight },
}}
```

2. **Modify fragment shader**:
```glsl
uniform sampler2D map;
uniform float showSkeleton;
uniform float skeletonOpacity;
uniform float time;

void main() {
  vec4 texColor = texture2D(map, uv);

  // Skeleton pulse effect
  float pulse = 0.4 + 0.2 * sin(time * 3.0);
  vec4 skeletonColor = vec4(0.55, 0.55, 0.55, pulse);

  // Mix based on loading state
  vec4 finalColor = mix(
    texColor,
    mix(texColor, skeletonColor, skeletonOpacity),
    showSkeleton
  );

  gl_FragColor = finalColor * alpha;
}
```

3. **Update time uniform in component**:
```typescript
useFrame((_, delta) => {
  if (materialRef.current && isLoading) {
    materialRef.current.uniforms.time.value += delta;
    materialRef.current.uniforms.showSkeleton.value =
      THREE.MathUtils.damp(materialRef.current.uniforms.showSkeleton.value, 1.0, 3, delta);
  }
});
```

### Option 2: Layered Approach
Keep materials separate but render both simultaneously:

```tsx
return (
  <group>
    <mesh>
      <planeGeometry args={[displayWidth, displayHeight]} />
      <VideoShaderMaterial
        texture={thumbnailTexture}
        displayWidth={displayWidth}
        displayHeight={displayHeight}
      />
    </mesh>

    {isLoading && (
      <FadeTransitionGroup fadeIn={true} duration={0.3}>
        <mesh position-z={0.001}>
          <planeGeometry args={[displayWidth, displayHeight]} />
          <SkeletonMaterial
            width={displayWidth}
            height={displayHeight}
            radiusX={BLOCK_BORDER_RADIUS}
            radiusY={BLOCK_BORDER_RADIUS}
            opacity={0.5}
          />
        </mesh>
      </FadeTransitionGroup>
    )}
  </group>
);
```

## Files to Modify

### For Option 1 (Shader Modification):
1. `/src/components/r3f/shaders/video-and-image-shaders.ts` - Add skeleton blending to fragment shader
2. `/src/components/r3f/blocks/video/video-shader-material.tsx` - Add new uniforms
3. `/src/components/r3f/blocks/video/video-material.tsx` - Update loading state logic

### For Option 2 (Layered Approach):
1. `/src/components/r3f/blocks/video/video-material.tsx` - Change return structure
2. `/src/components/r3f/shaders/skeleton.tsx` - Add opacity uniform support if needed

## Performance Considerations

1. **Option 1** (Shader mixing) is more performant - single draw call
2. **Option 2** (Layered meshes) is simpler but requires two draw calls
3. Both approaches should use `invalidate()` to trigger renders only when needed
4. Consider using `THREE.MathUtils.damp()` for smooth transitions

## Constants and Utilities

```typescript
// From constants.ts
export const OPACITY_LERP_T = 0.25;
export const LOADING_PROGRESS_ANIMATION_THRESHOLD = 0.001;

// Skeleton color
const backgroundColor = new THREE.Color(0.55, 0.55, 0.55);

// Pulse timing
const PULSE_FREQUENCY = 3.0; // sin(time * 3.0)
const PULSE_AMPLITUDE = 0.2;
const PULSE_BASE = 0.4;
```

## Related Research Documents

- `2025-10-17-ENG-1057-shader-texture-switching-issue.md` - Technical details on texture switching
- `2025-01-17-ENG-1057-shader-texture-switching-fix.md` - Fix implementation patterns
- `2025-10-16-ENG-1057-video-thumbnail-display.md` - Two-stage texture loading approach

## Open Questions

1. Should the skeleton opacity be configurable or fixed?
2. Should the transition timing match existing `OPACITY_LERP_T` constant?
3. Should the skeleton pulse continue during the mixed state or be static?
4. What opacity values work best visually (0.3/0.7, 0.5/0.5)?