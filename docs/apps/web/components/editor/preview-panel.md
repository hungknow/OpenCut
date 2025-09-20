# PreviewPanel Component

A comprehensive video editor preview component that handles frame rendering, caching, audio playback, and user interactions for the video editor timeline.

## Overview

The `PreviewPanel` component is the central preview interface that displays the current frame of the video timeline with real-time rendering, intelligent frame caching, and audio synchronization. It supports both normal and fullscreen modes with drag-and-drop text element editing.

## Core Architecture

### Component Structure

```typescript
PreviewPanel
├── Preview Container (responsive sizing)
├── Canvas Element (frame rendering)
├── Layout Guide Overlay
├── Preview Toolbar (controls)
└── Fullscreen Mode (expanded view)
```

## Logic Groups

### 1. Preview Sizing and Layout

**Purpose**: Responsive preview sizing that maintains aspect ratio and adapts to container dimensions.

**Key Logic**:

- Calculates available space considering padding, gaps, and toolbar height
- Maintains canvas aspect ratio (`canvasSize.width / canvasSize.height`)
- Supports both normal and fullscreen modes
- Uses ResizeObserver for dynamic resizing

**Related React Code**:

```typescript
// State for preview dimensions
const [previewDimensions, setPreviewDimensions] = useState({
  width: 0,
  height: 0,
});

// Responsive sizing calculation
useEffect(() => {
  const updatePreviewSize = () => {
    if (!containerRef.current) return;

    let availableWidth, availableHeight;

    if (isExpanded) {
      // Fullscreen mode: use window dimensions
      const controlsHeight = 80;
      const marginSpace = 24;
      availableWidth = window.innerWidth - marginSpace;
      availableHeight = window.innerHeight - controlsHeight - marginSpace;
    } else {
      // Normal mode: use container dimensions
      const container = containerRef.current.getBoundingClientRect();
      // ... calculate available space considering padding, gaps, toolbar
    }

    // Maintain aspect ratio
    const targetRatio = canvasSize.width / canvasSize.height;
    const containerRatio = availableWidth / availableHeight;

    if (containerRatio > targetRatio) {
      height = availableHeight * (isExpanded ? 0.95 : 1);
      width = height * targetRatio;
    } else {
      width = availableWidth * (isExpanded ? 0.95 : 1);
      height = width / targetRatio;
    }

    setPreviewDimensions({ width, height });
  };
}, [canvasSize.width, canvasSize.height, isExpanded]);
```

### 2. Frame Rendering and Caching

**Purpose**: Efficient frame rendering with intelligent caching to avoid re-rendering identical frames.

**Key Logic**:

- Uses `useFrameCache` hook for frame caching
- Renders frames on canvas with proper resolution scaling
- Implements FPS throttling during playback
- Uses offscreen canvas for non-blocking rendering
- Pre-renders nearby frames for smooth scrubbing

**Related React Code**:

```typescript
// Frame cache integration
const { getCachedFrame, cacheFrame, invalidateCache, preRenderNearbyFrames } =
  useFrameCache();

// Main rendering effect
useEffect(() => {
  const draw = async () => {
    const canvas = canvasRef.current;
    if (!canvas) return;
    const mainCtx = canvas.getContext("2d");
    if (!mainCtx) return;

    // Set canvas internal resolution to avoid blurry scaling
    const displayWidth = Math.max(1, Math.floor(previewDimensions.width));
    const displayHeight = Math.max(1, Math.floor(previewDimensions.height));
    if (canvas.width !== displayWidth || canvas.height !== displayHeight) {
      canvas.width = displayWidth;
      canvas.height = displayHeight;
    }

    // FPS throttling during playback
    const fps = activeProject?.fps || DEFAULT_FPS;
    const minDelta = 1 / fps;
    if (isPlaying) {
      if (currentTime - lastFrameTimeRef.current < minDelta) {
        return; // Skip rendering if too soon
      }
      lastFrameTimeRef.current = currentTime;
    }

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
      preRenderNearbyFrames(/* ... */);
      return;
    }

    // Cache miss - render from scratch using offscreen canvas
    // ... offscreen rendering logic
  };

  void draw();
}, [activeElements, currentTime, previewDimensions /* other deps */]);
```

### 3. Audio Playback System

**Purpose**: Web Audio API integration for synchronized audio playback during video editing.

