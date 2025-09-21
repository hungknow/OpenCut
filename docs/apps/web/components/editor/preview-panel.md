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

#### Available Space Calculation

The component calculates available space by subtracting UI elements and spacing from container dimensions.

**Example Calculations**:

_Fullscreen Mode_ (1920×1080 window):

- Window: 1920×1080
- Reserved: 80px toolbar + 24px margins = 104px
- Available: 1920×976 (width: 1920-24, height: 1080-80-24)

_Normal Mode_ (800×600 container):

- Container: 800×600
- Padding: 16px all sides = 32px total
- Toolbar: 40px + 16px gap = 56px
- Available: 768×512 (width: 800-32, height: 600-32-56)

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
      // Fullscreen mode: use entire browser window minus reserved space
      const controlsHeight = 80; // Height reserved for fullscreen toolbar
      const marginSpace = 24; // Margin around preview (12px on each side)
      availableWidth = window.innerWidth - marginSpace;
      availableHeight = window.innerHeight - controlsHeight - marginSpace;
    } else {
      // Normal mode: use container dimensions minus all UI elements
      const container = containerRef.current.getBoundingClientRect();
      const computedStyle = getComputedStyle(containerRef.current);

      // Extract all CSS padding values from the container
      const paddingTop = parseFloat(computedStyle.paddingTop);
      const paddingBottom = parseFloat(computedStyle.paddingBottom);
      const paddingLeft = parseFloat(computedStyle.paddingLeft);
      const paddingRight = parseFloat(computedStyle.paddingRight);

      // Extract gap between elements (defaults to 16px if not set)
      const gap = parseFloat(computedStyle.gap) || 16;

      // Find and measure the toolbar height dynamically
      const toolbar = containerRef.current.querySelector("[data-toolbar]");
      const toolbarHeight = toolbar
        ? toolbar.getBoundingClientRect().height
        : 0;

      // Calculate available space by subtracting all reserved space
      availableWidth = container.width - paddingLeft - paddingRight;
      availableHeight =
        container.height -
        paddingTop -
        paddingBottom -
        toolbarHeight -
        (toolbarHeight > 0 ? gap : 0); // Only subtract gap if toolbar exists
    }

    // Maintain canvas aspect ratio while fitting available space
    const targetRatio = canvasSize.width / canvasSize.height; // e.g., 16:9 = 1.78
    const containerRatio = availableWidth / availableHeight; // e.g., 1920:976 = 1.97

    if (containerRatio > targetRatio) {
      // Container is wider than needed - fit to height and calculate width
      // Example: 1920×976 available, 16:9 canvas → fit to 976px height
      height = availableHeight * (isExpanded ? 0.95 : 1); // 976 × 0.95 = 927px
      width = height * targetRatio; // 927 × 1.78 = 1650px
    } else {
      // Container is taller than needed - fit to width and calculate height
      // Example: 768×512 available, 16:9 canvas → fit to 768px width
      width = availableWidth * (isExpanded ? 0.95 : 1); // 768 × 1 = 768px
      height = width / targetRatio; // 768 ÷ 1.78 = 431px
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

**Performance Examples**:

_FPS Throttling_ (30fps project):

- Min frame interval: 1/30 = 33.33ms
- If last frame was 16ms ago → skip rendering
- If last frame was 40ms ago → render new frame

_Canvas Resolution_ (1920×1080 display):

- Display size: 1920×1080
- Canvas internal: 1920×1080 (1:1 scaling)
- Memory usage: 1920×1080×4 = 8.3MB per frame

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

**Audio Timing Examples**:

_Element Timing_ (10s audio element):

- Element starts at: 5.0s
- Current time: 7.0s
- Trim start: 1.0s
- Local time: 7.0 - 5.0 + 1.0 = 3.0s (3 seconds into trimmed audio)

_Volume Control_:

- Volume: 0.8 (80%)
- Muted: false
- Final gain: 0.8 × 1.0 = 0.8
- Muted: true → Final gain: 0.8 × 0.0 = 0.0

**Related React Code**:

```typescript
// Audio state management
const audioContextRef = useRef<AudioContext | null>(null);
const audioGainRef = useRef<GainNode | null>(null);
const audioBuffersRef = useRef<Map<string, AudioBuffer>>(new Map());
const playingSourcesRef = useRef<Set<AudioBufferSourceNode>>(new Set());

// Audio scheduling effect
useEffect(() => {
  // Helper function to stop all playing sources
  const stopAll = () => {
    for (const src of playingSourcesRef.current) {
      try {
        src.stop(); // Stop the audio source
      } catch {} // Ignore errors (source might already be stopped)
    }
    playingSourcesRef.current.clear(); // Remove from tracking set
  };

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

    // Collect audible elements logic:
    // This finds all audio elements that should be playing at the current timeline position

    // 1. Get current timeline state and media files
    const tracksSnapshot = useTimelineStore.getState().tracks;
    const mediaList = mediaFiles;
    const idToMedia = new Map(mediaList.map((m) => [m.id, m] as const));
    const playbackNow = usePlaybackStore.getState().currentTime;

    // 2. Find all audio elements that should be playing at current time
    const uniqueIds = new Set<string>();
    for (const track of tracksSnapshot) {
      for (const element of track.elements) {
        // Only process media elements (skip text elements)
        if (element.type !== "media") continue;

        // Find the associated media file
        const media = idToMedia.get(element.mediaId);
        if (!media || media.type !== "audio") continue;

        // Calculate visible duration (after trimming)
        // Example: 10s element with 2s trimStart + 1s trimEnd = 7s visible
        const visibleDuration =
          element.duration - element.trimStart - element.trimEnd;
        if (visibleDuration <= 0) continue;

        // Check if element is active at current time
        // Example: element starts at 5s, current time is 7s, trimStart is 1s
        // localTime = 7 - 5 + 1 = 3s (3 seconds into the trimmed audio)
        const localTime = playbackNow - element.startTime + element.trimStart;
        if (localTime < 0 || localTime >= visibleDuration) continue;

        // Add to audible elements if not muted
        // This element should be playing audio right now
        audible.push({
          id: media.id,
          elementStart: element.startTime,
          trimStart: element.trimStart,
          trimEnd: element.trimEnd,
          duration: element.duration,
          muted: !!element.muted,
          trackMuted: !!track.muted,
        });
        uniqueIds.add(media.id);
      }
    }

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

      // Clean up source when it finishes playing naturally
      src.onended = () => {
        playingSourcesRef.current.delete(src);
      };

      src.start(startAt, localTime, playDuration);
      playingSourcesRef.current.add(src); // Track the source
    }
  };

  // Handle timeline seeking
  const onSeek = () => {
    if (!isPlaying) return;
    stopAll(); // Stop all current sources
    void scheduleNow(); // Schedule new sources for new time
  };

  // Cleanup on effect dependencies change
  stopAll(); // Stop all current sources
  if (isPlaying) {
    void scheduleNow(); // Schedule new sources if playing
  }

  // Cleanup on unmount
  return () => {
    stopAll(); // Stop all sources when component unmounts
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

**Coordinate Scaling Examples**:

_Mouse to Canvas Conversion_:

- Preview size: 800×600
- Canvas size: 1920×1080
- Scale ratio: 800/1920 = 0.417
- Mouse delta: +100px
- Canvas delta: 100/0.417 = 240px

_Boundary Constraints_ (1920×1080 canvas):

- Element size: 200×100
- Max X: 1920/2 - 200/2 = 860px
- Min X: -1920/2 + 200/2 = -860px
- Max Y: 1080/2 - 100/2 = 490px
- Min Y: -1080/2 + 100/2 = -490px

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

**Purpose**: Handles fullscreen preview with escape key support, body scroll locking, and dedicated fullscreen rendering.

**Key Logic**:

- Toggles between normal and fullscreen modes
- Locks body scroll in fullscreen
- Handles escape key to exit fullscreen
- Recalculates dimensions for fullscreen layout
- Renders dedicated fullscreen component with fixed positioning

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

// Fullscreen component rendering
{
  isExpanded && (
    <FullscreenPreview
      previewDimensions={previewDimensions}
      activeProject={activeProject}
      renderBlurBackground={renderBlurBackground}
      activeElements={activeElements}
      renderElement={renderElement}
      blurBackgroundElements={blurBackgroundElements}
      hasAnyElements={hasAnyElements}
      toggleExpanded={toggleExpanded}
      currentTime={currentTime}
      setCurrentTime={setCurrentTime}
      toggle={toggle}
      getTotalDuration={getTotalDuration}
    />
  );
}
```

### 6.1. Fullscreen Preview Component

**Purpose**: Renders the preview in fullscreen mode with fixed positioning and dedicated controls.

**Key Logic**:

- Fixed positioning covers entire viewport (`fixed inset-0 z-9999`)
- Centered preview with proper aspect ratio
- Dedicated fullscreen toolbar with timeline controls
- Same canvas rendering as normal mode but with fullscreen dimensions

**Related React Code**:

```typescript
function FullscreenPreview({
  previewDimensions,
  activeProject,
  renderBlurBackground,
  activeElements,
  renderElement,
  blurBackgroundElements,
  hasAnyElements,
  toggleExpanded,
  currentTime,
  setCurrentTime,
  toggle,
  getTotalDuration,
}) {
  return (
    <div className="fixed inset-0 z-9999 flex flex-col">
      <div className="flex-1 flex items-center justify-center bg-background">
        <div
          className="relative overflow-hidden border border-border m-3"
          style={{
            width: previewDimensions.width,
            height: previewDimensions.height,
            background:
              activeProject?.backgroundType === "blur"
                ? "#1a1a1a"
                : activeProject?.backgroundColor || "#1a1a1a",
          }}
        >
          {renderBlurBackground()}
          {activeElements.length === 0 ? (
            <div className="absolute inset-0 flex items-center justify-center text-white/60">
              No elements at current time
            </div>
          ) : (
            activeElements.map((elementData, index) =>
              renderElement(elementData, index)
            )
          )}
          <LayoutGuideOverlay />
        </div>
      </div>
      <div className="p-4 bg-background">
        <FullscreenToolbar
          hasAnyElements={hasAnyElements}
          onToggleExpanded={toggleExpanded}
          currentTime={currentTime}
          setCurrentTime={setCurrentTime}
          toggle={toggle}
          getTotalDuration={getTotalDuration}
        />
      </div>
    </div>
  );
}
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

The PreviewPanel component handles various user interactions and system events to provide a responsive editing experience.

### 1. Mouse Events

#### Text Element Dragging

**Purpose**: Enables interactive positioning of text elements on the canvas.

**Event Flow**:

1. `onMouseDown` on text element → Start drag operation
2. `onMouseMove` on document → Update drag position with constraints
3. `onMouseUp` on document → Complete drag and update element position

**Handler Implementation**:

```typescript
// Mouse down handler - initiates drag
const handleTextMouseDown = (
  e: React.MouseEvent<HTMLDivElement>,
  element: any,
  trackId: string
) => {
  e.preventDefault();
  e.stopPropagation();

  const rect = e.currentTarget.getBoundingClientRect();

  setDragState({
    isDragging: true,
    elementId: element.id,
    trackId,
    startX: e.clientX,
    startY: e.clientY,
    initialElementX: element.x,
    initialElementY: element.y,
    currentX: element.x,
    currentY: element.y,
    elementWidth: rect.width,
    elementHeight: rect.height,
  });
};

// Mouse move handler - updates drag position
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

  if (dragState.isDragging) {
    document.addEventListener("mousemove", handleMouseMove);
    document.addEventListener("mouseup", handleMouseUp);
    document.body.style.cursor = "grabbing";
    document.body.style.userSelect = "none";
  }

  return () => {
    document.removeEventListener("mousemove", handleMouseMove);
    document.removeEventListener("mouseup", handleMouseUp);
    document.body.style.cursor = "";
    document.body.style.userSelect = "";
  };
}, [dragState, previewDimensions, canvasSize, updateTextElement]);
```

**What Happens**:

- **Start**: Captures initial mouse position and element bounds
- **During**: Calculates scaled coordinates and constrains to canvas boundaries
- **End**: Updates element position in timeline store and cleans up drag state

#### Timeline Scrubbing (Fullscreen Mode)

**Purpose**: Allows direct timeline seeking by clicking/dragging on the progress bar.

**Event Flow**:

1. `onClick` on timeline → Jump to clicked position
2. `onMouseDown` on timeline → Start drag scrubbing
3. `onMouseMove` on document → Update timeline position
4. `onMouseUp` on document → Complete scrubbing

**Handler Implementation**:

```typescript
const handleTimelineClick = (e: React.MouseEvent<HTMLDivElement>) => {
  if (!hasAnyElements) return;
  const rect = e.currentTarget.getBoundingClientRect();
  const clickX = e.clientX - rect.left;
  const percentage = Math.max(0, Math.min(1, clickX / rect.width));
  const newTime = percentage * totalDuration;
  setCurrentTime(Math.max(0, Math.min(newTime, totalDuration)));
};

