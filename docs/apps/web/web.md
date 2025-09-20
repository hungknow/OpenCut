# OpenCut Web Application

OpenCut is a free, open-source video editor built with Next.js 15, React 18, and TypeScript.

## Architecture

**Framework**: Next.js 15 with App Router  
**UI**: React 18 + TypeScript + Tailwind CSS  
**State**: Zustand stores  
**Database**: PostgreSQL + Drizzle ORM  
**Auth**: Better Auth  
**Video Processing**: FFmpeg.wasm

## File Structure

### `/src/app/` - Next.js App Router

- `page.tsx` - Landing page (Hero + Header + Footer)
- `layout.tsx` - Root layout with providers
- `editor/[project_id]/page.tsx` - Main video editor interface
- `(auth)/` - Authentication pages (login/signup)
- `blog/`, `roadmap/`, `projects/` - Marketing pages

### `/src/components/` - React Components

#### Editor Components (`/editor/`)

- `timeline/` - Video timeline with tracks, playhead, zoom
- `media-panel/` - Media library with tabs (media, sounds, text, stickers)
- `preview-panel.tsx` - Video preview with playback controls
- `properties-panel/` - Element property editors
- `editor-header.tsx` - Top toolbar with project controls

#### UI Components (`/ui/`)

- Radix UI primitives (Button, Dialog, Sheet, etc.)
- Custom components with Tailwind styling
- Form components with React Hook Form

#### Layout Components

- `header.tsx` - Navigation header
- `footer.tsx` - Site footer
- `landing/` - Landing page components

### `/src/stores/` - State Management (Zustand)

- `timeline-store.ts` - Timeline tracks, elements, selection
- `project-store.ts` - Project management, canvas settings
- `media-store.ts` - Media file management
- `playback-store.ts` - Video playback state
- `scene-store.ts` - Multi-scene support

### `/src/hooks/` - Custom React Hooks

- Timeline interactions (zoom, snapping, selection)
- Playback controls and keyboard shortcuts
- Drag & drop functionality

## Key Features

**Video Editor Core**:

- Multi-track timeline with video/audio/text layers
- Drag & drop media import
- Real-time preview with FFmpeg.wasm
- Resizable panel layout
- Multi-scene support

**Media Management**:

- File upload and processing
- Sound library integration
- Text and sticker overlays
- Export functionality

**User Experience**:

- Keyboard shortcuts
- Responsive design
- Dark/light theme support
- Project persistence
