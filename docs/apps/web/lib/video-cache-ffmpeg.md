# Video Cache System (FFmpeg Implementation)

## Overview

The `VideoCache` class is a high-performance video frame caching system designed for real-time video editing and timeline rendering. This FFmpeg-based implementation replaces the MediaBunny library with direct FFmpeg.wasm integration, providing efficient frame extraction and caching for video files with intelligent frame seeking and iteration strategies.

## Use Cases

The video cache is primarily used in:

1. **Timeline Rendering** (`timeline-renderer.ts`): Renders video frames at specific timestamps during timeline playback
2. **Media Management** (`media-store.ts`): Cleans up video cache when media files are removed
3. **Frame Caching** (`use-frame-cache.ts`): Works alongside the frame cache hook for optimized video rendering

## Core Architecture

### Main Class: `VideoCache`

A singleton class that manages FFmpeg instances and frame extraction for multiple video files simultaneously.

### Key Interfaces

```typescript
interface VideoSinkData {
  ffmpeg: FFmpeg; // FFmpeg.wasm instance
  videoInfo: VideoInfo; // Video metadata (duration, fps, dimensions)
  currentFrame: VideoFrame | null; // Currently cached frame
  lastTime: number; // Timestamp of last processed frame
  frameCache: Map<number, VideoFrame>; // Frame cache for nearby timestamps
  isInitialized: boolean; // Initialization state
}

interface VideoInfo {
  duration: number; // Video duration in seconds
  fps: number; // Frames per second
  width: number; // Video width in pixels
  height: number; // Video height in pixels
  codec: string; // Video codec name
}

interface VideoFrame {
  canvas: HTMLCanvasElement; // Canvas element containing the frame
  timestamp: number; // Frame timestamp in seconds
  duration: number; // Frame duration in seconds
  width: number; // Frame width
  height: number; // Frame height
}
```

## State Management

The `VideoCache` maintains the following state:

### Primary State Maps

1. **`sinks: Map<string, VideoSinkData>`**

   - Maps media IDs to their FFmpeg sink data
   - Each entry contains the FFmpeg instance, video info, current frame, and frame cache

2. **`initPromises: Map<string, Promise<void>>`**
   - Tracks ongoing initialization promises for each media file
   - Prevents duplicate initialization attempts for the same media

### Per-Media State (VideoSinkData)

- **`ffmpeg`**: FFmpeg.wasm instance for video processing
- **`videoInfo`**: Video metadata including duration, fps, dimensions, and codec
- **`currentFrame`**: Currently cached frame with timestamp and duration
- **`lastTime`**: Timestamp of the last processed frame
- **`frameCache`**: Map of nearby frames for efficient retrieval
- **`isInitialized`**: Boolean flag indicating if the FFmpeg instance is ready

## Flow Logic

### 1. Frame Request Flow (`getFrameAt`)

The frame request process follows these steps:

1. **Ensure Sink**: Verify FFmpeg instance exists for the specified mediaId
2. **Check Frame Cache**: Look for the requested frame in the local frame cache
3. **Check Current Frame**: If the current frame is valid for the requested time, return it immediately
4. **Check Sequential Access**: If the target time is within 2 seconds of the current position, try sequential frame extraction
5. **Direct Seek**: If sequential access fails or the target is too far, seek directly to the target time
6. **Cache Frame**: Store the extracted frame in the local cache
7. **Return Result**: Return the frame or null if extraction fails

### 2. Sink Initialization Flow (`ensureSink`)

The sink initialization process works as follows:

1. **Check Existing Sink**: If a sink already exists for the mediaId, return immediately
2. **Check Ongoing Initialization**: If initialization is already in progress, wait for the existing promise
3. **Start New Initialization**: If no sink exists and no initialization is running, begin the process
4. **Load FFmpeg**: Load the FFmpeg.wasm core and initialize the FFmpeg instance
5. **Write File**: Write the video file to the FFmpeg virtual filesystem
6. **Extract Metadata**: Use FFmpeg to extract video information (duration, fps, dimensions, codec)
7. **Store Sink Data**: Save the sink data in the sinks map with initialized state

