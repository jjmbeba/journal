# Phase 2: Editor & UI Components

## Overview

Builds the writing interface and design system. This phase creates the "Editorial Sanctuary" aesthetic with a focus on distraction-free writing, elegant typography, and responsive layouts that work seamlessly across desktop and mobile.

**Status**: Not Started  
**Estimated Duration**: 2 weeks  
**Priority**: High  
**Blocked by**: Phase 1 (Foundation & Encryption)

---

## Key Features

### Design System: "Editorial Sanctuary"

A contemplative, distraction-free writing environment that feels like a premium notebook meets modern digital privacy.

**Design Philosophy**:
- Minimal visual noise
- Typography-driven hierarchy
- Generous whitespace
- Intentional, sparse motion
- Warm, paper-like color palette

### TipTap Editor

- Rich text editing with minimal UI
- Markdown shortcuts support
- Auto-save with visual feedback
- Word count display
- Focus mode (distraction-free)

### Responsive Layout

- **Desktop**: Sidebar navigation with entry list
- **Mobile**: Bottom navigation bar
- **Tablet**: Adaptive layout
- **Touch-optimized**: Large tap targets, swipe gestures

### Mood Tracking

- 5-point mood scale
- Visual mood selector
- Mood stored with each entry

---

## Dependencies

### New Dependencies

```bash
bun add @tiptap/react @tiptap/starter-kit @tiptap/extension-placeholder @tiptap/extension-typography
bun add date-fns
bun add framer-motion
bun add @fontsource/source-serif-4 @fontsource/inter
```

| Package | Purpose | Size |
|---------|---------|------|
| `@tiptap/react` | React integration for TipTap | ~45KB |
| `@tiptap/starter-kit` | Basic editor extensions | ~35KB |
| `@tiptap/extension-placeholder` | Placeholder text | ~2KB |
| `@tiptap/extension-typography` | Smart typography | ~3KB |
| `date-fns` | Date formatting | ~20KB (tree-shakeable) |
| `framer-motion` | UI animations | ~40KB |
| `@fontsource/source-serif-4` | Display font | ~80KB |
| `@fontsource/inter` | Body font | ~50KB |

**Total Bundle Impact**: ~275KB

### Existing Dependencies

- `tailwindcss` - Styling framework (v4 configured)
- `lucide-react` - Icon library
- `radix-ui` - Accessible UI primitives (via shadcn)
- `class-variance-authority` - Component variants

---

## Design System Specifications

### Typography

```css
/* Font Families */
--font-display: "Source Serif 4", Georgia, "Times New Roman", serif;
--font-body: "Inter", -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
--font-mono: "JetBrains Mono", "Fira Code", Consolas, Monaco, monospace;

/* Font Sizes */
--text-xs: 0.75rem;      /* 12px - Captions */
--text-sm: 0.875rem;     /* 14px - Small text */
--text-base: 1rem;       /* 16px - Body */
--text-lg: 1.125rem;     /* 18px - Lead */
--text-xl: 1.25rem;      /* 20px - H4 */
--text-2xl: 1.5rem;      /* 24px - H3 */
--text-3xl: 1.875rem;    /* 30px - H2 */
--text-4xl: 2.25rem;     /* 36px - H1 */

/* Line Heights */
--leading-tight: 1.25;
--leading-snug: 1.375;
--leading-normal: 1.5;
--leading-relaxed: 1.625;
--leading-loose: 2;

/* Font Weights */
--font-normal: 400;
--font-medium: 500;
--font-semibold: 600;
--font-bold: 700;
```

### Color Palette

#### Light Mode (Default)