const handleTimelineDrag = (e: React.MouseEvent<HTMLDivElement>) => {
  if (!hasAnyElements) return;
  e.preventDefault();
  e.stopPropagation();
  const rect = e.currentTarget.getBoundingClientRect();
  setIsDragging(true);

  const handleMouseMove = (moveEvent: MouseEvent) => {
    moveEvent.preventDefault();
    const dragX = moveEvent.clientX - rect.left;
    const percentage = Math.max(0, Math.min(1, dragX / rect.width));
    const newTime = percentage * totalDuration;
    setCurrentTime(Math.max(0, Math.min(newTime, totalDuration)));
  };

  const handleMouseUp = () => {
    setIsDragging(false);
    document.removeEventListener("mousemove", handleMouseMove);
    document.removeEventListener("mouseup", handleMouseUp);
    document.body.style.userSelect = "";
  };

  document.body.style.userSelect = "none";
  document.addEventListener("mousemove", handleMouseMove);
  document.addEventListener("mouseup", handleMouseUp);
  handleMouseMove(e.nativeEvent);
};
```

### 2. Keyboard Events

#### Fullscreen Escape Key

**Purpose**: Provides keyboard shortcut to exit fullscreen mode.

**Event Flow**:

1. `keydown` on document → Check for Escape key
2. If fullscreen active → Exit fullscreen mode

**Handler Implementation**:

```typescript
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

