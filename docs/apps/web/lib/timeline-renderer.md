# Timeline Renderer

A comprehensive canvas-based rendering system for video editor timeline frames. This module handles the rendering of all timeline elements including media, text, and background effects onto a canvas context.

## Overview

The timeline renderer is responsible for:

- **Canvas-based frame rendering** for video editor preview
- **Multi-layer composition** of timeline elements
- **Background effect rendering** including blur and color backgrounds
- **Element positioning and scaling** based on project canvas size
- **Media element rendering** for video and image content
- **Text element rendering** with styling and positioning

## Core Architecture

### RenderContext Interface

The renderer operates on a `RenderContext` object that contains all necessary data for rendering:

```typescript
interface RenderContext {
  ctx: CanvasRenderingContext2D; // Canvas context for drawing
  time: number; // Current timeline time
  canvasWidth: number; // Canvas display width
  canvasHeight: number; // Canvas display height
  tracks: TimelineTrack[]; // Timeline tracks with elements
  mediaFiles: MediaFile[]; // Available media files
  backgroundColor?: string; // Background color or gradient
  backgroundType?: "color" | "blur"; // Background rendering type
  blurIntensity?: BlurIntensity; // Blur effect intensity
  projectCanvasSize?: { width: number; height: number }; // Project dimensions
}
```

### Data Flow

1. **Input Processing**: Timeline tracks and media files are processed
2. **Active Element Detection**: Elements active at current time are identified
3. **Background Rendering**: Background color, gradient, or blur is applied
4. **Element Rendering**: Active elements are rendered in reverse track order
5. **Canvas Output**: Final composed frame is drawn to canvas context

## Main Rendering Function

### `renderTimelineFrame()`

The primary rendering function that orchestrates the entire frame composition process.

#### Process Flow

1. **Canvas Initialization**

   ```typescript
   ctx.clearRect(0, 0, canvasWidth, canvasHeight);
   ```

2. **Background Rendering**

   - Solid color backgrounds
   - CSS gradient backgrounds
   - Blur background effects

3. **Scaling Calculation**

   ```typescript
   const scaleX = projectCanvasSize ? canvasWidth / projectCanvasSize.width : 1;
   const scaleY = projectCanvasSize
     ? canvasHeight / projectCanvasSize.height
     : 1;
   ```

4. **Active Element Detection**

   - Iterates through tracks in reverse order (bottom to top)
   - Identifies elements active at current time
   - Skips hidden elements
   - Calculates element visibility considering trim settings

5. **Element Rendering**
   - Renders elements in reverse track order (topmost last)
   - Handles different element types (media, text)
   - Applies proper scaling and positioning

## Background Rendering

### Color Backgrounds

```typescript
if (
  backgroundColor &&
  backgroundColor !== "transparent" &&
  !backgroundColor.includes("gradient")
) {
  ctx.fillStyle = backgroundColor;
  ctx.fillRect(0, 0, canvasWidth, canvasHeight);
}
```

### Gradient Backgrounds

```typescript
if (backgroundColor && backgroundColor.includes("gradient")) {
  drawCssBackground(ctx, canvasWidth, canvasHeight, backgroundColor);
}
```

### Blur Background Effects

When `backgroundType === "blur"`:

1. Finds suitable media element (video or image)
2. Renders media as blurred background layer
3. Uses CSS filter blur effect
4. Scales media to cover entire canvas

```typescript
if (backgroundType === "blur") {
  const blurPx = Math.max(0, blurIntensity ?? 8);
  const bgCandidate = active.find(({ element, mediaItem }) => {
    return (
      element.type === "media" &&
      mediaItem !== null &&
      (mediaItem.type === "video" || mediaItem.type === "image")
    );
  });

  if (bgCandidate && bgCandidate.mediaItem) {
    // Render blurred background
    ctx.save();
    ctx.filter = `blur(${blurPx}px)`;
    ctx.drawImage(/* media content */);
    ctx.restore();
  }
}
```

## Element Rendering

### Media Element Rendering

#### Video Elements

```typescript
if (mediaItem.type === "video") {
  const localTime = time - element.startTime + element.trimStart;
  const frame = await videoCache.getFrameAt(
    mediaItem.id,
    mediaItem.file,
    localTime
  );

  if (frame) {
    // Calculate scaling to fit canvas (contain behavior)
    const containScale = Math.min(canvasWidth / mediaW, canvasHeight / mediaH);
    const drawW = mediaW * containScale;
    const drawH = mediaH * containScale;
    const drawX = (canvasWidth - drawW) / 2;
    const drawY = (canvasHeight - drawH) / 2;

    ctx.drawImage(frame.canvas, drawX, drawY, drawW, drawH);
  }
}
```

#### Image Elements

```typescript
if (mediaItem.type === "image") {
  const img = await getImageElement(mediaItem);
  // Similar scaling and positioning logic as video
  ctx.drawImage(img, drawX, drawY, drawW, drawH);
}
```

**Key Features:**

- **Contain scaling**: Media fits within canvas while maintaining aspect ratio
- **Center positioning**: Media is centered on canvas
- **Video frame caching**: Uses `videoCache` for efficient video frame extraction
- **Image caching**: Caches loaded images to avoid repeated loading

### Text Element Rendering

