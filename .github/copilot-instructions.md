# Quartz v4 AI Agent Guide

## Project Overview

This is **MoutonQuartz** - a fork of a static site generator (SSG) for publishing digital gardens and notes. It transforms Markdown files in `content/` into a fully-featured website with features like backlinks, graph views, search, and Obsidian compatibility.
This is a modified fork for themes and feature purposes.

**Core Technology Stack:**
- TypeScript + Preact (JSX) for UI components
- Unified/Remark/Rehype pipeline for Markdown processing
- pnpm for package management
- Node.js 20 or >=22 required

## Architecture

### Three-Phase Build System

Quartz uses a **transformer → filter → emitter** pipeline ([quartz/build.ts](../quartz/build.ts)):

1. **Transformers** ([quartz/plugins/transformers/](../quartz/plugins/transformers/)): Parse and modify Markdown/HTML AST
   - Examples: `FrontMatter()`, `ObsidianFlavoredMarkdown()`, `GitHubFlavoredMarkdown()`
   - Interface: `QuartzTransformerPluginInstance` with `markdownPlugins()` and/or `htmlPlugins()`
   
2. **Filters** ([quartz/plugins/filters/](../quartz/plugins/filters/)): Decide which content to publish
   - Example: `RemoveDrafts()` skips files with `draft: true` in frontmatter
   - Interface: `QuartzFilterPluginInstance.shouldPublish()`

3. **Emitters** ([quartz/plugins/emitters/](../quartz/plugins/emitters/)): Generate output files (HTML, assets, etc.)
   - Examples: `ContentPage()`, `FolderPage()`, `Assets()`, `Static()`
   - Interface: `QuartzEmitterPluginInstance.emit()` + `getQuartzComponents()`

**Critical**: All plugins configured in [quartz.config.ts](../quartz.config.ts) `plugins` object.

### Component System

Components are Preact functions ([quartz/components/](../quartz/components/)) that:
- Receive `QuartzComponentProps` (includes `ctx`, `fileData`, `cfg`, `allFiles`)
- Export optional static properties: `.css`, `.beforeDOMLoaded`, `.afterDOMLoaded`
- Are composed in [quartz.layout.ts](../quartz.layout.ts) into `beforeBody`, `left`, `right` layout regions

**Layout Example:**
```typescript
export const defaultContentPageLayout: PageLayout = {
  beforeBody: [Component.Breadcrumbs(), Component.ArticleTitle()],
  left: [Component.PageTitle(), Component.Search()],
  right: [Component.Graph(), Component.TableOfContents()],
}
```

### Path & Slug System

**Strict type system** ([quartz/util/path.ts](../quartz/util/path.ts)):
- `FilePath`: Absolute path with extension (e.g., `content/notes/test.md`)
- `FullSlug`: Web path without extension (e.g., `notes/test`)
- `SimpleSlug`: Like FullSlug but no trailing `/index`
- `RelativeURL`: Relative path starting with `./` or `../`

**Never mix these types** - use conversion functions like `slugifyFilePath()`, `resolveRelative()`, `pathToRoot()`.

### Build Process

- Entry: [quartz/bootstrap-cli.mjs](../quartz/bootstrap-cli.mjs) → CLI handlers in [quartz/cli/](../quartz/cli/)
- Main build: [quartz/build.ts](../quartz/build.ts) `buildQuartz()` function
- Workers: Uses `workerpool` for parallel Markdown processing ([quartz/worker.ts](../quartz/worker.ts))
- Hot reload: Chokidar watches files in serve mode (`--serve`), supports fast rebuilds (`--fastRebuild`)
- Dependency graphs: Emitters can provide dependency graphs for incremental rebuilds

## Development Workflows

### Local Development
```bash
npx quartz build --serve    # Build + serve with hot reload
npm run docs                # Build docs/ folder specifically
```

### Key Commands
- `npm run check`: TypeScript check + Prettier validation
- `npm run format`: Auto-format with Prettier
- `npm test`: Run path and depgraph tests

### File Conventions