**What Happens**:

- **Escape Pressed**: Exits fullscreen mode and restores normal layout
- **Body Scroll**: Locked during fullscreen to prevent background scrolling

### 3. Resize Events

#### Dynamic Preview Sizing

**Purpose**: Maintains proper aspect ratio and responsive sizing when container or window dimensions change.

**Event Flow**:

1. Container resize → Recalculate available space
2. Window resize (fullscreen) → Recalculate fullscreen dimensions
3. Update preview dimensions → Trigger re-render

**Handler Implementation**:

```typescript
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
      const computedStyle = getComputedStyle(containerRef.current);
      const paddingTop = parseFloat(computedStyle.paddingTop);
      const paddingBottom = parseFloat(computedStyle.paddingBottom);
      const paddingLeft = parseFloat(computedStyle.paddingLeft);
      const paddingRight = parseFloat(computedStyle.paddingRight);
      const gap = parseFloat(computedStyle.gap) || 16;
      const toolbar = containerRef.current.querySelector("[data-toolbar]");
      const toolbarHeight = toolbar
        ? toolbar.getBoundingClientRect().height
        : 0;

      availableWidth = container.width - paddingLeft - paddingRight;
      availableHeight =
        container.height -
        paddingTop -
        paddingBottom -
        toolbarHeight -
        (toolbarHeight > 0 ? gap : 0);
    }

    // Maintain aspect ratio
    const targetRatio = canvasSize.width / canvasSize.height;
    const containerRatio = availableWidth / availableHeight;
    let width, height;

    if (containerRatio > targetRatio) {
      height = availableHeight * (isExpanded ? 0.95 : 1);
      width = height * targetRatio;
    } else {
      width = availableWidth * (isExpanded ? 0.95 : 1);
      height = width / targetRatio;
    }

    setPreviewDimensions({ width, height });
  };

  updatePreviewSize();
  const resizeObserver = new ResizeObserver(updatePreviewSize);
  if (containerRef.current) {
    resizeObserver.observe(containerRef.current);
  }
  if (isExpanded) {
    window.addEventListener("resize", updatePreviewSize);
  }

  return () => {
    resizeObserver.disconnect();
    if (isExpanded) {
      window.removeEventListener("resize", updatePreviewSize);
    }
  };
}, [canvasSize.width, canvasSize.height, isExpanded]);
```

