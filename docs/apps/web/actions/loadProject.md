# LoadProject Action Analysis

## Overview

The `loadProject` action is a function that loads a complete project from storage, including all associated data (media, timeline, scenes) and initializes the application state for editing.

## Function Signature

```typescript
loadProject: (id: string) => Promise<void>;
```

## Logic Flow

### 1. <a id="initialization-check">Initialization Check</a>

- Sets loading state if the store hasn't been initialized yet
- Prevents UI flicker during initial app load
  - `get().isInitialized`
  - `set({ isLoading: true })`

### 2. <a id="store-cleanup">Store Cleanup (Prevent Flicker)</a>

- Clears all related stores before loading new project
- Prevents visual artifacts when switching between projects
- Ensures clean state for the new project
  - `useMediaStore.getState()`
  - `mediaStore.clearAllMedia()`
  - `timelineStore.clearTimeline()`
  - `sceneStore.clearScenes()`

### 3. <a id="project-loading">Project Loading</a>

- Loads project data from storage service
- Sets the active project in the store
- Throws error if project not found
  - `storageService.loadProject({ id })`
  - `set({ activeProject: project })`
  - `throw new Error("Project with id ${id} not found")`

### 4. <a id="scene-initialization">Scene Initialization</a>

- Initializes scene store with project scenes
- Determines current scene using fallback logic:
  1. Project's currentSceneId
  2. Main scene (isMain: true)
  3. First scene in array
  - `sceneStore.initializeScenes({ scenes, currentSceneId })`
  - `project.scenes.find()` (currentSceneId)
  - `project.scenes.find()` (isMain)
  - `project.scenes[0]` (fallback)

### 5. <a id="parallel-data-loading">Parallel Data Loading</a>

- Loads media files and timeline data in parallel
- Uses current scene ID for timeline loading
- Optimizes performance by avoiding sequential loading
  - `Promise.all([...])`
  - `mediaStore.loadProjectMedia(id)`
  - `timelineStore.loadProjectTimeline({ projectId, sceneId })`

### 6. <a id="error-handling-cleanup">Error Handling & Cleanup</a>

- Logs errors for debugging
- Re-throws errors for UI error handling
- Always sets loading to false
  - `console.error("Failed to load project:", error)`
  - `throw error` (re-throw)
  - `set({ isLoading: false })`

## Store State Changes Table

