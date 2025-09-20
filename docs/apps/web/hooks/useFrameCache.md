# useFrameCache Hook

A React hook that provides intelligent frame caching for video editor timeline rendering, optimizing performance by caching rendered frames and implementing smart pre-rendering strategies.

## Overview

The `useFrameCache` hook manages a shared global cache of rendered video frames, avoiding re-rendering identical frames when the timeline state hasn't changed. It includes cache invalidation, pre-rendering, and memory management features.

## Core Concepts

- **Frame Caching**: Frames cached based on timeline state hash at specific time
- **Timeline Hashing**: Deterministic hashes from active elements, project settings, and scene info
- **Cache Management**: LRU eviction, automatic invalidation, shared singleton cache (HMR-safe)

## API

### Types

```typescript
interface CachedFrame {
  imageData: ImageData; // The rendered frame data
  timelineHash: string; // Hash of timeline state when cached
  timestamp: number; // When the frame was cached (for LRU)
}

interface FrameCacheOptions {
  maxCacheSize?: number; // Maximum cached frames (default: 300)
  cacheResolution?: number; // Frames per second to cache (default: 30)
}
```

### Hook Returns

```typescript
{
  isFrameCached: (time, tracks, mediaFiles, activeProject, sceneId?) => boolean;
  getCachedFrame: (time, tracks, mediaFiles, activeProject, sceneId?) => ImageData | null;
  cacheFrame: (time, imageData, tracks, mediaFiles, activeProject, sceneId?) => void;
  invalidateCache: () => void;
  getRenderStatus: (time, tracks, mediaFiles, activeProject, sceneId?) => "cached" | "not-cached";
  preRenderNearbyFrames: (currentTime, tracks, mediaFiles, activeProject, renderFunction, sceneId?, range?) => Promise<void>;
  cacheSize: number;
}
```

## Key Functions

### getTimelineHash

**Purpose**: Generates deterministic hash representing timeline state at specific time.

**Logic Flow**:

1. Iterate through all non-muted tracks
2. Find elements active at given time
3. Extract relevant properties for each element type:
   - **Media elements**: id, type, timing, mediaId
   - **Text elements**: id, type, timing, content, styling, positioning
4. Include project settings (backgroundColor, canvasSize, etc.)
5. Quantize time to cache resolution for consistent hashing
6. Handle element visibility (skip hidden elements) and trimming
7. Include scene-specific information if provided

### isFrameCached

**Purpose**: Checks if frame is cached and still valid.

**Logic Flow**:

1. Calculate frame key using `Math.floor(time * cacheResolution)`
2. Retrieve cached frame from global cache
3. Generate current timeline hash using `getTimelineHash`
4. Compare cached hash with current hash
5. Return `true` if frame exists and is valid, `false` otherwise

### getCachedFrame

**Purpose**: Retrieves cached frame if available and valid.

**Logic Flow**:

1. Calculate frame key using quantized time
2. Retrieve cached frame from global cache
3. Validate timeline hash by comparing with current state
4. Remove stale cache entries if hash doesn't match
5. Log cache misses for debugging
6. Return `ImageData` or `null`

### cacheFrame

**Purpose**: Stores rendered frame in cache with memory management.

**Logic Flow**:

1. Calculate frame key and generate timeline hash
2. Check if cache exceeds `maxCacheSize` limit
3. If limit exceeded, perform LRU eviction:
   - Sort all entries by timestamp
   - Remove oldest 20% of entries
4. Store frame with metadata (imageData, timelineHash, timestamp)
5. Update cache size counter

### preRenderNearbyFrames

**Purpose**: Intelligently pre-renders frames around current time for smooth playback.

**Logic Flow**:

1. Identify uncached frames within specified range
2. Expand to full 1-second buckets to avoid fragmented cache regions
3. Sort frames by proximity to current time (forward-first priority)
4. Cap total scheduled renders to prevent performance issues
5. Use `requestIdleCallback` for non-blocking pre-rendering
6. Execute render function for each uncached frame
7. Cache newly rendered frames

## Integration Example: Preview Panel

The `useFrameCache` hook is used in the `PreviewPanel` component for video editor frame rendering.

### Basic Integration

```typescript
const { getCachedFrame, cacheFrame, invalidateCache, preRenderNearbyFrames } =
  useFrameCache();

// Cache invalidation when background settings change
useEffect(() => {
  invalidateCache();
}, [
  mediaFiles,
  activeProject?.backgroundColor,
  activeProject?.backgroundType,
  invalidateCache,
]);
```

### Frame Rendering Workflow

```typescript
useEffect(() => {
  const draw = async () => {
    const canvas = canvasRef.current;
    if (!canvas) return;

    // Check for cached frame first
    const cachedFrame = getCachedFrame(
      currentTime,
      tracks,
      mediaFiles,
      activeProject,
      currentScene?.id
    );

    if (cachedFrame) {
      // Use cached frame - instant display
      mainCtx.putImageData(cachedFrame, 0, 0);

      // Pre-render nearby frames in background
      preRenderNearbyFrames(
        currentTime,
        tracks,
        mediaFiles,
        activeProject,
        async (time: number) => {
          const tempCanvas = document.createElement("canvas");
          const tempCtx = tempCanvas.getContext("2d");
          await renderTimelineFrame({
            /* render params */
          });
          return tempCtx.getImageData(0, 0, width, height);
        },
        currentScene?.id,
        isPlaying ? 1 : 3 // 1s for playback, 3s for scrubbing
      );
      return;
    }

    // Cache miss - render from scratch
    await renderTimelineFrame({
      /* render params */
    });
    const imageData = offCtx.getImageData(0, 0, width, height);
    cacheFrame(
      currentTime,
      imageData,
      tracks,
      mediaFiles,
      activeProject,
      currentScene?.id
    );
  };

  void draw();
}, [activeElements, currentTime, previewDimensions /* other deps */]);
```

### Performance Optimizations

- **FPS Throttling**: Skip rendering if too soon during playback
- **Canvas Resolution**: Set internal resolution to avoid blurry scaling
- **Offscreen Canvas**: Use for rendering to avoid blocking main thread
- **Pre-rendering Strategy**: 3s range for scrubbing, 1s for playback

## Usage Examples

### Basic Frame Caching

```typescript
const { isFrameCached, getCachedFrame, cacheFrame } = useFrameCache();

if (isFrameCached(time, tracks, mediaFiles, project)) {
  const cachedFrame = getCachedFrame(time, tracks, mediaFiles, project);
  // Use cached frame
} else {
  const imageData = await renderFrame(time);
  cacheFrame(time, imageData, tracks, mediaFiles, project);
}
```

### Pre-rendering

```typescript
const { preRenderNearbyFrames } = useFrameCache();

await preRenderNearbyFrames(
  currentTime,
  tracks,
  mediaFiles,
  project,
  renderFunction,
  sceneId,
  3
);
```

### Cache Management

```typescript
const { invalidateCache, cacheSize } = useFrameCache();

invalidateCache(); // Clear cache when timeline changes
console.log(`Cache size: ${cacheSize} frames`);
```

## Integration Notes

- **Timeline Store**: Works with `TimelineTrack[]`, `MediaFile[]`, and `TProject`
- **Element Types**: Media elements (ID + timing), text elements (all properties)
- **Scene Support**: Optional `sceneId` for per-scene cache isolation