```css
/* Backgrounds */
--color-paper: #FAF9F6;          /* Main background */
--color-surface: #FFFFFF;         /* Cards, panels */
--color-surface-hover: #F5F4F0;   /* Hover states */

/* Text */
--color-ink: #1a1a1a;            /* Primary text */
--color-ink-secondary: #4a4a4a;  /* Secondary text */
--color-ink-muted: #6b6b6b;      /* Muted text */
--color-ink-subtle: #9a9a9a;     /* Subtle text */

/* Accents */
--color-accent: #8B7355;          /* Warm brown */
--color-accent-hover: #6B5A45;    /* Darker brown */
--color-accent-light: #E8E4E0;    /* Light cream */

/* Semantic */
--color-success: #10B981;         /* Green */
--color-warning: #F59E0B;         /* Amber */
--color-error: #EF4444;           /* Red */
--color-info: #3B82F6;            /* Blue */

/* Borders */
--color-border: #E5E5E5;
--color-border-subtle: #F0F0F0;
```

#### Dark Mode

```css
--color-paper: #1a1a1a;
--color-surface: #2a2a2a;
--color-surface-hover: #3a3a3a;
--color-ink: #FAF9F6;
--color-ink-secondary: #C0C0C0;
--color-ink-muted: #808080;
--color-border: #404040;
```

#### OLED Mode (Optional)

```css
--color-paper: #000000;           /* True black */
--color-surface: #0a0a0a;
--color-border: #1a1a1a;
```

### Spacing Scale

```css
--space-0: 0;
--space-1: 0.25rem;    /* 4px */
--space-2: 0.5rem;     /* 8px */
--space-3: 0.75rem;    /* 12px */
--space-4: 1rem;       /* 16px */
--space-5: 1.25rem;    /* 20px */
--space-6: 1.5rem;     /* 24px */
--space-8: 2rem;       /* 32px */
--space-10: 2.5rem;    /* 40px */
--space-12: 3rem;      /* 48px */
--space-16: 4rem;      /* 64px */
--space-20: 5rem;      /* 80px */
--space-24: 6rem;      /* 96px */
```

### Layout Specifications

```css
/* Content Width */
--content-max-width: 680px;
--content-padding-x: 2rem;        /* Desktop */
--content-padding-x-mobile: 1rem; /* Mobile */

/* Sidebar */
--sidebar-width: 280px;
--sidebar-collapsed-width: 64px;

/* Mobile Nav */
--mobile-nav-height: 64px;

/* Border Radius */
--radius-sm: 0.25rem;
--radius-md: 0.375rem;
--radius-lg: 0.5rem;
--radius-xl: 0.75rem;
--radius-2xl: 1rem;
--radius-full: 9999px;

/* Shadows */
--shadow-sm: 0 1px 2px 0 rgb(0 0 0 / 0.05);
--shadow-md: 0 4px 6px -1px rgb(0 0 0 / 0.1), 0 2px 4px -2px rgb(0 0 0 / 0.1);
--shadow-lg: 0 10px 15px -3px rgb(0 0 0 / 0.1), 0 4px 6px -4px rgb(0 0 0 / 0.1);
```

### Motion Design

**Philosophy**: Sparse, purposeful, high-impact

```css
/* Durations */
--duration-fast: 150ms;
--duration-normal: 250ms;
--duration-slow: 350ms;

/* Easing */
--ease-out: cubic-bezier(0.16, 1, 0.3, 1);
--ease-in-out: cubic-bezier(0.65, 0, 0.35, 1);
--ease-spring: cubic-bezier(0.34, 1.56, 0.64, 1);
```

**Animation Guidelines**:
- Use motion to guide attention, not decorate
- Prefer `transform` and `opacity` for performance
- Respect `prefers-reduced-motion`
- Maximum 3 simultaneous animations

---

## Component Architecture

### Layout Components

```typescript
// AppShell - Main application wrapper
interface AppShellProps {
  children: React.ReactNode;
  sidebar?: React.ReactNode;
  bottomNav?: React.ReactNode;
}

// DesktopNav - Sidebar navigation (desktop only)
interface DesktopNavProps {
  entries: JournalEntry[];
  activeEntryId?: string;
  onEntrySelect: (id: string) => void;
  onNewEntry: () => void;
}

// MobileNav - Bottom navigation (mobile only)
interface MobileNavProps {
  activeRoute: string;
  onNavigate: (route: string) => void;
}

// FocusMode - Distraction-free wrapper
interface FocusModeProps {
  children: React.ReactNode;
  isActive: boolean;
  onExit: () => void;
}
```