### 3. Frame Validation Logic

A frame is considered valid if the requested time falls within the frame's display duration:

```typescript
time >= frame.timestamp && time < frame.timestamp + frame.duration;
```

### 4. Frame Extraction Strategies

The system uses four strategies for frame extraction:

1. **Frame Cache Lookup**: Check local frame cache for nearby frames
2. **Current Frame Reuse**: If current frame is valid for requested time
3. **Sequential Extraction**: If target time is within 2 seconds of current position
4. **Direct Seek**: For larger time jumps or when other methods fail

### 5. FFmpeg Frame Extraction Process

The frame extraction process using FFmpeg:

1. **Calculate Frame Number**: Convert timestamp to frame number using video FPS
2. **Generate Command**: Create FFmpeg command to extract specific frame
3. **Execute Command**: Run FFmpeg command to extract frame as PNG
4. **Read Output**: Read the extracted frame from FFmpeg output
5. **Create Canvas**: Convert PNG data to HTMLCanvasElement
6. **Create Frame Object**: Wrap canvas in VideoFrame object with metadata

### 6. Frame Caching Strategy

The system implements intelligent frame caching:

1. **Local Cache**: Each video sink maintains a Map of nearby frames
2. **Cache Size**: Limited to 10 frames per video to manage memory
3. **Cache Eviction**: LRU (Least Recently Used) eviction when cache is full
4. **Cache Key**: Frame timestamp rounded to nearest frame boundary
5. **Cache Hit**: Direct return for cached frames without FFmpeg processing

### 7. Cleanup Flow (`clearVideo`)

The single video cleanup process:

1. **Get Sink Data**: Retrieve sink data for the specified mediaId
2. **Terminate FFmpeg**: Properly terminate the FFmpeg instance
3. **Clear Frame Cache**: Clear all cached frames for the video
4. **Remove Sink Data**: Remove sink data from the sinks map
5. **Remove Init Promise**: Remove initialization promise if it exists

### 8. Bulk Cleanup Flow (`clearAll`)

The bulk cleanup process for all videos:

1. **Iterate Through Sinks**: Loop through all sink entries in the sinks map
2. **For Each Sink**:
   - Terminate the FFmpeg instance
   - Clear the frame cache
   - Remove the sink data from the sinks map
3. **Clear Init Promises**: Clear all initialization promises from the initPromises map
4. **Complete Cleanup**: All video data is now cleaned up

## Key Methods

### Public Methods

- `getFrameAt(mediaId, file, time)`: Main entry point for frame extraction
- `clearVideo(mediaId)`: Clean up specific video sink and FFmpeg instance
- `clearAll()`: Clean up all video sinks
- `getStats()`: Get cache statistics (total sinks, active sinks, cached frames)

### Private Methods

- `ensureSink(mediaId, file)`: Ensure FFmpeg sink exists for media
- `initializeSink(mediaId, file)`: Initialize new FFmpeg sink
- `isFrameValid(frame, time)`: Check if frame is valid for timestamp
- `extractFrameSequentially(sinkData, targetTime)`: Extract frame using sequential method
- `extractFrameDirect(sinkData, time)`: Extract frame using direct seek
- `extractFrameWithFFmpeg(sinkData, time)`: Core FFmpeg frame extraction
- `createVideoFrame(canvas, timestamp, duration)`: Create VideoFrame object
- `getFrameFromCache(sinkData, time)`: Retrieve frame from local cache
- `addFrameToCache(sinkData, frame)`: Add frame to local cache with LRU eviction

## Frame Structure

The `getFrameAt()` method returns a `VideoFrame` object with the following structure:

```typescript
interface VideoFrame {
  canvas: HTMLCanvasElement; // Canvas element containing the frame image
  timestamp: number; // Frame timestamp in seconds (precise to frame boundary)
  duration: number; // Frame duration in seconds (1/fps)
  width: number; // Frame width in pixels
  height: number; // Frame height in pixels
}
```

## Performance Optimizations

