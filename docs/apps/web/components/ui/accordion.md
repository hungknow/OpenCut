# Radix UI Component Extension Template

## Accordion Component Analysis

This document serves as a template for extending Radix UI components in the OpenCut project.

### Base Radix UI vs Custom Implementation

**Base Radix UI Accordion**:

- Raw primitives: `Root`, `Item`, `Trigger`, `Content`
- No styling or visual enhancements
- Basic functionality only

**Custom OpenCut Accordion**:

- Enhanced styling with Tailwind CSS
- Custom animations and transitions
- Icon integration with state-based rotation
- Improved accessibility and UX

### Key Modifications Made

#### 1. **Component Structure**

```tsx
// Base: Direct primitive usage
import { Accordion as AccordionPrimitive } from "radix-ui";

// Custom: Wrapped with forwardRef and styling
const Accordion = AccordionPrimitive.Root; // Direct alias
const AccordionItem = React.forwardRef<...> // Enhanced with styling
```

#### 2. **Styling Enhancements**

**AccordionItem**:

- Added `border-b` class for visual separation
- Maintained all original props with `...props`

**AccordionTrigger**:

- Wrapped in `AccordionPrimitive.Header` with `flex` class
- Enhanced className with:
  - `flex flex-1 items-center justify-between` - Layout
  - `py-4` - Vertical padding
  - `text-sm font-medium` - Typography
  - `transition-all hover:underline` - Interactions
  - `text-left` - Text alignment
  - `[&[data-state=open]>svg]:rotate-180` - State-based icon rotation

**AccordionContent**:

- Added custom animations: `data-[state=closed]:animate-accordion-up data-[state=open]:animate-accordion-down`
- Wrapped content in div with `pb-4 pt-0` for consistent spacing
- `overflow-hidden` for clean animations

#### 3. **Icon Integration**

- Added `ChevronDown` from Lucide React
- Applied `h-4 w-4 shrink-0 text-muted-foreground transition-transform duration-200`
- Automatic rotation based on `data-state` attribute

#### 4. **Accessibility Improvements**

- Maintained all Radix UI accessibility features
- Added proper ARIA attributes through Radix primitives
- Enhanced visual feedback for keyboard navigation

### Template for Extending Other Radix UI Components

#### Step 1: Import and Setup

```tsx
"use client";

import * as React from "react";
import { ComponentName as ComponentPrimitive } from "radix-ui";
import { IconName } from "lucide-react";
import { cn } from "../../lib/utils";
```

#### Step 2: Create Enhanced Components

```tsx
// Root component (usually direct alias)
const ComponentName = ComponentPrimitive.Root;

// Enhanced sub-components
const ComponentSubComponent = React.forwardRef<
  React.ElementRef<typeof ComponentPrimitive.SubComponent>,
  React.ComponentPropsWithoutRef<typeof ComponentPrimitive.SubComponent>
>(({ className, children, ...props }, ref) => (
  <ComponentPrimitive.SubComponent
    ref={ref}
    className={cn("base-classes", className)}
    {...props}
  >
    {children}
    {/* Add icons, animations, or additional elements */}
  </ComponentPrimitive.SubComponent>
));
ComponentSubComponent.displayName = ComponentPrimitive.SubComponent.displayName;
```

#### Step 3: Apply Customizations

**Styling**:

- Use `cn()` utility for conditional classes
- Add Tailwind classes for layout, typography, spacing
- Include hover, focus, and state-based styles
- Add transition classes for smooth animations

**Icons**:

- Import from Lucide React
- Apply consistent sizing (`h-4 w-4`)
- Add state-based transformations
- Use `text-muted-foreground` for subtle appearance

**Animations**:

- Use `data-[state=...]` selectors for state-based animations
- Define custom animations in Tailwind config
- Apply `transition-*` classes for smooth changes

**Accessibility**:

- Preserve all original props with `...props`
- Maintain Radix UI's built-in accessibility features
- Add `displayName` for better debugging

#### Step 4: Export Pattern

```tsx
export { ComponentName, ComponentSubComponent1, ComponentSubComponent2 };
```

### Best Practices

1. **Consistency**: Follow the same pattern across all UI components
2. **Accessibility**: Never break Radix UI's accessibility features
3. **Styling**: Use design system tokens and consistent spacing
4. **Performance**: Use `forwardRef` for proper ref forwarding
5. **TypeScript**: Maintain full type safety with proper generics
6. **Documentation**: Add JSDoc comments for complex customizations

### Usage Example

```tsx
import {
  Accordion,
  AccordionItem,
  AccordionTrigger,
  AccordionContent,
} from "@/components/ui/accordion";

<Accordion type="single" collapsible>
  <AccordionItem value="item-1">
    <AccordionTrigger>Question?</AccordionTrigger>
    <AccordionContent>Answer content here.</AccordionContent>
  </AccordionItem>
</Accordion>;
```

This template ensures consistent, accessible, and well-styled Radix UI component extensions throughout the OpenCut project.