### Editor Components

```typescript
// Editor - Main writing interface
interface EditorProps {
  initialContent?: string;
  onChange: (content: string) => void;
  onSave: () => void;
  placeholder?: string;
  autoFocus?: boolean;
}

// Toolbar - Floating formatting toolbar
interface ToolbarProps {
  editor: Editor | null;
  isVisible: boolean;
}

// MoodSelector - Mood picker
interface MoodSelectorProps {
  value?: number;
  onChange: (mood: number) => void;
  size?: 'sm' | 'md' | 'lg';
}

// WordCount - Live word/character count
interface WordCountProps {
  content: string;
  goal?: number;
}

// AutoSaveIndicator - Save status display
interface AutoSaveIndicatorProps {
  status: 'saved' | 'saving' | 'error' | 'unsaved';
  lastSaved?: Date;
}
```

### Entry Components

```typescript
// EntryList - List of journal entries
interface EntryListProps {
  entries: JournalEntry[];
  selectedId?: string;
  onSelect: (id: string) => void;
  onDelete: (id: string) => void;
}

// EntryCard - Individual entry preview
interface EntryCardProps {
  entry: JournalEntry;
  isSelected: boolean;
  onClick: () => void;
  onDelete: () => void;
}

// EntryDetail - Full entry view
interface EntryDetailProps {
  entry: JournalEntry;
  onEdit: () => void;
  onDelete: () => void;
}

// Timeline - Calendar/timeline view
interface TimelineProps {
  entries: JournalEntry[];
  selectedDate?: Date;
  onDateSelect: (date: Date) => void;
}
```

### UI Primitives

```typescript
// Button - Action button
interface ButtonProps {
  variant?: 'primary' | 'secondary' | 'ghost' | 'danger';
  size?: 'sm' | 'md' | 'lg';
  isLoading?: boolean;
  isDisabled?: boolean;
  children: React.ReactNode;
  onClick?: () => void;
}

// IconButton - Icon-only button
interface IconButtonProps {
  icon: LucideIcon;
  label: string;
  size?: 'sm' | 'md' | 'lg';
  variant?: 'default' | 'ghost';
  onClick?: () => void;
}

// Card - Content container
interface CardProps {
  children: React.ReactNode;
  isInteractive?: boolean;
  isSelected?: boolean;
  onClick?: () => void;
}

// Tooltip - Helper text on hover
interface TooltipProps {
  content: string;
  children: React.ReactNode;
  position?: 'top' | 'bottom' | 'left' | 'right';
}

// Modal - Dialog overlay
interface ModalProps {
  isOpen: boolean;
  onClose: () => void;
  title: string;
  children: React.ReactNode;
  actions?: React.ReactNode;
}
```

---

## Success Metrics

### Performance Metrics

| Metric | Target | Tool |
|--------|--------|------|
| First Contentful Paint | <1.5s | Lighthouse |
| Largest Contentful Paint | <2.5s | Lighthouse |
| Time to Interactive | <3s | Lighthouse |
| Cumulative Layout Shift | <0.1 | Lighthouse |
| First Input Delay | <100ms | Chrome UX Report |
| Editor load time | <500ms | Custom timing |
| Bundle size (initial) | <500KB | Webpack Bundle Analyzer |

### Quality Metrics

| Metric | Target | Method |
|--------|--------|--------|
| Lighthouse Performance | >90 | Automated audit |
| Lighthouse Accessibility | 100 | axe-core |
| Lighthouse Best Practices | 100 | Automated audit |
| Lighthouse SEO | 100 | Automated audit |
| Test coverage | >80% | Jest/Vitest |
| TypeScript strict mode | 0 errors | tsc |

### UX Metrics