```typescript
if (element.type === "text") {
  const text = element;
  const posX = canvasWidth / 2 + text.x * scaleX;
  const posY = canvasHeight / 2 + text.y * scaleY;

  ctx.save();
  ctx.translate(posX, posY);
  ctx.rotate((text.rotation * Math.PI) / 180);
  ctx.globalAlpha = Math.max(0, Math.min(1, text.opacity));

  // Font setup
  const px = text.fontSize * scaleX;
  const weight = text.fontWeight === "bold" ? "bold " : "";
  const style = text.fontStyle === "italic" ? "italic " : "";
  ctx.font = `${style}${weight}${px}px ${text.fontFamily}`;
  ctx.fillStyle = text.color;
  ctx.textAlign = text.textAlign as CanvasTextAlign;
  ctx.textBaseline = "middle";

  // Background rendering
  if (text.backgroundColor) {
    // Calculate background dimensions
    const metrics = ctx.measureText(text.content);
    const textW = metrics.width;
    const textH = ascent + descent;
    const padX = 8 * scaleX;
    const padY = 4 * scaleX;

    ctx.fillStyle = text.backgroundColor;
    ctx.fillRect(
      bgLeft - padX,
      -textH / 2 - padY,
      textW + padX * 2,
      textH + padY * 2
    );
  }

  ctx.fillText(text.content, 0, 0);
  ctx.restore();
}
```

**Text Rendering Features:**

- **Positioning**: Text positioned relative to canvas center with scaling
- **Rotation**: Support for text rotation in degrees
- **Opacity**: Alpha blending support
- **Font styling**: Bold, italic, font family, and size support
- **Text alignment**: Left, center, right alignment
- **Background**: Optional background color with padding
- **Scaling**: Font size scales with canvas resolution

## Caching and Performance

### Image Element Caching

```typescript
const imageElementCache = new Map<string, HTMLImageElement>();

async function getImageElement(
  mediaItem: MediaFile
): Promise<HTMLImageElement> {
  const cacheKey = mediaItem.id;
  const cached = imageElementCache.get(cacheKey);
  if (cached) return cached;

  const img = new Image();
  await new Promise<void>((resolve, reject) => {
    img.onload = () => resolve();
    img.onerror = () => reject(new Error("Image load failed"));
    img.src = mediaItem.url || URL.createObjectURL(mediaItem.file);
  });

  imageElementCache.set(cacheKey, img);
  return img;
}
```

### Video Frame Caching

- Uses `videoCache.getFrameAt()` for efficient video frame extraction
- Caches video frames to avoid repeated decoding
- Handles video timing and trimming

## Scaling and Positioning

### Canvas Scaling

The renderer supports two scaling modes:

1. **Display Scaling**: Canvas dimensions for display
2. **Project Scaling**: Original project dimensions

```typescript
const scaleX = projectCanvasSize ? canvasWidth / projectCanvasSize.width : 1;
const scaleY = projectCanvasSize ? canvasHeight / projectCanvasSize.height : 1;
```

### Element Positioning

- **Media elements**: Centered with contain scaling
- **Text elements**: Positioned relative to canvas center with project scaling
- **Background elements**: Cover entire canvas

## Error Handling

### Graceful Degradation

```typescript
try {
  // Video frame extraction
  const frame = await videoCache.getFrameAt(/* params */);
  // ... rendering logic
} catch (error) {
  console.warn(`Failed to render video frame for ${mediaItem.name}:`, error);
}
```

### Background Blur Fallback

```typescript
try {
  // Blur background rendering
} catch {
  // Ignore background blur failures; foreground will still render
}
```

## Integration with Frame Cache

The timeline renderer works closely with the frame cache system:

1. **Rendering Function**: Used as the render function for `preRenderNearbyFrames()`
2. **Canvas Context**: Renders to offscreen canvas for caching
3. **ImageData Output**: Returns `ImageData` for cache storage
4. **State Dependencies**: Renders based on timeline state for cache validation

## Usage Example

```typescript
import { renderTimelineFrame } from "@/lib/timeline-renderer";

// Render a frame at specific time
await renderTimelineFrame({
  ctx: canvasContext,
  time: 5.5, // 5.5 seconds
  canvasWidth: 1920,
  canvasHeight: 1080,
  tracks: timelineTracks,
  mediaFiles: projectMediaFiles,
  backgroundColor: "#000000",
  backgroundType: "color",
  projectCanvasSize: { width: 1920, height: 1080 },
});
```

## Performance Considerations

### Rendering Order

- **Bottom-to-top**: Tracks rendered in reverse order
- **Element layering**: Later elements appear on top
- **Background first**: Background rendered before elements

### Memory Management

- **Image caching**: Prevents repeated image loading
- **Video frame caching**: Efficient video frame extraction
- **Canvas reuse**: Reuses canvas contexts when possible

### Scaling Optimization

- **Project scaling**: Maintains aspect ratios
- **Display scaling**: Adapts to different canvas sizes
- **Font scaling**: Text scales proportionally

## Future Enhancements

### Potential Improvements

- **WebGL rendering**: Hardware-accelerated rendering
- **Layer compositing**: More sophisticated blending modes
- **Effect rendering**: Filters, transitions, and effects
- **Animation support**: Keyframe-based animations
- **Performance monitoring**: Render time tracking and optimization

### Extensibility

- **Custom element types**: Plugin system for new element types
- **Effect system**: Modular effect rendering
- **Export formats**: Multiple output format support
- **Quality settings**: Configurable rendering quality levels