**Key Logic**:

- Uses Web Audio API for precise audio timing
- Handles audio buffer decoding and caching
- Manages audio gain for volume/mute control
- Synchronizes audio with timeline playback
- Supports multiple concurrent audio sources

**Related React Code**:

```typescript
// Audio state management
const audioContextRef = useRef<AudioContext | null>(null);
const audioGainRef = useRef<GainNode | null>(null);
const audioBuffersRef = useRef<Map<string, AudioBuffer>>(new Map());
const playingSourcesRef = useRef<Set<AudioBufferSourceNode>>(new Set());

// Audio scheduling effect
useEffect(() => {
  const scheduleNow = async () => {
    await ensureAudioGraph();
    const audioCtx = audioContextRef.current!;
    const gain = audioGainRef.current!;

    // Find audible elements at current time
    const audible: Array<{
      id: string;
      elementStart: number;
      trimStart: number;
      trimEnd: number;
      duration: number;
      muted: boolean;
      trackMuted: boolean;
    }> = [];

    // ... collect audible elements logic

    // Decode audio buffers as needed
    const decodePromises: Array<Promise<void>> = [];
    for (const id of uniqueIds) {
      if (!audioBuffersRef.current.has(id)) {
        const mediaItem = idToMedia.get(id);
        if (!mediaItem) continue;
        const p = (async () => {
          const arr = await mediaItem.file.arrayBuffer();
          const buf = await audioCtx.decodeAudioData(arr.slice(0));
          audioBuffersRef.current.set(id, buf);
        })();
        decodePromises.push(p);
      }
    }
    await Promise.all(decodePromises);

    // Schedule audio playback
    const startAt = audioCtx.currentTime + 0.02;
    for (const entry of audible) {
      if (entry.muted || entry.trackMuted) continue;
      const buffer = audioBuffersRef.current.get(entry.id);
      if (!buffer) continue;

      const src = audioCtx.createBufferSource();
      src.buffer = buffer;
      src.connect(gain);
      src.start(startAt, localTime, playDuration);
      playingSourcesRef.current.add(src);
    }
  };
}, [isPlaying, volume, muted, mediaFiles]);
```

### 4. Text Element Drag and Drop

**Purpose**: Interactive text element positioning with real-time visual feedback.

**Key Logic**:

- Tracks drag state (start position, current position, element bounds)
- Calculates scaled coordinates for canvas positioning
- Constrains dragging within canvas boundaries
- Updates element position on drag completion

**Related React Code**:

```typescript
// Drag state management
const [dragState, setDragState] = useState<TextElementDragState>({
  isDragging: false,
  elementId: null,
  trackId: null,
  startX: 0,
  startY: 0,
  initialElementX: 0,
  initialElementY: 0,
  currentX: 0,
  currentY: 0,
  elementWidth: 0,
  elementHeight: 0,
});

// Drag handling
useEffect(() => {
  const handleMouseMove = (e: MouseEvent) => {
    if (!dragState.isDragging) return;

    const deltaX = e.clientX - dragState.startX;
    const deltaY = e.clientY - dragState.startY;

    // Scale coordinates to canvas space
    const scaleRatio = previewDimensions.width / canvasSize.width;
    const newX = dragState.initialElementX + deltaX / scaleRatio;
    const newY = dragState.initialElementY + deltaY / scaleRatio;

    // Constrain within canvas boundaries
    const halfWidth = dragState.elementWidth / scaleRatio / 2;
    const halfHeight = dragState.elementHeight / scaleRatio / 2;

    const constrainedX = Math.max(
      -canvasSize.width / 2 + halfWidth,
      Math.min(canvasSize.width / 2 - halfWidth, newX)
    );
    const constrainedY = Math.max(
      -canvasSize.height / 2 + halfHeight,
      Math.min(canvasSize.height / 2 - halfHeight, newY)
    );

    setDragState((prev) => ({
      ...prev,
      currentX: constrainedX,
      currentY: constrainedY,
    }));
  };

  const handleMouseUp = () => {
    if (dragState.isDragging && dragState.trackId && dragState.elementId) {
      updateTextElement(dragState.trackId, dragState.elementId, {
        x: dragState.currentX,
        y: dragState.currentY,
      });
    }
    setDragState((prev) => ({ ...prev, isDragging: false }));
  };
}, [dragState, previewDimensions, canvasSize, updateTextElement]);
```