| Metric | Target | Method |
|--------|--------|--------|
| Mobile viewport coverage | 100% | Visual regression |
| Touch target size | >44x44px | Manual audit |
| Keyboard navigation | 100% functional | Manual test |
| Screen reader support | Full | NVDA/VoiceOver |
| Animation smoothness | 60fps | Chrome DevTools |

---

## Test Requirements

### Component Tests

```typescript
// tests/components/Editor.test.tsx
describe('Editor', () => {
  it('renders with placeholder text', () => {
    // Verify placeholder visible
  });
  
  it('accepts and displays text input', () => {
    // Simulate typing
    // Verify content updated
  });
  
  it('triggers onChange with HTML content', () => {
    // Type text
    // Verify onChange called with HTML
  });
  
  it('formats text with toolbar buttons', () => {
    // Select text
    // Click bold button
    // Verify bold formatting applied
  });
  
  it('supports markdown shortcuts', () => {
    // Type **bold**
    // Verify converted to bold
  });
  
  it('displays word count', () => {
    // Type 10 words
    // Verify count shows 10
  });
  
  it('enters focus mode on keyboard shortcut', () => {
    // Press Cmd/Ctrl+Shift+F
    // Verify focus mode active
  });
  
  it('auto-saves after debounce period', async () => {
    // Type content
    // Wait for debounce
    // Verify onSave called
  });
  
  it('shows save indicator', () => {
    // Verify indicator visible during save
  });
  
  it('is accessible via keyboard', () => {
    // Tab navigation
    // Verify focus states
  });
});

// tests/components/MoodSelector.test.tsx
describe('MoodSelector', () => {
  it('displays 5 mood options', () => {
    // Verify 5 buttons rendered
  });
  
  it('selects mood on click', () => {
    // Click mood 4
    // Verify onChange called with 4
  });
  
  it('shows selected state', () => {
    // Pass value prop
    // Verify selected mood highlighted
  });
  
  it('is accessible via keyboard', () => {
    // Tab through options
    // Enter to select
  });
  
  it('has proper ARIA labels', () => {
    // Verify screen reader text
  });
});

// tests/components/EntryList.test.tsx
describe('EntryList', () => {
  it('renders list of entries', () => {
    // Pass 5 entries
    // Verify 5 cards rendered
  });
  
  it('highlights selected entry', () => {
    // Pass selectedId
    // Verify correct entry highlighted
  });
  
  it('calls onSelect when entry clicked', () => {
    // Click entry
    // Verify onSelect called with id
  });
  
  it('shows entry preview', () => {
    // Verify title and snippet visible
  });
  
  it('displays formatted date', () => {
    // Verify relative date ("2 hours ago")
  });
  
  it('handles empty state', () => {
    // Pass empty array
    // Verify empty message shown
  });
});
```

### Integration Tests

```typescript
// tests/integration/editor-flow.test.tsx
describe('Editor Flow', () => {
  it('creates new entry from editor', async () => {
    // Open editor
    // Type content
    // Select mood
    // Save
    // Verify entry created in list
  });
  
  it('edits existing entry', async () => {
    // Select entry from list
    // Click edit
    // Modify content
    // Save
    // Verify entry updated
  });
  
  it('maintains scroll position in list', () => {
    // Scroll entry list
    // Select entry
    // Go back
    // Verify scroll position restored
  });
  
  it('auto-saves draft on navigation', async () => {
    // Type in editor
    // Navigate away without saving
    // Verify draft saved
  });
});

// tests/integration/responsive.test.tsx
describe('Responsive Layout', () => {
  it('shows sidebar on desktop', () => {
    // Render at 1200px width
    // Verify sidebar visible
    // Verify bottom nav hidden
  });
  
  it('shows bottom nav on mobile', () => {
    // Render at 375px width
    // Verify bottom nav visible
    // Verify sidebar hidden
  });
  
  it('adapts content width', () => {
    // Desktop: max-width 680px
    // Mobile: full width with padding
  });
  
  it('adjusts touch targets on mobile', () => {
    // Verify minimum 44x44px
  });
});
```