| Action/Event                                            | Project Store                                | Media Store                                              | Timeline Store                                        | Scene Store                                      |
| ------------------------------------------------------- | -------------------------------------------- | -------------------------------------------------------- | ----------------------------------------------------- | ------------------------------------------------ |
| **App Initialization**                                  | `isLoading: true`<br>`isInitialized: false`  | `mediaFiles: []`<br>`isLoading: false`                   | `_tracks: []`<br>`_playheadPosition: 0`<br>`_zoom: 1` | `scenes: []`<br>`currentScene: null`             |
| **[1] [Initialization Check](#initialization-check)**   | `isLoading: true`<br>`isInitialized: true`   | _(unchanged)_                                            | _(unchanged)_                                         | _(unchanged)_                                    |
| **[2] [Store Cleanup](#store-cleanup)**                 | _(unchanged)_                                | `mediaFiles: []` _(cleared)_<br>`isLoading: true`        | `_tracks: []` _(cleared)_<br>`_playheadPosition: 0`   | `scenes: []` _(cleared)_<br>`currentScene: null` |
| **[3] [Project Loading](#project-loading)**             | `activeProject: {...}`<br>`isLoading: false` | _(unchanged)_                                            | _(unchanged)_                                         | _(unchanged)_                                    |
| **[4] [Scene Initialization](#scene-initialization)**   | _(unchanged)_                                | _(unchanged)_                                            | _(unchanged)_                                         | `scenes: [...]`<br>`currentScene: {...}`         |
| **[5] [Parallel Data Loading](#parallel-data-loading)** | _(unchanged)_                                | `mediaFiles: [...]`<br>`isLoading: false`                | `_tracks: [...]`<br>`_duration: calculated`           | _(unchanged)_                                    |
| **[6] [Error Handling](#error-handling-cleanup)**       | `isLoading: false`<br>`activeProject: null`  | `mediaFiles: []`<br>`isLoading: false`<br>`error: "..."` | `_tracks: []`<br>`_duration: 0`                       | `scenes: []`<br>`currentScene: null`             |

## Detailed Store State Examples

### Before LoadProject

```typescript
// Project Store
{
  activeProject: null,
  savedProjects: [...],
  isLoading: true,
  isInitialized: false,
  invalidProjectIds: new Set()
}

// Media Store
{
  mediaFiles: [],
  isLoading: false,
  error: null
}

// Timeline Store
{
  _tracks: [],
  _playheadPosition: 0,
  _zoom: 1,
  _scrollPosition: 0,
  _isPlaying: false,
  _duration: 0
}

// Scene Store
{
  scenes: [],
  currentScene: null,
  isLoading: false
}
```

### During LoadProject

```typescript
// Project Store
{
  activeProject: null, // Still null during loading
  savedProjects: [...],
  isLoading: true,
  isInitialized: true,
  invalidProjectIds: new Set()
}

// Media Store
{
  mediaFiles: [], // Cleared by clearAllMedia()
  isLoading: true, // Set during loadProjectMedia()
  error: null
}

// Timeline Store
{
  _tracks: [], // Cleared by clearTimeline()
  _playheadPosition: 0,
  _zoom: 1,
  _scrollPosition: 0,
  _isPlaying: false,
  _duration: 0
}

// Scene Store
{
  scenes: [], // Cleared by clearScenes()
  currentScene: null,
  isLoading: false
}
```

### After Successful LoadProject

```typescript
// Project Store
{
  activeProject: {
    id: "project-123",
    name: "My Video Project",
    thumbnail: "",
    createdAt: "2024-01-01T00:00:00.000Z",
    updatedAt: "2024-01-01T12:00:00.000Z",
    scenes: [
      {
        id: "scene-1",
        name: "Main Scene",
        isMain: true,
        createdAt: "2024-01-01T00:00:00.000Z",
        updatedAt: "2024-01-01T00:00:00.000Z"
      }
    ],
    currentSceneId: "scene-1",
    backgroundColor: "#000000",
    backgroundType: "color",
    blurIntensity: 8,
    bookmarks: [],
    fps: 30,
    canvasSize: { width: 1920, height: 1080 },
    canvasMode: "preset"
  },
  savedProjects: [...],
  isLoading: false,
  isInitialized: true,
  invalidProjectIds: new Set()
}

// Media Store
{
  mediaFiles: [
    {
      id: "media-1",
      name: "video-sample.mp4",
      type: "video",
      url: "blob:...",
      duration: 10.5,
      width: 1920,
      height: 1080,
      thumbnail: "blob:...",
      projectId: "project-123",
      createdAt: "2024-01-01T00:00:00.000Z",
      updatedAt: "2024-01-01T00:00:00.000Z"
    },
    {
      id: "media-2",
      name: "audio-track.mp3",
      type: "audio",
      url: "blob:...",
      duration: 30.0,
      projectId: "project-123",
      createdAt: "2024-01-01T00:00:00.000Z",
      updatedAt: "2024-01-01T00:00:00.000Z"
    }
  ],
  isLoading: false,
  error: null
}

// Timeline Store
{
  _tracks: [
    {
      id: "track-1",
      type: "video",
      name: "Video Track 1",
      elements: [
        {
          id: "element-1",
          mediaId: "media-1",
          startTime: 0,
          endTime: 10.5,
          trackId: "track-1",
          projectId: "project-123",
          sceneId: "scene-1",
          createdAt: "2024-01-01T00:00:00.000Z",
          updatedAt: "2024-01-01T00:00:00.000Z"
        }
      ],
      projectId: "project-123",
      sceneId: "scene-1",
      createdAt: "2024-01-01T00:00:00.000Z",
      updatedAt: "2024-01-01T00:00:00.000Z"
    },
    {
      id: "track-2",
      type: "audio",
      name: "Audio Track 1",
      elements: [
        {
          id: "element-2",
          mediaId: "media-2",
          startTime: 0,
          endTime: 30.0,
          trackId: "track-2",
          projectId: "project-123",
          sceneId: "scene-1",
          createdAt: "2024-01-01T00:00:00.000Z",
          updatedAt: "2024-01-01T00:00:00.000Z"
        }
      ],
      projectId: "project-123",
      sceneId: "scene-1",
      createdAt: "2024-01-01T00:00:00.000Z",
      updatedAt: "2024-01-01T00:00:00.000Z"
    }
  ],
  _playheadPosition: 0,
  _zoom: 1,
  _scrollPosition: 0,
  _isPlaying: false,
  _duration: 30.0
}

// Scene Store
{
  scenes: [
    {
      id: "scene-1",
      name: "Main Scene",
      isMain: true,
      createdAt: "2024-01-01T00:00:00.000Z",
      updatedAt: "2024-01-01T00:00:00.000Z"
    }
  ],
  currentScene: {
    id: "scene-1",
    name: "Main Scene",
    isMain: true,
    createdAt: "2024-01-01T00:00:00.000Z",
    updatedAt: "2024-01-01T00:00:00.000Z"
  },
  isLoading: false
}
```

## Store Interactions

### Media Store Integration

- **Clear**: `mediaStore.clearAllMedia()` - Removes all media files
- **Load**: `mediaStore.loadProjectMedia(id)` - Loads project-specific media
- **State**: Manages `mediaFiles[]` array with video/image/audio files

### Timeline Store Integration

- **Clear**: `timelineStore.clearTimeline()` - Resets timeline tracks
- **Load**: `timelineStore.loadProjectTimeline({ projectId, sceneId })` - Loads timeline data
- **State**: Manages `_tracks[]` array with timeline elements

### Scene Store Integration

- **Clear**: `sceneStore.clearScenes()` - Removes all scenes
- **Initialize**: `sceneStore.initializeScenes({ scenes, currentSceneId })` - Sets up scenes
- **State**: Manages `scenes[]` array and `currentScene` reference

## UI Flow and User Experience

### 1. From Projects Page

**State Changes:**

1. User clicks on a project from the projects list
2. `loadProject(projectId)` is called
   - **Project Store**: `isLoading` → `true`
   - **Media Store**: All media cleared
   - **Timeline Store**: Timeline cleared
   - **Scene Store**: Scenes cleared
3. Project data is loaded from storage
   - **Project Store**: `activeProject` → Project object
4. Scene store is initialized
   - **Scene Store**: `scenes` → Project scenes, `currentScene` → Main scene
5. Media and timeline data load in parallel
   - **Media Store**: `mediaFiles` → Project media
   - **Timeline Store**: `_tracks` → Project timeline
6. User is redirected to `/editor/{projectId}`
7. Editor page loads with the project data

### 2. Direct URL Navigation

**State Changes:**

1. User navigates directly to `/editor/{projectId}`
2. System attempts to load the project
   - **Project Store**: `isLoading` → `true`
3. Project loading succeeds
   - Same state changes as scenario 1
4. Editor page displays the loaded project

### 3. Error Recovery (Project Not Found)

1. User navigates to `/editor/{invalidProjectId}`
2. System attempts to load the project
   - **Project Store**: `isLoading` → `true`
3. Project not found error occurs
   - **Project Store**: `invalidProjectIds` → Invalid ID added to set
4. `createNewProject("Untitled Project")` is called automatically
   - **Project Store**: `activeProject` → New project object
   - **Media Store**: All media cleared
   - **Timeline Store**: Timeline cleared
   - **Scene Store**: Initialized with main scene
5. User is redirected to `/editor/{newProjectId}` with the new project

### 4. Error Recovery (Storage Issues)

1. User navigates to `/editor/{projectId}`
2. System attempts to load the project
   - **Project Store**: `isLoading` → `true`
3. Storage error occurs (IndexedDB/OPFS issues)
   - **Project Store**: `isLoading` → `false`
   - Error is logged and re-thrown
4. UI shows error message or fallback behavior
5. User may retry or navigate away

## Error Scenarios

### 1. Project Not Found

**Trigger**: `storageService.loadProject()` returns `null`
**Error**: `"Project with id ${id} not found"`
**UI Handling**:

- Mark project ID as invalid
- Create new project as fallback
- Redirect to new project

### 2. Storage Service Errors

**Trigger**: IndexedDB/OPFS access failures
**Error**: Various storage-related errors
**UI Handling**:

- Log error for debugging
- Re-throw for UI error handling
- May trigger fallback project creation

### 3. Scene Initialization Errors

**Trigger**: Invalid scene data or missing main scene
**Error**: Handled by `sceneStore.initializeScenes()`
**UI Handling**:

- Scene store handles gracefully
- Ensures main scene exists
- Falls back to first available scene

### 4. Media Loading Errors

**Trigger**: Corrupted media files or storage issues
**Error**: Handled by `mediaStore.loadProjectMedia()`
**UI Handling**:

- Media store logs errors
- Continues loading other media
- Shows partial media list

### 5. Timeline Loading Errors

**Trigger**: Corrupted timeline data
**Error**: Handled by `timelineStore.loadProjectTimeline()`
**UI Handling**:

- Timeline store creates default tracks
- Logs error for debugging
- Ensures timeline is functional

### 6. Race Condition Errors

**Trigger**: Multiple simultaneous load attempts
**UI Prevention**:

- `isInvalidProjectId()` check
- `handledProjectIds` local tracking
- `isInitializingRef` flag
