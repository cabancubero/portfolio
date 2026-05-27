# Coding Portfolio Website — Scoping Document

**Document Version:** 1.0  
**Last Updated:** January 2026  
**Project Status:** Planning Phase

---

## Executive Summary

A dynamic, interactive coding portfolio website designed to showcase 6-8 projects with runnable code snippets, blog content, documentation links, and personal branding. The site features a unique **dual-mode viewing experience** that toggles between "Hiring Manager" and "Learner" perspectives, highlighting different aspects of your work based on the viewer's intent.

**Key Differentiators:**
- Dual-mode toggle (Portfolio Viewer vs. Learner Mode)
- In-browser code execution for JavaScript/TypeScript, Python, and SQL
- Playful micro-interactions and animations
- Blue and orange color palette with dark mode support
- Svelte and React component compatibility

---

## Table of Contents

1. [Site Architecture](#site-architecture)
2. [Dual-Mode System](#dual-mode-system)
3. [Page Specifications](#page-specifications)
4. [Interactive Code Execution](#interactive-code-execution)
5. [Design System](#design-system)
6. [Technical Stack](#technical-stack)
7. [Data Models](#data-models)
8. [Animations & Micro-interactions](#animations--micro-interactions)
9. [Deployment & Hosting](#deployment--hosting)
10. [Development Phases](#development-phases)
11. [Future Considerations](#future-considerations)

---

## Site Architecture

```
/
├── Homepage (/)
│   ├── Hero Section
│   ├── Mode Toggle (persistent across site)
│   ├── Featured Projects (3-4)
│   ├── Recent Blog Posts (2-3)
│   └── Quick Links
│
├── Projects (/projects)
│   ├── Filter Bar (Skills / Platforms / Purposes)
│   ├── Project Grid
│   └── Individual Project (/projects/[slug])
│       ├── Project Overview
│       ├── Code Playground
│       ├── GitHub Link
│       └── Related Projects
│
├── Blog (/blog)
│   ├── Post Listing
│   ├── Tag Filtering
│   └── Individual Post (/blog/[slug])
│       ├── Rich Content (MDX)
│       ├── Embedded Code Demos
│       └── Related Posts
│
├── Documentation (/docs)
│   └── Open Source Contributions List
│
└── About (/about)
    ├── Bio
    ├── Skills Visualization
    ├── Timeline
    └── Email Contact Link
```

---

## Dual-Mode System

### Concept

A persistent toggle in the navigation allows visitors to switch between two viewing modes. This creates a personalized experience that surfaces different information based on intent.

### Mode Definitions

| Aspect | Portfolio Mode (Hiring) | Learner Mode |
|--------|-------------------------|--------------|
| **Icon** | 💼 Briefcase | 🎓 Graduation Cap |
| **Tagline** | "See what I can build" | "Learn how I built it" |
| **Project Cards** | Emphasize outcomes, impact, tech stack | Emphasize learning journey, challenges, lessons |
| **Code Snippets** | Collapsed by default, focus on results | Expanded, with detailed comments |
| **Blog Excerpts** | Professional summaries | "What I learned" callouts |
| **About Page** | Professional bio, experience | Learning path, curiosity, growth mindset |
| **Animations** | Subtle, professional | More playful, exploratory |

### Implementation Details

```typescript
// Mode context structure
type ViewMode = 'portfolio' | 'learner';

interface ModeContext {
  mode: ViewMode;
  setMode: (mode: ViewMode) => void;
  // Content variants
  getProjectDescription: (project: Project) => string;
  getCodeComments: (snippet: CodeSnippet) => string;
  getAboutContent: () => AboutContent;
}
```

### Toggle Behavior

- Persists across page navigation (stored in localStorage)
- Smooth transition animation when switching
- First-time visitors see a brief tooltip explaining the modes
- Toggle lives in the navigation bar, always accessible
- URL parameter option for shareable links: `?mode=learner`

### Content Variants by Mode

**Project Descriptions:**
```
Portfolio Mode:
"Built a sentiment analysis pipeline processing 10K+ reviews daily, 
achieving 94% accuracy and reducing manual review time by 60%."

Learner Mode:
"I was curious how companies know what customers think at scale. 
This project taught me NLP fundamentals, the trade-offs between 
accuracy and speed, and how to structure data pipelines."
```

**Code Snippet Display:**
```
Portfolio Mode:
- Clean, production-ready code
- Minimal comments
- Focus on the solution

Learner Mode:
- Step-by-step comments explaining "why"
- Common mistakes highlighted
- Alternative approaches discussed
- "Try changing this value!" prompts
```

---

## Page Specifications

### 1. Homepage

**URL:** `/`

**Purpose:** Create intrigue, showcase best work, guide visitors to explore

**Sections:**

#### Hero Section
- Animated greeting with your name
- Dynamic subtitle that types out based on mode:
  - Portfolio: "Full-Stack Developer • Problem Solver • Builder"
  - Learner: "Curious Coder • Lifelong Learner • Knowledge Sharer"
- Mode toggle prominently featured with brief explanation
- Subtle animated background (gradient mesh or particle effect)

#### Featured Projects Grid
- 3-4 hand-picked projects
- Card hover effects: lift, glow, reveal additional info
- Mode-aware descriptions
- Quick tech stack badges
- "View Project" CTA with arrow animation

#### Recent Blog Posts
- 2-3 latest posts
- Reading time estimate
- Tag pills
- Hover effect: title underline animation

#### Footer CTA
- "Want to see more?" linking to Projects
- "Let's connect" with email link

**Interactions:**
- Scroll-triggered animations for each section
- Parallax effect on hero background
- Cards animate in with staggered delay

---

### 2. Projects Page

**URL:** `/projects`

**Purpose:** Comprehensive, filterable showcase of all work

#### Filter System

**Filter Categories:**

| Category | Options (Examples) |
|----------|-------------------|
| Skills | JavaScript, TypeScript, Python, SQL, Algorithmic Design |
| Platforms | AppsScript, GCP, React, Svelte, Node.js |
| Purposes | Field Planning, Sentiment Analysis, Data Visualization, Automation |

**Filter UX:**
- Horizontal pill-style buttons
- Multi-select within categories (OR logic)
- Cross-category filtering (AND logic)
- Active filters shown as dismissible chips
- "Clear All" button when filters active
- URL updates with filter state for shareability
- Animated filter transitions (cards shuffle/reorganize)

**Filter URL Structure:**
```
/projects?skills=javascript,python&platforms=gcp&purposes=automation
```

#### Project Cards

**Card Content:**
- Thumbnail/screenshot
- Title
- Mode-aware description (2-3 sentences)
- Tech stack badges (max 5 shown, "+N more" if exceeded)
- Date/timeframe
- Status badge (Completed, In Progress, Maintained)

**Card Interactions:**
- Hover: lift effect, border glow (blue/orange based on theme)
- Tech badges: tooltip with proficiency level on hover
- Entire card clickable

---

### 3. Individual Project Page

**URL:** `/projects/[slug]`

**Purpose:** Deep dive with runnable code

#### Header Section
- Project title with animated underline
- Status badge
- Date range
- Tech stack as interactive badges
- GitHub repo link (animated icon)
- Live demo link (if applicable)

#### Content Sections (Mode-Aware)

**Overview Tab:**
- Problem statement
- Solution approach
- Key outcomes/metrics (Portfolio Mode emphasizes)
- Learning journey (Learner Mode emphasizes)
- Screenshots/GIFs in lightbox gallery

**Code Playground Tab:**
- Multiple runnable code snippets
- Language tabs (JS/TS, Python, SQL)
- Each snippet has:
  - Description/context
  - Syntax-highlighted editor (read-only or editable)
  - Run button with loading state
  - Output panel
  - Reset button
  - "Open in CodeSandbox/Replit" for complex examples
- Learner Mode: additional "Explain This" expandable sections

**Architecture Tab (Optional):**
- System diagram
- Data flow visualization
- Component breakdown

#### Related Projects
- 2-3 related projects based on shared tags
- Horizontal scroll or grid

#### Navigation
- Previous/Next project arrows
- Back to all projects

---

### 4. Blog Page

**URL:** `/blog`

**Purpose:** Share knowledge, demonstrate communication skills

#### Listing Page

**Post Card Content:**
- Title
- Excerpt (mode-aware)
- Publication date
- Reading time
- Tags as pills
- Thumbnail (optional)

**Filtering:**
- Tag-based filtering
- Search by title/content
- Sort: newest, oldest, popular

**Layout:**
- List view (default) or grid view toggle
- Pagination or infinite scroll

#### Individual Blog Post

**URL:** `/blog/[slug]`

**Content Features:**
- MDX support for rich content
- Syntax-highlighted code blocks
- Embedded runnable code snippets (same component as projects)
- Images with captions
- Callout boxes (tip, warning, info)
- Table of contents (auto-generated for long posts)

**Post Footer:**
- Tags
- Share buttons (copy link, Twitter, LinkedIn)
- Related posts
- "Back to Blog" link

---

### 5. Documentation Page

**URL:** `/docs`

**Purpose:** Showcase open source contributions

#### Content Structure

**For Each Contribution:**
- Project name and logo
- Your role (Maintainer, Core Contributor, Contributor)
- Contribution description
- Link to documentation
- Link to specific PRs/commits (optional)
- Tech stack used

**Layout:**
- Card grid or detailed list
- Filter by contribution type
- Sort by date or project name

**Enhancement Ideas:**
- GitHub API integration to pull live stats
- Contribution activity graph

---

### 6. About Page

**URL:** `/about`

**Purpose:** Personal branding and contact

#### Sections

**Hero:**
- Professional photo with subtle animation (parallax or Ken Burns)
- Name and title
- Mode-aware tagline

**Bio Section:**
- Mode-aware content:
  - Portfolio: Professional background, expertise, value proposition
  - Learner: Learning journey, what excites you, teaching philosophy

**Skills Visualization:**
- Interactive skill chart or grid
- Categories: Languages, Frameworks, Tools, Concepts
- Proficiency indicators
- Hover for details

**Timeline:**
- Career/learning milestones
- Animated reveal on scroll
- Mode-aware framing

**Contact:**
- Email link (mailto: with subject line pre-filled)
- Social links with animated icons:
  - GitHub
  - LinkedIn
  - Twitter/X (optional)
- No contact form per requirements

---

## Interactive Code Execution

### Supported Languages

| Language | Execution Method | Library/Service |
|----------|------------------|-----------------|
| JavaScript/TypeScript | In-browser | Native + esbuild for TS |
| Python | WebAssembly | Pyodide |
| SQL | In-browser SQLite | sql.js |

### Code Playground Component

```
┌─────────────────────────────────────────────────────────────┐
│  ┌─────────┐ ┌─────────┐ ┌─────────┐                       │
│  │   JS    │ │ Python  │ │   SQL   │  (language tabs)      │
│  └─────────┘ └─────────┘ └─────────┘                       │
├─────────────────────────────────────────────────────────────┤
│  // Calculate fibonacci sequence                            │
│  function fib(n) {                                          │
│    if (n <= 1) return n;                                    │
│    return fib(n - 1) + fib(n - 2);                         │
│  }                                                          │
│                                                             │
│  console.log(fib(10));                                      │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│  [▶ Run]  [↺ Reset]  [📋 Copy]  [🔗 Open in Sandbox]       │
├─────────────────────────────────────────────────────────────┤
│  Output:                                                    │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  55                                                  │   │
│  │  ✓ Executed in 2ms                                  │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### Execution Details

**JavaScript/TypeScript:**
- Sandboxed execution using iframe or Web Worker
- Console output capture
- Error handling with friendly messages
- TypeScript compiled on-the-fly with esbuild-wasm

**Python:**
- Pyodide loaded on demand (lazy loading)
- Pre-installed packages: numpy, pandas (as needed)
- stdout/stderr capture
- Loading indicator while Pyodide initializes

**SQL:**
- sql.js with pre-loaded sample databases
- Multiple sample datasets:
  - `users` table
  - `orders` table
  - `products` table
- Query result displayed as formatted table
- Execution time shown

### Learner Mode Enhancements

- "Explain This Code" button with AI-generated or pre-written explanations
- "Try This Challenge" prompts
- Hints system
- Step-through execution visualization (future)

### Svelte & React Compatibility

**Approach:** Use Next.js as the primary framework with strategic Svelte integration.

**Options:**

1. **Svelte Components via Astro Islands** (if switching to Astro)
2. **Svelte Widgets as Standalone Bundles** embedded in Next.js
3. **Showcase Svelte in Code Playground** (display and run Svelte examples via Svelte REPL embed)

**Recommended Approach:**
- Primary site built in Next.js (React)
- Svelte projects showcased via:
  - Embedded Svelte REPL for interactive demos
  - Links to Svelte-specific codebases
  - Screenshots and GIFs of Svelte apps
- This demonstrates proficiency in both without over-complicating the build

---

## Design System

### Color Palette

**Primary Colors:**

| Name | Hex | Usage |
|------|-----|-------|
| Ocean Blue | `#2563EB` | Primary actions, links, accents |
| Deep Blue | `#1E40AF` | Hover states, depth |
| Sunset Orange | `#F97316` | Secondary accent, CTAs, highlights |
| Warm Orange | `#EA580C` | Hover states |

**Neutral Palette:**

| Name | Light Mode | Dark Mode | Usage |
|------|------------|-----------|-------|
| Background | `#FAFAFA` | `#0F172A` | Page background |
| Surface | `#FFFFFF` | `#1E293B` | Cards, panels |
| Border | `#E2E8F0` | `#334155` | Dividers, borders |
| Text Primary | `#0F172A` | `#F8FAFC` | Headings, body |
| Text Secondary | `#64748B` | `#94A3B8` | Captions, meta |

**Semantic Colors:**

| Name | Hex | Usage |
|------|-----|-------|
| Success | `#10B981` | Success states, online |
| Warning | `#F59E0B` | Warnings, pending |
| Error | `#EF4444` | Errors, destructive |
| Info | `#3B82F6` | Informational |

### Typography

**Font Selection:**

| Role | Font | Fallback | Weight |
|------|------|----------|--------|
| Display/Headings | **Plus Jakarta Sans** | system-ui, sans-serif | 600, 700 |
| Body | **Plus Jakarta Sans** | system-ui, sans-serif | 400, 500 |
| Code | **JetBrains Mono** | Consolas, monospace | 400, 500 |

**Why Plus Jakarta Sans:**
- Modern, geometric sans-serif
- Excellent readability at all sizes
- Distinctive but not distracting
- Variable font for performance
- Free (Google Fonts)

**Type Scale:**

| Name | Size | Line Height | Usage |
|------|------|-------------|-------|
| Display | 3.5rem (56px) | 1.1 | Hero headings |
| H1 | 2.5rem (40px) | 1.2 | Page titles |
| H2 | 2rem (32px) | 1.25 | Section headings |
| H3 | 1.5rem (24px) | 1.3 | Card titles |
| H4 | 1.25rem (20px) | 1.4 | Subsections |
| Body | 1rem (16px) | 1.6 | Paragraphs |
| Small | 0.875rem (14px) | 1.5 | Captions, meta |
| XSmall | 0.75rem (12px) | 1.4 | Badges, labels |

### Spacing System

Using 4px base unit:

| Name | Value | Usage |
|------|-------|-------|
| xs | 4px | Tight spacing |
| sm | 8px | Small gaps |
| md | 16px | Default gaps |
| lg | 24px | Section padding |
| xl | 32px | Large sections |
| 2xl | 48px | Page sections |
| 3xl | 64px | Major breaks |

### Border Radius

| Name | Value | Usage |
|------|-------|-------|
| sm | 4px | Badges, small elements |
| md | 8px | Buttons, inputs |
| lg | 12px | Cards |
| xl | 16px | Large cards, modals |
| full | 9999px | Pills, avatars |

### Shadows

**Light Mode:**
```css
--shadow-sm: 0 1px 2px rgba(0, 0, 0, 0.05);
--shadow-md: 0 4px 6px -1px rgba(0, 0, 0, 0.1);
--shadow-lg: 0 10px 15px -3px rgba(0, 0, 0, 0.1);
--shadow-glow-blue: 0 0 20px rgba(37, 99, 235, 0.3);
--shadow-glow-orange: 0 0 20px rgba(249, 115, 22, 0.3);
```

**Dark Mode:**
```css
--shadow-sm: 0 1px 2px rgba(0, 0, 0, 0.3);
--shadow-md: 0 4px 6px -1px rgba(0, 0, 0, 0.4);
--shadow-lg: 0 10px 15px -3px rgba(0, 0, 0, 0.5);
--shadow-glow-blue: 0 0 30px rgba(37, 99, 235, 0.4);
--shadow-glow-orange: 0 0 30px rgba(249, 115, 22, 0.4);
```

### Dark Mode Implementation

**Strategy:** CSS custom properties with class-based toggle

```css
:root {
  --color-bg: #FAFAFA;
  --color-surface: #FFFFFF;
  --color-text: #0F172A;
  /* ... */
}

[data-theme="dark"] {
  --color-bg: #0F172A;
  --color-surface: #1E293B;
  --color-text: #F8FAFC;
  /* ... */
}
```

**Toggle Behavior:**
- Respects system preference by default
- User preference persisted in localStorage
- Smooth transition when toggling (0.2s)
- Toggle icon: Sun ↔ Moon with rotation animation

---

## Animations & Micro-interactions

### Philosophy

Animations should feel playful and responsive without being distracting. They guide attention and provide feedback.

### Global Animations

**Page Transitions:**
- Fade in with subtle upward motion
- Duration: 300ms
- Easing: ease-out

**Scroll Reveals:**
- Elements animate in as they enter viewport
- Staggered delays for lists/grids
- Options: fade-up, fade-in, scale-in

### Component-Specific Interactions

**Navigation:**
- Logo: subtle bounce on hover
- Nav links: underline grows from center on hover
- Mode toggle: flip animation when switching
- Theme toggle: sun/moon rotate and scale

**Buttons:**
- Hover: lift (translateY -2px) + shadow increase
- Active: press down (translateY 0) + shadow decrease
- Loading: pulse animation or spinner
- Success: checkmark appear animation

**Cards:**
- Hover: lift + border glow (blue in Portfolio mode, orange in Learner mode)
- Tech badges: scale up slightly on hover
- Image: subtle zoom on card hover

**Code Playground:**
- Run button: pulse when ready, spinner when executing
- Output: slide down + fade in
- Success: green flash on border
- Error: red flash + shake

**Filter Pills:**
- Active: fill animation from left
- Hover: background fade in
- Click: satisfying press effect

**Links:**
- Inline links: underline grows from left on hover
- Arrow links: arrow moves right on hover

**Form Elements:**
- Input focus: border color transition + subtle glow
- Checkbox/toggle: smooth slide animation

### Signature Animations

**Mode Toggle (Signature Interaction):**
```
Portfolio Mode → Learner Mode

┌─────────────────────────────┐
│  [💼]────────○──────[🎓]   │  (toggle slides)
└─────────────────────────────┘

- Background gradient shifts (blue → orange tint)
- Content cards flip/transform to reveal alternate content
- Subtle particle burst on toggle
- Sound effect option (muted by default)
```

**Typing Effect (Hero):**
- Typewriter effect for subtitle
- Cursor blinks
- Text changes based on mode

**Skill Bars (About Page):**
- Bars fill on scroll into view
- Percentage counter animates up

### Animation Specifications

| Animation | Duration | Easing | Trigger |
|-----------|----------|--------|---------|
| Page fade | 300ms | ease-out | Route change |
| Card hover | 200ms | ease-out | Mouse enter |
| Button press | 100ms | ease-in-out | Click |
| Scroll reveal | 500ms | ease-out | Viewport enter |
| Mode toggle | 400ms | spring | Toggle click |
| Typing effect | 50ms/char | linear | Page load |

### Reduced Motion Support

```css
@media (prefers-reduced-motion: reduce) {
  * {
    animation-duration: 0.01ms !important;
    transition-duration: 0.01ms !important;
  }
}
```

---

## Technical Stack

### Core Framework

| Technology | Purpose | Rationale |
|------------|---------|-----------|
| **Next.js 14+** | Framework | App Router, RSC, great DX, Vercel-optimized |
| **TypeScript** | Type Safety | Better DX, fewer bugs, self-documenting |
| **Tailwind CSS** | Styling | Rapid development, consistent design system |
| **Framer Motion** | Animations | Powerful, declarative, React-native |

### Content Management

| Technology | Purpose | Rationale |
|------------|---------|-----------|
| **MDX** | Blog/Content | Rich content with components |
| **Contentlayer** | Content Processing | Type-safe content, fast builds |
| **Gray Matter** | Frontmatter | Parse markdown metadata |

### Code Execution

| Technology | Purpose |
|------------|---------|
| **Pyodide** | Python in browser |
| **sql.js** | SQLite in browser |
| **esbuild-wasm** | TypeScript compilation |
| **Monaco Editor** | Code editing (optional) |
| **Prism/Shiki** | Syntax highlighting |

### State Management

| Technology | Purpose |
|------------|---------|
| **React Context** | Mode toggle, theme |
| **Zustand** | Complex state (if needed) |
| **localStorage** | Persistence |

### Development Tools

| Technology | Purpose |
|------------|---------|
| **ESLint** | Linting |
| **Prettier** | Formatting |
| **Husky** | Git hooks |
| **Jest/Vitest** | Testing |
| **Playwright** | E2E testing |

### Dependencies Overview

```json
{
  "dependencies": {
    "next": "^14.x",
    "react": "^18.x",
    "react-dom": "^18.x",
    "framer-motion": "^10.x",
    "contentlayer": "^0.3.x",
    "next-contentlayer": "^0.3.x",
    "next-themes": "^0.2.x",
    "pyodide": "^0.24.x",
    "sql.js": "^1.8.x",
    "prismjs": "^1.29.x",
    "@next/font": "^14.x"
  },
  "devDependencies": {
    "typescript": "^5.x",
    "tailwindcss": "^3.x",
    "autoprefixer": "^10.x",
    "postcss": "^8.x",
    "@types/react": "^18.x",
    "@types/node": "^20.x",
    "eslint": "^8.x",
    "prettier": "^3.x"
  }
}
```

---

## Data Models

### Project

```typescript
interface Project {
  // Identification
  slug: string;
  title: string;
  
  // Descriptions (mode-aware)
  description: {
    portfolio: string;  // Outcome-focused
    learner: string;    // Journey-focused
  };
  longDescription: {
    portfolio: string;
    learner: string;
  };
  
  // Media
  thumbnail: string;
  images: string[];
  
  // Categorization
  skills: Skill[];
  platforms: Platform[];
  purposes: Purpose[];
  
  // Code
  codeSnippets: CodeSnippet[];
  repoUrl: string;
  liveUrl?: string;
  
  // Metadata
  featured: boolean;
  status: 'completed' | 'in-progress' | 'maintained';
  startDate: Date;
  endDate?: Date;
  
  // Related
  relatedProjects: string[];  // slugs
}

type Skill = 
  | 'javascript' 
  | 'typescript' 
  | 'python' 
  | 'sql' 
  | 'algorithmic-design'
  | 'react'
  | 'svelte';

type Platform = 
  | 'appsscript' 
  | 'gcp' 
  | 'aws' 
  | 'vercel' 
  | 'node';

type Purpose = 
  | 'field-planning' 
  | 'sentiment-analysis' 
  | 'data-viz' 
  | 'automation'
  | 'web-app';
```

### CodeSnippet

```typescript
interface CodeSnippet {
  id: string;
  title: string;
  
  // Code content
  language: 'javascript' | 'typescript' | 'python' | 'sql';
  code: string;
  
  // Execution
  runnable: boolean;
  defaultOutput?: string;
  
  // For SQL
  sampleData?: {
    tableName: string;
    schema: string;
    data: string;
  }[];
  
  // Mode-aware
  comments: {
    portfolio: string;  // Minimal
    learner: string;    // Detailed explanations
  };
  
  // Learner mode extras
  challenges?: string[];
  hints?: string[];
  explanation?: string;
}
```

### BlogPost

```typescript
interface BlogPost {
  slug: string;
  title: string;
  
  excerpt: {
    portfolio: string;
    learner: string;
  };
  
  content: MDXContent;
  
  // Metadata
  publishedAt: Date;
  updatedAt?: Date;
  readingTime: number;
  
  // Categorization
  tags: string[];
  featured: boolean;
  
  // Media
  coverImage?: string;
  
  // Related
  relatedPosts: string[];
  relatedProjects?: string[];
}
```

### OpenSourceContribution

```typescript
interface OpenSourceContribution {
  id: string;
  projectName: string;
  projectLogo?: string;
  
  role: 'maintainer' | 'core-contributor' | 'contributor';
  description: string;
  
  docsUrl: string;
  repoUrl: string;
  
  contributions: {
    type: 'feature' | 'bugfix' | 'docs' | 'other';
    description: string;
    prUrl?: string;
  }[];
  
  techStack: string[];
  date: Date;
}
```

### Site Configuration

```typescript
interface SiteConfig {
  name: string;
  title: string;
  description: string;
  url: string;
  
  author: {
    name: string;
    email: string;
    github: string;
    linkedin: string;
    twitter?: string;
  };
  
  // Default mode for first-time visitors
  defaultMode: 'portfolio' | 'learner';
  
  // Feature flags
  features: {
    blog: boolean;
    docs: boolean;
    codeExecution: boolean;
  };
}
```

---

## File Structure

```
portfolio/
├── public/
│   ├── images/
│   │   ├── projects/
│   │   ├── blog/
│   │   └── about/
│   ├── fonts/
│   └── favicon.ico
│
├── src/
│   ├── app/
│   │   ├── layout.tsx
│   │   ├── page.tsx                 # Homepage
│   │   ├── projects/
│   │   │   ├── page.tsx             # Projects listing
│   │   │   └── [slug]/
│   │   │       └── page.tsx         # Individual project
│   │   ├── blog/
│   │   │   ├── page.tsx             # Blog listing
│   │   │   └── [slug]/
│   │   │       └── page.tsx         # Individual post
│   │   ├── docs/
│   │   │   └── page.tsx             # Documentation
│   │   └── about/
│   │       └── page.tsx             # About
│   │
│   ├── components/
│   │   ├── layout/
│   │   │   ├── Navigation.tsx
│   │   │   ├── Footer.tsx
│   │   │   ├── ModeToggle.tsx
│   │   │   └── ThemeToggle.tsx
│   │   ├── ui/
│   │   │   ├── Button.tsx
│   │   │   ├── Card.tsx
│   │   │   ├── Badge.tsx
│   │   │   ├── Input.tsx
│   │   │   └── ...
│   │   ├── projects/
│   │   │   ├── ProjectCard.tsx
│   │   │   ├── ProjectFilter.tsx
│   │   │   ├── ProjectGrid.tsx
│   │   │   └── CodePlayground.tsx
│   │   ├── blog/
│   │   │   ├── PostCard.tsx
│   │   │   ├── PostList.tsx
│   │   │   └── TableOfContents.tsx
│   │   ├── home/
│   │   │   ├── Hero.tsx
│   │   │   ├── FeaturedProjects.tsx
│   │   │   └── RecentPosts.tsx
│   │   └── shared/
│   │       ├── AnimatedSection.tsx
│   │       ├── SkillBadge.tsx
│   │       └── SocialLinks.tsx
│   │
│   ├── lib/
│   │   ├── utils.ts
│   │   ├── constants.ts
│   │   └── code-execution/
│   │       ├── javascript.ts
│   │       ├── python.ts
│   │       └── sql.ts
│   │
│   ├── hooks/
│   │   ├── useMode.ts
│   │   ├── useTheme.ts
│   │   └── useCodeExecution.ts
│   │
│   ├── context/
│   │   ├── ModeContext.tsx
│   │   └── ThemeContext.tsx
│   │
│   ├── styles/
│   │   └── globals.css
│   │
│   └── types/
│       └── index.ts
│
├── content/
│   ├── projects/
│   │   ├── project-one.mdx
│   │   └── project-two.mdx
│   └── blog/
│       ├── post-one.mdx
│       └── post-two.mdx
│
├── contentlayer.config.ts
├── tailwind.config.ts
├── next.config.js
├── tsconfig.json
└── package.json
```

---

## Deployment & Hosting

### Platform: Vercel

**Rationale:**
- Native Next.js support
- Automatic deployments from Git
- Edge functions for performance
- Analytics built-in
- Preview deployments for PRs
- Free tier sufficient for portfolio

### Custom Domain Setup

**Steps:**
1. Purchase domain (Namecheap, Google Domains, etc.)
2. Add domain in Vercel project settings
3. Configure DNS:
   - A record: `76.76.21.21`
   - CNAME for www: `cname.vercel-dns.com`
4. Enable HTTPS (automatic)
5. Configure redirects (www → non-www or vice versa)

### Environment Variables

```
# .env.local
NEXT_PUBLIC_SITE_URL=https://yourdomain.com
NEXT_PUBLIC_GA_ID=G-XXXXXXXXXX  # Optional: Analytics
```

### Deployment Configuration

```json
// vercel.json
{
  "buildCommand": "npm run build",
  "outputDirectory": ".next",
  "framework": "nextjs",
  "regions": ["iad1"],
  "headers": [
    {
      "source": "/fonts/(.*)",
      "headers": [
        {
          "key": "Cache-Control",
          "value": "public, max-age=31536000, immutable"
        }
      ]
    }
  ]
}
```

### Performance Targets

| Metric | Target |
|--------|--------|
| Lighthouse Performance | > 90 |
| First Contentful Paint | < 1.5s |
| Largest Contentful Paint | < 2.5s |
| Time to Interactive | < 3.5s |
| Cumulative Layout Shift | < 0.1 |

### SEO Configuration

- Dynamic meta tags per page
- Open Graph images
- Structured data (JSON-LD)
- Sitemap generation
- robots.txt

---

## Development Phases

### Phase 1: Foundation (Week 1)
**Goal:** Core infrastructure and design system

**Tasks:**
- [ ] Initialize Next.js project with TypeScript
- [ ] Configure Tailwind with custom design tokens
- [ ] Set up ESLint, Prettier, Husky
- [ ] Create base layout (Navigation, Footer)
- [ ] Implement theme toggle (light/dark)
- [ ] Implement mode toggle (portfolio/learner) with context
- [ ] Create core UI components (Button, Card, Badge)
- [ ] Set up Framer Motion and base animations
- [ ] Deploy to Vercel (staging)

**Deliverable:** Navigable shell with working toggles

---

### Phase 2: Content Infrastructure (Week 2)
**Goal:** Content management and data layer

**Tasks:**
- [ ] Set up Contentlayer for MDX
- [ ] Define content schemas (Project, BlogPost)
- [ ] Create sample content (2 projects, 2 blog posts)
- [ ] Build content utility functions
- [ ] Implement mode-aware content switching

**Deliverable:** Content system ready for real data

---

### Phase 3: Homepage & Projects (Week 3)
**Goal:** Main showcase pages

**Tasks:**
- [ ] Build Homepage:
  - [ ] Hero section with typing animation
  - [ ] Featured projects grid
  - [ ] Recent blog posts
  - [ ] Scroll animations
- [ ] Build Projects page:
  - [ ] Filter system (skills, platforms, purposes)
  - [ ] Project cards with mode-aware content
  - [ ] URL-based filter state
  - [ ] Animations and transitions
- [ ] Build Individual Project page:
  - [ ] Header and metadata
  - [ ] Mode-aware descriptions
  - [ ] Image gallery
  - [ ] Related projects

**Deliverable:** Functional homepage and projects section

---

### Phase 4: Code Execution (Week 4)
**Goal:** Interactive code playground

**Tasks:**
- [ ] Build CodePlayground component:
  - [ ] Syntax highlighting
  - [ ] Language tabs
  - [ ] Run/Reset/Copy buttons
  - [ ] Output display
- [ ] Implement JavaScript execution (sandboxed)
- [ ] Implement Python execution (Pyodide)
- [ ] Implement SQL execution (sql.js)
- [ ] Add sample databases for SQL
- [ ] Learner mode enhancements (explanations, challenges)
- [ ] Error handling and loading states

**Deliverable:** Fully functional code playground

---

### Phase 5: Blog & Documentation (Week 5)
**Goal:** Content pages

**Tasks:**
- [ ] Build Blog listing page:
  - [ ] Post cards
  - [ ] Tag filtering
  - [ ] Search (optional)
- [ ] Build Individual Blog Post page:
  - [ ] MDX rendering
  - [ ] Table of contents
  - [ ] Code blocks with playground integration
  - [ ] Related posts
- [ ] Build Documentation page:
  - [ ] Contribution cards
  - [ ] Links and metadata

**Deliverable:** Complete blog and docs sections

---

### Phase 6: About & Polish (Week 6)
**Goal:** Personal page and final touches

**Tasks:**
- [ ] Build About page:
  - [ ] Bio section with mode variants
  - [ ] Skills visualization
  - [ ] Timeline
  - [ ] Email contact link
  - [ ] Social links
- [ ] Polish all animations
- [ ] Responsive design audit
- [ ] Accessibility audit (a11y)
- [ ] Performance optimization
- [ ] SEO implementation
- [ ] Cross-browser testing

**Deliverable:** Complete, polished website

---

### Phase 7: Content & Launch (Week 7)
**Goal:** Real content and go-live

**Tasks:**
- [ ] Write all project descriptions (both modes)
- [ ] Create code snippets for each project
- [ ] Write blog posts
- [ ] Document open source contributions
- [ ] Write about page content
- [ ] Gather screenshots and media
- [ ] Custom domain setup
- [ ] Analytics setup
- [ ] Final QA
- [ ] Launch! 🚀

**Deliverable:** Live portfolio website

---

## Future Considerations

### Potential Enhancements

**Near-term:**
- Comment system for blog (utterances or giscus)
- RSS feed for blog
- Newsletter signup
- Search across all content
- Reading progress indicator

**Medium-term:**
- GitHub API integration for live stats
- Embedded Svelte REPL for Svelte projects
- Code snippet sharing (generate shareable links)
- Achievement/badge system in learner mode
- Dark/light mode per-section override

**Long-term:**
- Interactive tutorials (step-by-step guided learning)
- Code challenges with validation
- Progress tracking for learners
- Multiple language translations
- Native mobile app (React Native)

### Analytics & Feedback

**Track:**
- Page views and time on page
- Mode toggle usage (which mode is more popular?)
- Code execution events
- Filter usage patterns
- Scroll depth

**Gather feedback:**
- Simple feedback button ("Was this helpful?")
- Optional exit survey
- GitHub issues for suggestions

---

## Appendix

### A. Sample Project Entry

```mdx
---
slug: sentiment-analyzer
title: Customer Sentiment Analyzer
description:
  portfolio: "Built a sentiment analysis pipeline processing 10K+ reviews daily with 94% accuracy, reducing manual review time by 60%."
  learner: "I was curious how companies understand customer feelings at scale. This project taught me NLP fundamentals, ML model selection, and data pipeline architecture."
thumbnail: /images/projects/sentiment/thumb.png
skills: [python, sql, algorithmic-design]
platforms: [gcp]
purposes: [sentiment-analysis, automation]
featured: true
status: completed
startDate: 2024-03-01
endDate: 2024-06-15
repoUrl: https://github.com/username/sentiment-analyzer
---

## Overview

<ModeContent
  portfolio={`
    ### Problem
    Manual review of customer feedback was consuming 20+ hours weekly...
    
    ### Solution
    Implemented an automated sentiment analysis pipeline...
    
    ### Results
    - 94% accuracy on sentiment classification
    - 60% reduction in manual review time
    - Real-time dashboard for stakeholder visibility
  `}
  learner={`
    ### The Curiosity
    I started wondering: how do big companies actually know what their customers think?
    
    ### The Journey
    This project took me through several learning phases...
    
    ### Key Lessons
    - NLP is more art than science
    - Start simple, add complexity only when needed
    - Domain-specific training data is gold
  `}
/>

## Code Examples

<CodePlayground
  snippets={[
    {
      language: "python",
      title: "Sentiment Classification",
      code: `
from textblob import TextBlob

def analyze_sentiment(text):
    analysis = TextBlob(text)
    if analysis.sentiment.polarity > 0:
        return 'positive'
    elif analysis.sentiment.polarity < 0:
        return 'negative'
    return 'neutral'

# Try it!
review = "This product exceeded my expectations!"
print(analyze_sentiment(review))
      `,
      runnable: true
    }
  ]}
/>
```

### B. Color Palette Visual Reference

```
┌─────────────────────────────────────────────────────────────┐
│  PRIMARY                                                    │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐           │
│  │ #2563EB │ │ #1E40AF │ │ #F97316 │ │ #EA580C │           │
│  │ Ocean   │ │ Deep    │ │ Sunset  │ │ Warm    │           │
│  │ Blue    │ │ Blue    │ │ Orange  │ │ Orange  │           │
│  └─────────┘ └─────────┘ └─────────┘ └─────────┘           │
│                                                             │
│  LIGHT MODE                     DARK MODE                   │
│  ┌─────────┐ ┌─────────┐       ┌─────────┐ ┌─────────┐     │
│  │ #FAFAFA │ │ #FFFFFF │       │ #0F172A │ │ #1E293B │     │
│  │ BG      │ │ Surface │       │ BG      │ │ Surface │     │
│  └─────────┘ └─────────┘       └─────────┘ └─────────┘     │
│  ┌─────────┐ ┌─────────┐       ┌─────────┐ ┌─────────┐     │
│  │ #0F172A │ │ #64748B │       │ #F8FAFC │ │ #94A3B8 │     │
│  │ Text    │ │ Text 2  │       │ Text    │ │ Text 2  │     │
│  └─────────┘ └─────────┘       └─────────┘ └─────────┘     │
└─────────────────────────────────────────────────────────────┘
```

### C. Animation Easing Reference

```
Standard:    ease-out      (0.0, 0.0, 0.2, 1.0)  - Most UI transitions
Entrance:    ease-out      (0.0, 0.0, 0.2, 1.0)  - Elements appearing
Exit:        ease-in       (0.4, 0.0, 1.0, 1.0)  - Elements leaving
Bounce:      spring        (stiffness: 400, damping: 30)
Smooth:      ease-in-out   (0.4, 0.0, 0.2, 1.0)  - Continuous motion
```

---

## Document History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | Jan 2026 | Initial scoping document |

---

*End of Scoping Document*