**What Happens**:

- **Container Resize**: Uses ResizeObserver for efficient container dimension tracking
- **Window Resize**: Only active in fullscreen mode to avoid unnecessary calculations
- **Aspect Ratio**: Maintains canvas aspect ratio while fitting available space
- **Canvas Update**: Triggers re-render with new dimensions

### 4. Playback Events

#### Timeline Seeking

**Purpose**: Handles timeline position changes and triggers immediate frame rendering.

**Event Flow**:

1. `playback-seek` custom event → Reset frame timing
2. Audio seeking → Stop current audio sources
3. Frame rendering → Display new frame immediately

**Handler Implementation**:

```typescript
// Frame rendering seek handling
useEffect(() => {
  const onSeek = () => {
    lastFrameTimeRef.current = -Infinity;
    renderSeqRef.current++;
  };
  window.addEventListener("playback-seek", onSeek as EventListener);
  lastFrameTimeRef.current = -Infinity;
  return () => {
    window.removeEventListener("playback-seek", onSeek as EventListener);
  };
}, []);

// Audio seeking handling
useEffect(() => {
  const onSeek = () => {
    if (!isPlaying) return;
    for (const src of playingSourcesRef.current) {
      try {
        src.stop();
      } catch {}
    }
    playingSourcesRef.current.clear();
    void scheduleNow();
  };

  window.addEventListener("playback-seek", onSeek as EventListener);
  return () => {
    window.removeEventListener("playback-seek", onSeek as EventListener);
  };
}, [isPlaying, volume, muted, mediaFiles]);
```

