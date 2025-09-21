# Video Cache System

## Overview

The `VideoCache` class is a high-performance video frame caching system designed for real-time video editing and timeline rendering. It provides efficient frame extraction and caching for video files using the `mediabunny` library, with intelligent frame seeking and iteration strategies.

## Use Cases

The video cache is primarily used in:

1. **Timeline Rendering** (`timeline-renderer.ts`): Renders video frames at specific timestamps during timeline playback
2. **Media Management** (`media-store.ts`): Cleans up video cache when media files are removed
3. **Frame Caching** (`use-frame-cache.ts`): Works alongside the frame cache hook for optimized video rendering

## Core Architecture

### Main Class: `VideoCache`

A singleton class that manages video sinks and frame extraction for multiple video files simultaneously.

### Key Interfaces

```typescript
interface VideoSinkData {
  sink: CanvasSink; // MediaBunny canvas sink for frame extraction
  iterator: AsyncGenerator<WrappedCanvas, void, unknown> | null; // Frame iterator
  currentFrame: WrappedCanvas | null; // Currently cached frame
  lastTime: number; // Timestamp of last processed frame
}
```

## State Management

The `VideoCache` maintains the following state:

### Primary State Maps

1. **`sinks: Map<string, VideoSinkData>`**

   - Maps media IDs to their video sink data
   - Each entry contains the canvas sink, iterator, current frame, and last processed time

2. **`initPromises: Map<string, Promise<void>>`**
   - Tracks ongoing initialization promises for each media file
   - Prevents duplicate initialization attempts for the same media

### Per-Media State (VideoSinkData)

- **`sink`**: MediaBunny CanvasSink instance for frame extraction
- **`iterator`**: Async generator for sequential frame iteration
- **`currentFrame`**: Currently cached frame with timestamp and duration
- **`lastTime`**: Timestamp of the last processed frame

## Flow Logic

### 1. Frame Request Flow (`getFrameAt`)

The frame request process follows these steps:

1. Ensure sink exists for the specified mediaId
2. Check current frame validity - if the current frame is valid for the requested time, return it immediately
3. Check iteration possibility - if the target time is within 2 seconds of the current position, iterate to the target time
4. Direct seek - if iteration fails or the target is too far, seek directly to the target time
5. Return result - return the frame or null if extraction fails

### 2. Sink Initialization Flow (`ensureSink`)

The sink initialization process works as follows:

1. Check existing sink - if a sink already exists for the mediaId, return immediately
2. Check ongoing initialization - if initialization is already in progress, wait for the existing promise
3. Start new initialization - if no sink exists and no initialization is running, begin the process
4. Create input source - create an Input with BlobSource from the file
5. Get video track - retrieve the primary video track from the input
6. Verify codec support - ensure the video codec can be decoded
7. Create canvas sink - instantiate a CanvasSink with pool size 3
8. Store sink data - save the sink data in the sinks map

### 3. Frame Validation Logic

A frame is considered valid if the requested time falls within the frame's display duration:

```typescript
time >= frame.timestamp && time < frame.timestamp + frame.duration;
```

### 4. Iteration Strategy

The system uses three strategies for frame extraction:

1. Current Frame Reuse: If current frame is valid for requested time
2. Sequential Iteration: If target time is within 2 seconds of current position
3. Direct Seek: For larger time jumps or when iteration fails

### 5. Cleanup Flow (`clearVideo`)

The single video cleanup process:

1. Get sink data for the specified mediaId
2. Close active iterator if one is currently running
3. Remove sink data from the sinks map
4. Remove initialization promise if it exists
5. Complete cleanup - if no sink exists, no action is needed

### 6. Bulk Cleanup Flow (`clearAll`)

The bulk cleanup process for all videos:

1. Iterate through all sink entries in the sinks map
2. For each sink:
   - Close the active iterator if running
   - Remove the sink data from the sinks map
3. Clear all initialization promises from the initPromises map
4. Complete bulk cleanup - all video data is now cleaned up

## Key Methods

### Public Methods

- `getFrameAt(mediaId, file, time)`: Main entry point for frame extraction
- `clearVideo(mediaId)`: Clean up specific video sink and iterator
- `clearAll()`: Clean up all video sinks
- `getStats()`: Get cache statistics (total sinks, active sinks, cached frames)

### Private Methods

- `ensureSink(mediaId, file)`: Ensure video sink exists for media
- `initializeSink(mediaId, file)`: Initialize new video sink
- `isFrameValid(frame, time)`: Check if frame is valid for timestamp
- `iterateToTime(sinkData, targetTime)`: Iterate to target time from current position
- `seekToTime(sinkData, time)`: Direct seek to target time

## Frame Structure

The `getFrameAt()` method returns a `WrappedCanvas` object. See [timeline-renderer.md](./timeline-renderer.md#frame-structure-wrappedcanvas) for detailed frame structure documentation.

## Performance Optimizations

1. Lazy Initialization: Sinks are only created when first needed
2. Frame Reuse: Valid frames are reused without re-extraction
3. Smart Iteration: Sequential iteration for nearby frames (within 2 seconds)
4. Pool Management: CanvasSink uses pool size of 3 for efficient memory usage
5. Error Recovery: Failed iterations trigger direct seek as fallback

## Error Handling

- Codec Support: Validates video codec before creating sink
- Iterator Failures: Gracefully handles iterator errors with fallback to direct seek
- Seek Failures: Logs warnings for seek failures and returns null
- Initialization Errors: Properly cleans up failed initialization attempts

## Memory Management

- Automatic Cleanup: `clearVideo()` properly disposes iterators and removes sink data
- Bulk Cleanup: `clearAll()` removes all video sinks
- Iterator Disposal: Iterators are properly closed when seeking or clearing

## Integration Points

### Timeline Renderer

```typescript
const frame = await videoCache.getFrameAt(
  mediaItem.id,
  mediaItem.file,
  localTime
);
```

### Media Store

```typescript
// Cleanup when removing media
videoCache.clearVideo(id);

// Cleanup all media
videoCache.clearAll();
```

## Configuration

The video cache uses the following MediaBunny configuration:

- Pool Size: 3 (for efficient memory usage)
- Fit Mode: "contain" (maintains aspect ratio)
- Formats: ALL_FORMATS (supports all video formats)

## Statistics

The `getStats()` method provides:

- `totalSinks`: Number of initialized video sinks
- `activeSinks`: Number of sinks with active iterators
- `cachedFrames`: Number of sinks with cached frames

This information is useful for monitoring cache performance and memory usage.