**User-Editable:**
- [quartz.config.ts](../quartz.config.ts): Plugin configuration, theme, analytics, base URL
- [quartz.layout.ts](../quartz.layout.ts): Component layout for different page types
- [content/](../content/): Markdown files to publish

**Framework Code:**
- [quartz/plugins/](../quartz/plugins/): Core plugin implementations
- [quartz/components/](../quartz/components/): UI components (Preact)
- [quartz/util/](../quartz/util/): Shared utilities (paths, resources, theme)
- [quartz/processors/](../quartz/processors/): Markdown parsing/filtering/emission logic

## Important Patterns

### Creating a Plugin

**Transformer Example:**
```typescript
// quartz/plugins/transformers/MyTransformer.ts
export const MyTransformer: QuartzTransformerPlugin = () => {
  return {
    name: "MyTransformer",
    markdownPlugins() {
      return [remarkPlugin]  // Remark plugin for MD AST
    },
    htmlPlugins() {
      return [rehypePlugin]  // Rehype plugin for HTML AST
    },
  }
}
```

**Emitter Example:**
```typescript
export const MyEmitter: QuartzEmitterPlugin = () => {
  return {
    name: "MyEmitter",
    async emit(ctx, content, resources) {
      // Generate files, return array of FilePaths
      return []
    },
    getQuartzComponents() {
      return [MyComponent]
    },
  }
}
```

### Creating a Component

```typescript
// quartz/components/MyComponent.tsx
import { QuartzComponent, QuartzComponentProps } from "./types"

const MyComponent: QuartzComponent = ({ fileData, cfg }: QuartzComponentProps) => {
  return <div>Content</div>
}

MyComponent.css = `/* Component styles */`
MyComponent.afterDOMLoaded = `/* Client-side JS as string */`

export default (() => MyComponent) as QuartzComponentConstructor
```

### Preact Usage

- **JSX Pragma**: `jsxImportSource: "preact"` in [tsconfig.json](../tsconfig.json)
- **Server-side rendering**: `preact-render-to-string` in [quartz/components/renderPage.tsx](../quartz/components/renderPage.tsx)
- **Import**: `import { jsx, jsxs } from "preact/jsx-runtime"`
- Components are **stateless** - rendered once server-side, hydrated with vanilla JS

### Markdown Processing

**Unified Pipeline** ([quartz/processors/parse.ts](../quartz/processors/parse.ts)):
1. `remarkParse`: Markdown → MD AST
2. Transformer `markdownPlugins`: MD AST → modified MD AST
3. `remarkRehype`: MD AST → HTML AST
4. Transformer `htmlPlugins`: HTML AST → modified HTML AST

**Access AST**: Transformers receive `(ctx: BuildCtx) => PluggableList`

### Configuration Access

- Build context: `ctx.cfg` (from [quartz.config.ts](../quartz.config.ts))
- Theme colors: `ctx.cfg.configuration.theme.colors`
- Plugins: `ctx.cfg.plugins.{transformers|filters|emitters}`
- All slugs: `ctx.allSlugs` (computed during build)

## Common Pitfalls

1. **Don't use Node.js path APIs in browser-shared code** - [quartz/util/path.ts](../quartz/util/path.ts) is isomorphic
2. **Component CSS must be strings** - use `Component.css = "..."` not imports
3. **Fast rebuild requires dependency graphs** - implement `getDependencyGraph()` in emitters
4. **Plugin order matters** - transformers run sequentially, order in [quartz.config.ts](../quartz.config.ts) is critical
5. **Resource loading**: `externalResources()` in transformers adds global CSS/JS

## Project-Specific Notes

- **Theme**: Uses custom color scheme defined in [quartz.config.ts](../quartz.config.ts) `theme.colors`
- **Fonts**: Google Fonts via CDN (Schibsted Grotesk, Source Sans Pro, IBM Plex Mono)
- **Analytics**: Configured for Plausible in [quartz.config.ts](../quartz.config.ts)
- **Deployment**: GitHub CI auto-builds and deploys (mentioned in [README.md](../README.md))