### Accessibility Tests

```typescript
// tests/accessibility/components.test.tsx
describe('Accessibility', () => {
  it('Editor has no axe violations', async () => {
    const { container } = render(<Editor />);
    const results = await axe(container);
    expect(results).toHaveNoViolations();
  });
  
  it('supports keyboard navigation throughout', () => {
    // Tab through all interactive elements
    // Verify logical tab order
  });
  
  it('has proper heading hierarchy', () => {
    // Verify h1, h2, h3 order
  });
  
  it('has sufficient color contrast', () => {
    // All text meets WCAG AA (4.5:1)
  });
  
  it('announces dynamic content', () => {
    // Verify ARIA live regions
  });
  
  it('respects prefers-reduced-motion', () => {
    // Disable animations when set
  });
});
```

### Visual Regression Tests

```typescript
// tests/visual/editor.test.tsx
describe('Editor Visual', () => {
  it('matches snapshot - empty', () => {
    // Empty editor state
  });
  
  it('matches snapshot - with content', () => {
    // Editor with text
  });
  
  it('matches snapshot - focus mode', () => {
    // Focus mode active
  });
  
  it('matches snapshot - dark mode', () => {
    // Dark theme
  });
});
```

---

## Implementation Checklist

### Design System

- [ ] CSS custom properties (design tokens)
  - [ ] Colors (light/dark/oled modes)
  - [ ] Typography scale
  - [ ] Spacing scale
  - [ ] Shadows and borders
  - [ ] Animation timings
  
- [ ] Global styles
  - [ ] CSS reset (Tailwind preflight)
  - [ ] Base element styles
  - [ ] Font imports
  - [ ] Theme switching mechanism

- [ ] Tailwind configuration
  - [ ] Custom color palette
  - [ ] Custom font families
  - [ ] Custom spacing
  - [ ] Custom animations

### Core Components

- [ ] Layout components
  - [ ] `AppShell` - Main wrapper with responsive behavior
  - [ ] `DesktopNav` - Sidebar with entry list
  - [ ] `MobileNav` - Bottom navigation bar
  - [ ] `FocusMode` - Distraction-free container
  
- [ ] Editor components
  - [ ] `Editor` - TipTap integration
  - [ ] `Toolbar` - Floating formatting bar
  - [ ] `MoodSelector` - 5-point mood picker
  - [ ] `WordCount` - Live statistics
  - [ ] `AutoSaveIndicator` - Save status
  
- [ ] Entry components
  - [ ] `EntryList` - Virtualized list for performance
  - [ ] `EntryCard` - Entry preview card
  - [ ] `EntryDetail` - Full entry view
  - [ ] `Timeline` - Calendar navigation
  - [ ] `EmptyState` - No entries placeholder

- [ ] UI primitives
  - [ ] `Button` - Action buttons
  - [ ] `IconButton` - Icon-only buttons
  - [ ] `Card` - Content containers
  - [ ] `Tooltip` - Helper text
  - [ ] `Modal` - Dialog overlays
  - [ ] `Skeleton` - Loading states
  - [ ] `Toast` - Notifications

### Editor Implementation

- [ ] TipTap setup
  - [ ] Basic editor configuration
  - [ ] Custom extensions
  - [ ] Placeholder extension
  - [ ] Typography extension
  - [ ] Markdown shortcuts
  
- [ ] Toolbar functionality
  - [ ] Bold, italic, underline
  - [ ] Headers (H1, H2, H3)
  - [ ] Lists (ordered, unordered)
  - [ ] Quotes
  - [ ] Code blocks
  - [ ] Links
  
- [ ] Editor UX
  - [ ] Auto-save with debounce
  - [ ] Word/character count
  - [ ] Focus mode toggle
  - [ ] Placeholder text
  - [ ] Keyboard shortcuts

### Responsive Design

- [ ] Breakpoints
  - [ ] Mobile: < 640px
  - [ ] Tablet: 640px - 1024px
  - [ ] Desktop: > 1024px
  