### 5. Active Elements Detection

**Purpose**: Identifies which timeline elements are visible at the current time for rendering.

**Key Logic**:

- Iterates through all tracks (bottom to top for proper layering)
- Checks element visibility and timing
- Filters out hidden elements
- Associates media files with media elements
- Returns ordered list of active elements

**Related React Code**:

```typescript
const getActiveElements = (): ActiveElement[] => {
  const activeElements: ActiveElement[] = [];

  // Iterate tracks from bottom to top so topmost track renders last (on top)
  [...tracks].reverse().forEach((track) => {
    track.elements.forEach((element) => {
      if (element.hidden) return;

      const elementStart = element.startTime;
      const elementEnd =
        element.startTime +
        (element.duration - element.trimStart - element.trimEnd);

      if (currentTime >= elementStart && currentTime < elementEnd) {
        let mediaItem = null;
        if (element.type === "media") {
          mediaItem =
            element.mediaId === "test"
              ? null
              : mediaFiles.find((item) => item.id === element.mediaId) || null;
        }
        activeElements.push({ element, track, mediaItem });
      }
    });
  });

  return activeElements;
};

const activeElements = getActiveElements();
```

### 6. Fullscreen Mode Management

**Purpose**: Handles fullscreen preview with escape key support and body scroll locking.

**Key Logic**:

- Toggles between normal and fullscreen modes
- Locks body scroll in fullscreen
- Handles escape key to exit fullscreen
- Recalculates dimensions for fullscreen layout

**Related React Code**:

```typescript
const [isExpanded, setIsExpanded] = useState(false);

// Fullscreen escape key handling
useEffect(() => {
  const handleEscapeKey = (event: KeyboardEvent) => {
    if (event.key === "Escape" && isExpanded) {
      setIsExpanded(false);
    }
  };

  if (isExpanded) {
    document.addEventListener("keydown", handleEscapeKey);
    document.body.style.overflow = "hidden";
  } else {
    document.body.style.overflow = "";
  }

  return () => {
    document.removeEventListener("keydown", handleEscapeKey);
    document.body.style.overflow = "";
  };
}, [isExpanded]);
```

### 7. Cache Invalidation

**Purpose**: Clears frame cache when timeline state changes that affects rendering.

**Key Logic**:

- Invalidates cache when media files change
- Invalidates cache when background settings change
- Ensures fresh rendering after timeline modifications

**Related React Code**:

```typescript
// Clear the frame cache when background settings change
useEffect(() => {
  invalidateCache();
}, [
  mediaFiles,
  activeProject?.backgroundColor,
  activeProject?.backgroundType,
  invalidateCache,
]);
```

## Component Integration

### Preview Toolbar

The `PreviewToolbar` component provides playback controls and layout guide options:

- **Play/Pause**: Toggle playback state
- **Layout Guide**: Platform-specific overlay guides (TikTok, Instagram, etc.)
- **Fullscreen**: Enter/exit fullscreen mode
- **Time Display**: Current time and total duration

### Fullscreen Mode

The `FullscreenPreview` component renders the preview in fullscreen:

- **Fixed positioning**: Covers entire viewport
- **Centered content**: Preview centered with proper aspect ratio
- **Fullscreen toolbar**: Dedicated controls for fullscreen mode
- **Escape handling**: Keyboard shortcut to exit

## Performance Optimizations

1. **Frame Caching**: Avoids re-rendering identical frames
2. **FPS Throttling**: Limits rendering frequency during playback
3. **Offscreen Canvas**: Non-blocking frame rendering
4. **Pre-rendering**: Background frame preparation for smooth scrubbing
5. **Audio Buffer Caching**: Reuses decoded audio buffers
6. **ResizeObserver**: Efficient dimension change detection

## State Dependencies

The component depends on several stores:

- `useTimelineStore`: Track and element data
- `useMediaStore`: Media file references
- `usePlaybackStore`: Current time and playback state
- `useProjectStore`: Project settings and canvas size
- `useSceneStore`: Current scene information
- `useEditorStore`: Layout guide settings

## Event Handling

- **Mouse Events**: Text element dragging
- **Keyboard Events**: Fullscreen escape key
- **Resize Events**: Dynamic preview sizing
- **Playback Events**: Timeline seeking and state changes
