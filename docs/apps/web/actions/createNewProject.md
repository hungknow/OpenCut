# createNewProject Action Documentation

## Overview

The `createNewProject` function is a core action in the project store that creates a new video editing project with default settings and initializes all related stores.

## Function Signature

```typescript
createNewProject: (name: string) => Promise<string>;
```

## Logic Flow

### 1. Project Creation

<a id="project-creation"></a>

- Creates a new project object with default values (local variable only)
- No state changes yet - project exists only in memory
- Default values set:
  - Canvas size: 1920x1080 (DEFAULT_CANVAS_SIZE)
  - FPS: 30 (DEFAULT_FPS)
  - Background: Black color (#000000)
  - Background type: "color"
  - Blur intensity: 8
  - Creates a main scene with default settings
  - `createDefaultProject(name)`

### 2. Set Active Project

<a id="set-active-project"></a>

- Project is now the active project in the store
- `set({ activeProject: newProject })`

### 3. Clear Related Stores

<a id="clear-related-stores"></a>

- Clears all related stores before setting up new project
- Ensures clean state for the new project
- `useMediaStore.getState()`
- `useTimelineStore.getState()`
- `useSceneStore.getState()`
- `mediaStore.clearAllMedia()`
- `timelineStore.clearTimeline()`

### 4. Initialize Scene Store

<a id="initialize-scene-store"></a>

- Sets up the scene store with the new project's main scene
- `sceneStore.initializeScenes({ scenes, currentSceneId })`

### 5. Save to Storage

<a id="save-to-storage"></a>

- Project data persisted to IndexedDB
- No immediate state changes - storage is external
- `storageService.saveProject({ project: newProject })`

### 6. Reload All Projects

<a id="reload-all-projects"></a>

- Project list is refreshed to include the new project
- `get().loadAllProjects()`

### 7. Return Project ID

<a id="return-project-id"></a>

- Returns the newly created project ID for navigation
- No state changes - just returns the ID
- `return newProject.id`

### 8. Error Handling

<a id="error-handling"></a>

- Toast notification displayed to user
- Error is re-thrown for upstream handling
- `toast.error("Failed to save new project")`
- `throw error`

## Store State Changes Table

| Action/Event                                          | Project Store                                         | Media Store                                        | Timeline Store                                           | Scene Store                                      |
| ----------------------------------------------------- | ----------------------------------------------------- | -------------------------------------------------- | -------------------------------------------------------- | ------------------------------------------------ |
| **App Initialization**                                | `activeProject: null`<br>`savedProjects: [...]`       | `mediaFiles: [...]`<br>`isLoading: false`          | `_tracks: [...]`<br>`_playheadPosition: 0`<br>`_zoom: 1` | `scenes: [...]`<br>`currentScene: {...}`         |
| **[Project Creation](#project-creation)**             | _(unchanged)_                                         | _(unchanged)_                                      | _(unchanged)_                                            | _(unchanged)_                                    |
| **[Set Active Project](#set-active-project)**         | `activeProject: {...}`<br>`isLoading: false`          | _(unchanged)_                                      | _(unchanged)_                                            | _(unchanged)_                                    |
| **[Clear Related Stores](#clear-related-stores)**     | _(unchanged)_                                         | `mediaFiles: []` _(cleared)_<br>`isLoading: false` | `_tracks: []` _(cleared)_<br>`_playheadPosition: 0`      | `scenes: []` _(cleared)_<br>`currentScene: null` |
| **[Initialize Scene Store](#initialize-scene-store)** | _(unchanged)_                                         | _(unchanged)_                                      | _(unchanged)_                                            | `scenes: [...]`<br>`currentScene: {...}`         |
| **[Save to Storage](#save-to-storage)**               | _(unchanged)_                                         | _(unchanged)_                                      | _(unchanged)_                                            | _(unchanged)_                                    |
| **[Reload All Projects](#reload-all-projects)**       | `savedProjects: [...]` _(updated)_                    | _(unchanged)_                                      | _(unchanged)_                                            | _(unchanged)_                                    |
| **[Return Project ID](#return-project-id)**           | _(unchanged)_                                         | _(unchanged)_                                      | _(unchanged)_                                            | _(unchanged)_                                    |
| **[Error Handling](#error-handling)**                 | _(unchanged)_<br>`activeProject: {...}` _(preserved)_ | _(unchanged)_                                      | _(unchanged)_                                            | _(unchanged)_                                    |

## React Components That Trigger This Logic

### 1. Projects Page (`/apps/web/src/app/projects/page.tsx`)

**Trigger Components:**

- `CreateButton` component (lines 597-604)
- `NoProjects` component (lines 606-623)

**Usage:**

```typescript
const handleCreateProject = async () => {
  const projectId = await createNewProject("New Project");
  console.log("projectId", projectId);
  router.push(`/editor/${projectId}`);
};
```

**Rendered Elements:**

- **Desktop**: "New project" button in the top-right corner (line 217)
- **Mobile**: "New project" button in the header (line 171)
- **Empty State**: "Create Your First Project" button when no projects exist (line 617)

### 2. Editor Page (`/apps/web/src/app/editor/[project_id]/page.tsx`)

**Trigger Condition:**

- When a project ID is not found or doesn't exist (lines 107-122)

**Usage:**

```typescript
if (isProjectNotFound) {
  markProjectIdAsInvalid(projectId);
  try {
    const newProjectId = await createNewProject("Untitled Project");
    router.replace(`/editor/${newProjectId}`);
  } catch (createError) {
    console.error("Failed to create new project:", createError);
  }
}
```

**Rendered Elements:**

- Automatically redirects to the new project's editor page
- No visible UI elements - this is a fallback mechanism

## UI Flow and User Experience

### 1. From Projects Page

**State Changes:**

1. User clicks "New project" button
2. `createNewProject("New Project")` is called
   - **Project Store**: `activeProject` → New project object
   - **Media Store**: All media cleared
   - **Timeline Store**: Timeline cleared
   - **Scene Store**: Initialized with main scene
   - **Project Store**: `savedProjects` → Updated with new project
3. New project is created and saved
4. User is redirected to `/editor/{newProjectId}`
5. Editor page loads with the new project

### 2. From Editor Page (Error Recovery)

**State Changes:**

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
   - **Project Store**: `savedProjects` → Updated with new project
5. User is redirected to `/editor/{newProjectId}` with the new project

### 3. Empty State

**State Changes:**

1. User visits projects page with no existing projects
   - **Project Store**: `savedProjects` → Empty array
2. "No projects yet" message is displayed
3. "Create Your First Project" button is shown
4. Clicking triggers the same flow as the regular "New project" button
   - Same state changes as scenario 1

## Related Store Dependencies

- **Project Store**: Manages project state and persistence
- **Media Store**: Cleared when creating new project
- **Timeline Store**: Cleared when creating new project
- **Scene Store**: Initialized with new project's scenes
- **Storage Service**: Handles project persistence

## Error Scenarios

1. **Storage Failure**: Toast error shown, error re-thrown
2. **Invalid Project ID**: Automatic fallback to create new project
3. **Network Issues**: Handled by storage service layer

## Detailed Store State Examples

### Before createNewProject

```typescript
// Project Store
{
  activeProject: null,
  savedProjects: [...],
  isLoading: false,
  isInitialized: true,
  invalidProjectIds: new Set()
}

// Media Store
{
  mediaFiles: [...],
  isLoading: false
}

// Timeline Store
{
  _tracks: [...],
  _playheadPosition: 0,
  _zoom: 1,
  _scrollPosition: 0,
  _isPlaying: false,
  _duration: 0
}

// Scene Store
{
  scenes: [...],
  currentScene: {...},
  isLoading: false
}
```

### After createNewProject Success

```typescript
// Project Store
{
  activeProject: {
    id: "new-project-uuid",
    name: "New Project",
    thumbnail: "",
    createdAt: "2024-01-01T00:00:00.000Z",
    updatedAt: "2024-01-01T00:00:00.000Z",
    scenes: [mainScene],
    currentSceneId: mainScene.id,
    backgroundColor: "#000000",
    backgroundType: "color",
    blurIntensity: 8,
    bookmarks: [],
    fps: 30,
    canvasSize: { width: 1920, height: 1080 },
    canvasMode: "preset"
  },
  savedProjects: [...], // Updated with new project
  isLoading: false,
  isInitialized: true,
  invalidProjectIds: new Set()
}

// Media Store
{
  mediaFiles: [], // Cleared
  isLoading: false
}

// Timeline Store
{
  _tracks: [], // Cleared
  _playheadPosition: 0,
  _zoom: 1,
  _scrollPosition: 0,
  _isPlaying: false,
  _duration: 0
}

// Scene Store
{
  scenes: [{
    id: "main-scene-uuid",
    name: "Main Scene",
    isMain: true,
    createdAt: "2024-01-01T00:00:00.000Z",
    updatedAt: "2024-01-01T00:00:00.000Z"
  }],
  currentScene: {
    id: "main-scene-uuid",
    name: "Main Scene",
    isMain: true,
    createdAt: "2024-01-01T00:00:00.000Z",
    updatedAt: "2024-01-01T00:00:00.000Z"
  },
  isLoading: false
}
```