- [ ] Mobile optimizations
  - [ ] Bottom navigation
  - [ ] Touch-friendly targets (44x44px)
  - [ ] Swipe gestures
  - [ ] Virtual keyboard handling
  
- [ ] Desktop features
  - [ ] Sidebar with entry list
  - [ ] Keyboard shortcuts
  - [ ] Hover states
  - [ ] Split view (optional)

### State Management

- [ ] Editor state
  - [ ] `useEditor` hook
  - [ ] Content management
  - [ ] Formatting state
  - [ ] Selection tracking
  
- [ ] Entry state
  - [ ] `useEntries` hook
  - [ ] List management
  - [ ] Selection state
  - [ ] CRUD operations
  
- [ ] UI state
  - [ ] `useFocusMode` hook
  - [ ] `useTheme` hook
  - [ ] Modal/sidebar state
  - [ ] Toast notifications

### Animation & Motion

- [ ] Page transitions
- [ ] Entry list animations
- [ ] Modal/dialog animations
- [ ] Save indicator animation
- [ ] Focus mode transition
- [ ] Reduced motion support

### Testing

- [ ] Unit tests for all components
- [ ] Integration tests for flows
- [ ] Accessibility audit
- [ ] Visual regression tests
- [ ] Responsive design tests
- [ ] Performance benchmarks

---

## Deliverables

### Code Files

| File | Purpose |
|------|---------|
| `src/styles/design-tokens.css` | CSS custom properties |
| `src/styles/global.css` | Global styles and resets |
| `src/components/layout/AppShell.tsx` | Main layout wrapper |
| `src/components/layout/DesktopNav.tsx` | Desktop sidebar |
| `src/components/layout/MobileNav.tsx` | Mobile bottom nav |
| `src/components/layout/FocusMode.tsx` | Focus mode container |
| `src/components/editor/Editor.tsx` | TipTap editor |
| `src/components/editor/Toolbar.tsx` | Formatting toolbar |
| `src/components/editor/MoodSelector.tsx` | Mood picker |
| `src/components/editor/WordCount.tsx` | Statistics display |
| `src/components/editor/AutoSaveIndicator.tsx` | Save status |
| `src/components/entries/EntryList.tsx` | Entry list |
| `src/components/entries/EntryCard.tsx` | Entry preview |
| `src/components/entries/EntryDetail.tsx` | Full entry view |
| `src/components/entries/Timeline.tsx` | Calendar view |
| `src/components/ui/Button.tsx` | Button component |
| `src/components/ui/IconButton.tsx` | Icon button |
| `src/components/ui/Card.tsx` | Card container |
| `src/components/ui/Tooltip.tsx` | Tooltip |
| `src/components/ui/Modal.tsx` | Modal dialog |
| `src/components/ui/Skeleton.tsx` | Loading skeleton |
| `src/components/ui/Toast.tsx` | Toast notification |
| `src/hooks/useEditor.ts` | Editor state management |
| `src/hooks/useAutoSave.ts` | Auto-save logic |
| `src/hooks/useFocusMode.ts` | Focus mode state |
| `src/hooks/useTheme.ts` | Theme switching |
| `src/contexts/ThemeContext.tsx` | Global theme state |
| `src/lib/editor.ts` | TipTap configuration |

### Route Files

| File | Route | Description |
|------|-------|-------------|
| `src/routes/index.tsx` | `/` | Dashboard/home |
| `src/routes/write.tsx` | `/write` | New entry |
| `src/routes/entry/$id.tsx` | `/entry/:id` | View entry |
| `src/routes/entry/$id.edit.tsx` | `/entry/:id/edit` | Edit entry |
| `src/routes/timeline.tsx` | `/timeline` | Timeline view |

### Test Files