**What Happens**:

- **Frame Seek**: Resets frame timing to force immediate re-render
- **Audio Seek**: Stops current audio sources and schedules new ones
- **Custom Event**: Uses `playback-seek` event for coordinated seeking across components

#### Playback State Changes

**Purpose**: Manages audio playback synchronization with timeline state.

**Event Flow**:

1. Play state change → Start/stop audio scheduling
2. Volume/mute change → Update audio gain
3. Media files change → Invalidate audio buffers

**Handler Implementation**:

```typescript
useEffect(() => {
  // Stop all current sources on any change
  for (const src of playingSourcesRef.current) {
    try {
      src.stop();
    } catch {}
  }
  playingSourcesRef.current.clear();

  // Apply volume/mute changes immediately
  void ensureAudioGraph();

  // Start/stop on play state changes
  if (isPlaying) {
    void scheduleNow();
  }
}, [isPlaying, volume, muted, mediaFiles]);
```

**What Happens**:

- **Play Start**: Schedules audio for current timeline position
- **Play Stop**: Stops all audio sources
- **Volume Change**: Updates audio gain node immediately
- **Media Change**: Clears audio buffer cache and re-decodes

### 5. Event Cleanup

**Purpose**: Prevents memory leaks and ensures proper event listener cleanup.

**Cleanup Strategy**:

```typescript
// Each useEffect returns cleanup function
useEffect(() => {
  // Event setup
  document.addEventListener("mousemove", handleMouseMove);
  document.addEventListener("mouseup", handleMouseUp);

  // Cleanup function
  return () => {
    document.removeEventListener("mousemove", handleMouseMove);
    document.removeEventListener("mouseup", handleMouseUp);
    document.body.style.cursor = "";
    document.body.style.userSelect = "";
  };
}, [dependencies]);
```

**What Happens**:

- **Component Unmount**: All event listeners are removed
- **Dependency Change**: Old listeners removed before adding new ones
- **State Cleanup**: Cursor and selection styles restored
- **Memory Management**: Audio sources and buffers properly cleaned up
