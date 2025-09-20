# Invalid Project IDs State

## Overview

The `invalidProjectIds` state is a global tracking mechanism in the project store that prevents race conditions and handles error scenarios when loading projects that don't exist or have been corrupted.

## State Structure

```typescript
interface ProjectStore {
  // ... other properties
  invalidProjectIds?: Set<string>;

  // Methods for managing invalid project IDs
  isInvalidProjectId: (id: string) => boolean;
  markProjectIdAsInvalid: (id: string) => void;
  clearInvalidProjectIds: () => void;
}
```

The state is implemented as a `Set<string>` that stores project IDs that have been identified as invalid, corrupted, or non-existent.

## Purpose

### 1. Race Condition Prevention

- Prevents multiple simultaneous attempts to load the same invalid project
- Avoids duplicate project creation when a project ID is invalid
- Ensures consistent error handling across the application

### 2. Error State Management

- Tracks project IDs that have failed to load due to:
  - Project not found in storage
  - Corrupted project data
  - Network/storage errors during load
- Provides a centralized way to check if a project ID should be avoided

### 3. User Experience Optimization

- Prevents infinite loading loops
- Enables graceful fallback to project creation
- Reduces unnecessary API calls and storage operations

## Usage Patterns

### Primary Usage Location

The main usage is in the editor page (`/editor/[project_id]/page.tsx`):

```typescript
const { isInvalidProjectId, markProjectIdAsInvalid } = useProjectStore();

// Check if project ID is invalid before attempting to load
if (isInvalidProjectId(projectId)) {
  return; // Skip loading attempt
}

// Mark project as invalid when load fails
if (isProjectNotFound) {
  markProjectIdAsInvalid(projectId);
  // Create new project as fallback
  const newProjectId = await createNewProject("Untitled Project");
  router.replace(`/editor/${newProjectId}`);
}
```

### Error Handling Flow

1. **Initial Check**: Before attempting to load a project, check if it's already marked as invalid
2. **Load Attempt**: If not invalid, attempt to load the project via `storageService.loadProject()`
3. **Error Detection**: If project is not found (returns `null`), mark it as invalid
4. **Fallback Action**: Create a new project and redirect to it
5. **Prevention**: Future attempts to load the same invalid ID are blocked

## Methods

### `isInvalidProjectId(id: string): boolean`

- **Purpose**: Checks if a project ID is in the invalid set
- **Returns**: `true` if the ID is marked as invalid, `false` otherwise
- **Usage**: Used before attempting to load a project to prevent unnecessary operations

### `markProjectIdAsInvalid(id: string): void`

- **Purpose**: Adds a project ID to the invalid set
- **Behavior**: Creates a new Set with the existing invalid IDs plus the new one
- **Usage**: Called when a project fails to load or is found to be corrupted

### `clearInvalidProjectIds(): void`

- **Purpose**: Clears all invalid project IDs from the set
- **Behavior**: Resets the `invalidProjectIds` to an empty Set
- **Usage**: Can be used to reset the invalid state (e.g., after storage cleanup)

## Implementation Details

### State Initialization

```typescript
export const useProjectStore = create<ProjectStore>((set, get) => ({
  // ... other state
  invalidProjectIds: new Set<string>(),
  // ... rest of implementation
}));
```

### Method Implementations

```typescript
isInvalidProjectId: (id: string) => {
  const invalidIds = get().invalidProjectIds || new Set();
  return invalidIds.has(id);
},

markProjectIdAsInvalid: (id: string) => {
  set((state) => ({
    invalidProjectIds: new Set([
      ...(state.invalidProjectIds || new Set()),
      id,
    ]),
  }));
},

clearInvalidProjectIds: () => {
  set({ invalidProjectIds: new Set() });
},
```

## Error Scenarios Handled

### 1. Project Not Found

- **Trigger**: `storageService.loadProject()` returns `null`
- **Action**: Mark project ID as invalid, create new project
- **Prevention**: Future loads of same ID are blocked

### 2. Corrupted Project Data

- **Trigger**: Project exists but data is malformed
- **Action**: Mark as invalid, provide fallback
- **Prevention**: Avoids repeated attempts to load corrupted data

### 3. Storage Errors

- **Trigger**: IndexedDB or OPFS errors during load
- **Action**: Mark as invalid, create new project
- **Prevention**: Avoids retrying failed storage operations

## Integration with Other Systems

### Project Store Integration

- Part of the main project store state
- Accessed via `useProjectStore()` hook
- Persists across component re-renders

### Editor Page Integration

- Used in the main editor initialization logic
- Prevents duplicate project creation
- Enables graceful error recovery

### Storage Service Integration

- Works with `storageService.loadProject()` return values
- Handles both `null` returns and thrown errors
- Provides fallback when storage operations fail

## Best Practices

1. **Always Check First**: Use `isInvalidProjectId()` before attempting to load any project
2. **Mark Early**: Mark project as invalid as soon as an error is detected
3. **Provide Fallback**: Always have a fallback action (like creating a new project)
4. **Clear When Appropriate**: Consider clearing invalid IDs after storage cleanup or app restart
5. **Handle Race Conditions**: The invalid ID check should be the first check in any load sequence

## Related Files

- `apps/web/src/stores/project-store.ts` - Main implementation
- `apps/web/src/app/editor/[project_id]/page.tsx` - Primary usage
- `apps/web/src/lib/storage/storage-service.ts` - Storage integration
- `docs/apps/web/actions/createNewProject.md` - Related documentation