| File | Coverage |
|------|----------|
| `tests/components/Editor.test.tsx` | Editor functionality |
| `tests/components/MoodSelector.test.tsx` | Mood picker |
| `tests/components/EntryList.test.tsx` | Entry list |
| `tests/components/layout/*.test.tsx` | Layout components |
| `tests/components/ui/*.test.tsx` | UI primitives |
| `tests/integration/editor-flow.test.tsx` | Editor integration |
| `tests/integration/responsive.test.tsx` | Responsive design |
| `tests/accessibility/components.test.tsx` | Accessibility |
| `tests/visual/*.test.tsx` | Visual regression |

---

## File Structure

```
src/
├── styles/
│   ├── design-tokens.css
│   └── global.css
├── components/
│   ├── layout/
│   │   ├── AppShell.tsx
│   │   ├── DesktopNav.tsx
│   │   ├── MobileNav.tsx
│   │   └── FocusMode.tsx
│   ├── editor/
│   │   ├── Editor.tsx
│   │   ├── Toolbar.tsx
│   │   ├── MoodSelector.tsx
│   │   ├── WordCount.tsx
│   │   └── AutoSaveIndicator.tsx
│   ├── entries/
│   │   ├── EntryList.tsx
│   │   ├── EntryCard.tsx
│   │   ├── EntryDetail.tsx
│   │   ├── Timeline.tsx
│   │   └── EmptyState.tsx
│   └── ui/
│       ├── Button.tsx
│       ├── IconButton.tsx
│       ├── Card.tsx
│       ├── Tooltip.tsx
│       ├── Modal.tsx
│       ├── Skeleton.tsx
│       └── Toast.tsx
├── hooks/
│   ├── useEditor.ts
│   ├── useAutoSave.ts
│   ├── useFocusMode.ts
│   └── useTheme.ts
├── contexts/
│   └── ThemeContext.tsx
├── lib/
│   └── editor.ts
└── routes/
    ├── index.tsx
    ├── write.tsx
    ├── entry/
    │   ├── $id.tsx
    │   └── $id.edit.tsx
    └── timeline.tsx
```

---

## Dependencies on Phase 1

| Dependency | From Phase 1 | Usage |
|------------|--------------|-------|
| Encryption context | Phase 1 | Encrypt entry content before save |
| Entry API | Phase 1 | Fetch and save entries |
| Master password | Phase 1 | Required to decrypt entries for display |
| Database schema | Phase 1 | Entry and settings tables |

---

## Dependencies for Phase 3

| Deliverable | Used in Phase 3 |
|-------------|-----------------|
| Editor component | PWA offline editing |
| Entry list | Offline entry browsing |
| Theme system | PWA theming |
| Auto-save | Background sync integration |

---

## Design Resources

### Font Choices

- **Display**: Source Serif 4 - Elegant, readable, literary feel
- **Body**: Inter - Clean, modern, excellent legibility
- **Monospace**: JetBrains Mono - Developer-friendly, distinct characters

### Color Inspiration

- **Warm Paper**: Leuchtturm1917 notebook cream
- **Ink**: Premium fountain pen black
- **Accent**: Leather journal brown

### References

- [iA Writer](https://ia.net/writer) - Minimalist writing app
- [Bear App](https://bear.app) - Elegant note-taking
- [Notion](https://notion.so) - Clean component design
- [Linear](https://linear.app) - Modern UI patterns

---

## Notes

### Browser Support

- **Minimum**: Chrome 80+, Firefox 75+, Safari 13+, Edge 80+
- **Required**: CSS Variables, Flexbox, Grid, ES2020
- **Progressive Enhancement**: Advanced animations, backdrop-filter

### Performance Targets

- Bundle size: <500KB initial JS
- Editor bundle: Loaded on demand (code splitting)
- Fonts: Preload critical fonts, lazy load others
- Images: None (text-only app), icons as SVG

### Known Challenges

- TipTap mobile performance on large documents
- Virtual keyboard pushing content on mobile
- Touch gesture conflicts with editor selection
- Theme switching without flash

### Future Enhancements

- Split view (editor + preview side by side)
- Collaborative cursors (if collaboration added)
- Custom themes beyond preset options
- Plugin system for editor extensions
