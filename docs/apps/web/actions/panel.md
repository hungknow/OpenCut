# Panel Store Documentation

## Overview

The `panel-store.ts` manages the layout and sizing of various panels in the video editor interface. It provides a flexible system for managing panel sizes with preset configurations and custom overrides.

## State Structure

### Core State Properties

```typescript
interface PanelState {
  // Panel sizes (in percentage or units)
  toolsPanel: number;
  previewPanel: number;
  propertiesPanel: number;
  mainContent: number;
  timeline: number;

  // Preset management
  activePreset: PanelPreset;
  presetCustomSizes: Record<PanelPreset, Partial<PanelSizes>>;
  resetCounter: number;

  // Media view configuration
  mediaViewMode: "grid" | "list";
}
```

### Panel Presets

The store supports four predefined panel layout presets:

| Preset             | Tools Panel | Preview Panel | Properties Panel | Main Content | Timeline |
| ------------------ | ----------- | ------------- | ---------------- | ------------ | -------- |
| `default`          | 25%         | 50%           | 25%              | 70%          | 30%      |
| `media`            | 30%         | 45%           | 25%              | 100%         | 25%      |
| `inspector`        | 30%         | 70%           | 30%              | 75%          | 25%      |
| `vertical-preview` | 30%         | 40%           | 30%              | 75%          | 25%      |

## Function Analysis

### Panel Size Setters

| Function                   | State Modified                                                       | Logic Flow                                                                                                                                                                      |
| -------------------------- | -------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `setToolsPanel(size)`      | `toolsPanel`, `presetCustomSizes[activePreset].toolsPanel`           | 1. Get current `activePreset` and `presetCustomSizes`<br>2. Update `toolsPanel` with new size<br>3. Update `presetCustomSizes[activePreset].toolsPanel` with new size           |
| `setPreviewPanel(size)`    | `previewPanel`, `presetCustomSizes[activePreset].previewPanel`       | 1. Get current `activePreset` and `presetCustomSizes`<br>2. Update `previewPanel` with new size<br>3. Update `presetCustomSizes[activePreset].previewPanel` with new size       |
| `setPropertiesPanel(size)` | `propertiesPanel`, `presetCustomSizes[activePreset].propertiesPanel` | 1. Get current `activePreset` and `presetCustomSizes`<br>2. Update `propertiesPanel` with new size<br>3. Update `presetCustomSizes[activePreset].propertiesPanel` with new size |
| `setMainContent(size)`     | `mainContent`, `presetCustomSizes[activePreset].mainContent`         | 1. Get current `activePreset` and `presetCustomSizes`<br>2. Update `mainContent` with new size<br>3. Update `presetCustomSizes[activePreset].mainContent` with new size         |
| `setTimeline(size)`        | `timeline`, `presetCustomSizes[activePreset].timeline`               | 1. Get current `activePreset` and `presetCustomSizes`<br>2. Update `timeline` with new size<br>3. Update `presetCustomSizes[activePreset].timeline` with new size               |

### Media View Management

| Function                 | State Modified  | Logic Flow                                                           |
| ------------------------ | --------------- | -------------------------------------------------------------------- |
| `setMediaViewMode(mode)` | `mediaViewMode` | 1. Directly set `mediaViewMode` to provided value ("grid" or "list") |

### Preset Management

| Function                  | State Modified                                                       | Logic Flow                                                                                                                                                                                                                              |
| ------------------------- | -------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `setActivePreset(preset)` | `activePreset`, `presetCustomSizes`, all panel sizes                 | 1. Save current panel sizes to `presetCustomSizes[currentPreset]`<br>2. Get default sizes for new preset from `PRESET_CONFIGS[preset]`<br>3. Merge with any custom sizes for the preset<br>4. Update all panel sizes and `activePreset` |
| `resetPreset(preset)`     | `presetCustomSizes[preset]`, `resetCounter`, panel sizes (if active) | 1. Clear custom sizes for the specified preset<br>2. Increment `resetCounter`<br>3. If preset is currently active, apply default sizes to all panels                                                                                    |
| `getCurrentPresetSizes()` | None (read-only)                                                     | 1. Return current panel sizes as `PanelSizes` object                                                                                                                                                                                    |

## State Persistence

The store uses Zustand's `persist` middleware with the key `"panel-sizes"` to automatically save and restore panel configurations across browser sessions.

## Example State Configurations

### Default State (Initial Load)

```typescript
{
  toolsPanel: 25,
  previewPanel: 50,
  propertiesPanel: 25,
  mainContent: 70,
  timeline: 30,
  activePreset: "default",
  presetCustomSizes: {
    default: {},
    media: {},
    inspector: {},
    "vertical-preview": {}
  },
  resetCounter: 0,
  mediaViewMode: "grid"
}
```

### After Switching to Media Preset

```typescript
{
  toolsPanel: 30,
  previewPanel: 45,
  propertiesPanel: 25,
  mainContent: 100,
  timeline: 25,
  activePreset: "media",
  presetCustomSizes: {
    default: {
      toolsPanel: 25,
      previewPanel: 50,
      propertiesPanel: 25,
      mainContent: 70,
      timeline: 30
    },
    media: {},
    inspector: {},
    "vertical-preview": {}
  },
  resetCounter: 0,
  mediaViewMode: "grid"
}
```

### After Customizing Inspector Preset

```typescript
{
  toolsPanel: 35,
  previewPanel: 65,
  propertiesPanel: 35,
  mainContent: 80,
  timeline: 20,
  activePreset: "inspector",
  presetCustomSizes: {
    default: {},
    media: {},
    inspector: {
      toolsPanel: 35,
      previewPanel: 65,
      propertiesPanel: 35,
      mainContent: 80,
      timeline: 20
    },
    "vertical-preview": {}
  },
  resetCounter: 0,
  mediaViewMode: "list"
}
```

### After Resetting Inspector Preset

```typescript
{
  toolsPanel: 30,
  previewPanel: 70,
  propertiesPanel: 30,
  mainContent: 75,
  timeline: 25,
  activePreset: "inspector",
  presetCustomSizes: {
    default: {},
    media: {},
    inspector: {},
    "vertical-preview": {}
  },
  resetCounter: 1,
  mediaViewMode: "list"
}
```