1. **Lazy Initialization**: FFmpeg instances are only created when first needed
2. **Frame Reuse**: Valid frames are reused without re-extraction
3. **Local Caching**: Nearby frames are cached to avoid repeated FFmpeg calls
4. **Smart Extraction**: Sequential extraction for nearby frames (within 2 seconds)
5. **Memory Management**: LRU cache eviction prevents memory bloat
6. **FFmpeg Optimization**: Uses optimized FFmpeg commands for frame extraction
7. **Canvas Pooling**: Reuses canvas elements when possible

## Error Handling

- **FFmpeg Loading**: Handles FFmpeg.wasm loading failures gracefully
- **File Processing**: Validates video files before processing
- **Frame Extraction**: Gracefully handles frame extraction errors with fallback strategies
- **Memory Errors**: Handles out-of-memory scenarios during frame extraction
- **Initialization Errors**: Properly cleans up failed initialization attempts
- **Cache Errors**: Handles cache corruption and memory issues

## Memory Management

- **Automatic Cleanup**: `clearVideo()` properly terminates FFmpeg instances and clears caches
- **Bulk Cleanup**: `clearAll()` removes all video sinks and clears all caches
- **Cache Eviction**: LRU eviction prevents unlimited memory growth
- **FFmpeg Termination**: Proper FFmpeg instance termination prevents memory leaks
- **Canvas Disposal**: Canvas elements are properly disposed when no longer needed

## FFmpeg Integration

### Dependencies

```typescript
import { FFmpeg } from "@ffmpeg/ffmpeg";
import { fetchFile, toBlobURL } from "@ffmpeg/util";
```

### FFmpeg Configuration

```typescript
const ffmpeg = new FFmpeg();
await ffmpeg.load({
  coreURL: await toBlobURL("/ffmpeg/ffmpeg-core.js", "text/javascript"),
  wasmURL: await toBlobURL("/ffmpeg/ffmpeg-core.wasm", "application/wasm"),
});
```

### Frame Extraction Command

```bash
ffmpeg -i input.mp4 -vf "select=eq(n\\,FRAME_NUMBER)" -vframes 1 -f image2 -y output.png
```

Where `FRAME_NUMBER` is calculated as: `Math.floor(timestamp * fps)`

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

The video cache uses the following FFmpeg configuration:

- **Core URL**: `/ffmpeg/ffmpeg-core.js`
- **WASM URL**: `/ffmpeg/ffmpeg-core.wasm`
- **Frame Cache Size**: 10 frames per video
- **Sequential Threshold**: 2 seconds
- **Frame Format**: PNG for optimal quality
- **Output Format**: HTMLCanvasElement for direct rendering

## Statistics

The `getStats()` method provides:

- `totalSinks`: Number of initialized FFmpeg sinks
- `activeSinks`: Number of sinks with active frame caches
- `cachedFrames`: Total number of cached frames across all videos
- `memoryUsage`: Estimated memory usage in MB
- `averageCacheHitRate`: Percentage of cache hits vs misses

## Advantages over MediaBunny

1. **Better Format Support**: FFmpeg supports virtually all video formats
2. **More Control**: Direct control over frame extraction parameters
3. **Better Performance**: Optimized FFmpeg commands for specific use cases
4. **Cross-Platform**: FFmpeg.wasm works consistently across platforms
5. **Extensibility**: Easy to add new video processing features
6. **Memory Efficiency**: Better control over memory usage and cleanup
7. **Error Recovery**: More robust error handling and recovery mechanisms

## Implementation Considerations

### FFmpeg Loading

- FFmpeg.wasm files must be served from the public directory
- Loading is asynchronous and may take time on first use
- Consider preloading FFmpeg for better user experience

### File Handling

- Video files are written to FFmpeg's virtual filesystem
- Large files may require chunked processing
- Consider file size limits for optimal performance

### Browser Compatibility

- Requires WebAssembly support
- Some older browsers may not support FFmpeg.wasm
- Consider fallback strategies for unsupported browsers

### Performance Monitoring

- Monitor memory usage with large video files
- Track frame extraction performance
- Implement performance metrics and alerts

This FFmpeg-based implementation provides a robust, high-performance alternative to MediaBunny while maintaining the same API surface and performance characteristics.
