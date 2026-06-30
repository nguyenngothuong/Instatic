# Design Brief Prototype and Local Sync Plan

> **Status:** proposed  
> **Date:** 2026-06-30  
> **Goal:** make AI-generated sites less generic by giving the agent a reusable design brief, then let users sync site output into a local folder they can own with Git/GitHub.

## Source References

- [getdesign.md](https://getdesign.md/) positions `DESIGN.md` as a reusable design reference for AI-built websites, covering colors, type, spacing, components, and reasoning so new pages keep a specific visual language.
- [VoltAgent/awesome-design-md](https://github.com/VoltAgent/awesome-design-md) describes `DESIGN.md` as a plain-text design system document for AI agents, with sections for visual theme, color roles, typography, component styling, layout rules, depth, do/don't guidance, responsive behavior, and agent prompts.
- Instatic already has a structured design-token system (`site.settings.framework`), a `doc` `SiteFile` type, AI token-writing tools, a full site-transfer ZIP format, and a publisher that emits static HTML/CSS/JS artefacts.

## Review

### 1. Design Brief Prototype Inspired By getdesign.md

The product gap is real. The current AI workflow can mutate tokens and components, but it does not have a durable design intent layer. When a user asks for a new page, the agent sees the current site state and tool descriptions, not a reusable design language comparable to `DESIGN.md`. That is why output can feel one-note or generic.

The right Instatic version is not just "paste a markdown file into chat". It should be a first-class **Design Brief**:

- stored with the site,
- readable by the AI runtime,
- previewable by the user,
- convertible into framework tokens,
- exportable as `DESIGN.md` for external coding agents.

Use getdesign.md as the product reference, not as a dependency. The MVP should support manually pasting/importing a `DESIGN.md` file and using it inside the AI context. A later version can add a catalog browser, reference-site analysis, screenshots, or paid/private brief generation.

### 2. Local Sync For User-Owned GitHub Workflow

The current `Site Transfer` flow is portable but download-oriented: it emits `site-bundle-<timestamp>.zip` with `.instatic/site-bundle.json` plus media. That solves backup/migration, but it is not a continuous local source workflow.

The sync feature should write files into a local folder. The user can then run GitHub Desktop, `git remote add`, `gh repo create`, or any Git workflow outside Instatic. Instatic should not start with GitHub OAuth or direct commits because that mixes product concerns with token storage, branch management, merge conflicts, and support burden.

The sync feature needs two output modes:

- **Editable snapshot:** round-trippable Instatic source, optimized for backup/reimport and Git history.
- **Published static output:** browser-ready HTML/CSS/JS/media, optimized for deployment and inspection.

Both are valuable. Editable snapshot is the product "source of truth"; static output is the deployable artifact.

## Product Shape

### Design Brief UX

Add a "Design Brief" area in the AI/site workspace:

1. **Import / Paste**
   - Paste markdown.
   - Upload `DESIGN.md`.
   - Optional starter templates: "Minimal SaaS", "Editorial", "Developer tool", "Commerce".

2. **Preview**
   - Show sections: Theme, Color roles, Typography, Components, Layout, Do/Don't.
   - Render color swatches and typography samples where the parser can identify them.
   - Show a validation state: missing color roles, missing type hierarchy, missing component guidance.

3. **Apply To Site**
   - "Use as AI context" stores it without mutation.
   - "Apply tokens" asks the AI/tool runner to map brief content into existing framework tokens.
   - "Restyle current page" uses the brief as a constraint for a controlled mutation pass.

4. **Export**
   - Include `DESIGN.md` in local sync.
   - Include it in site-transfer bundle because `SiteFile` entries already travel with the site shell.

### Local Sync UX

Add a "Local Sync" panel near Export/Import:

1. User creates a target:
   - label,
   - local folder path,
   - mode: `editable`, `static`, or `both`,
   - include media,
   - include design brief.

2. User previews:
   - files to create/update/delete,
   - estimated bytes,
   - warnings for missing publish, missing media files, path not allowed, or dirty target policy.

3. User runs sync:
   - writes to a temp directory,
   - atomically swaps/updates the target files,
   - records last sync summary.

4. User owns Git:
   - Instatic can show target path and last sync status.
   - Instatic should not store GitHub tokens in v1.
   - Optional v2 can detect `.git` and show branch/status read-only.

## Architecture

### Design Brief Storage

Use the existing `SiteFile` system for the raw markdown:

```ts
SiteFile {
  type: 'doc',
  path: 'DESIGN.md',
  content: string
}
```

Add a derived parser module, not a new stored schema first:

```text
src/core/designBrief/
├── parseDesignBrief.ts       — markdown section parser + tolerant extraction
├── digestDesignBrief.ts      — compact context for AI prompts
├── tokenSuggestions.ts       — optional color/type/spacing candidates
└── schemas.ts                — typed DesignBriefDigest / warnings
```

Keep markdown as source of truth. The parsed digest is derived state used by UI and AI prompts. This avoids locking users into a custom schema before the product learns what design briefs look like in practice.

### AI Integration

Thread a compact design brief digest into the site AI system prompt:

- Source: `DESIGN.md` site file, if present.
- Digest budget: keep it compact and structured; do not paste an unbounded markdown file into every turn.
- Priority: design brief is guidance, not a hard-coded mutation. Tool calls still use existing token and page-write APIs.

Candidate integration points:

- `server/ai/tools/site/snapshot.ts`: include brief metadata/digest in the site snapshot.
- `server/ai/tools/site/systemPrompt.ts`: add "Design brief constraints" section.
- `server/ai/tools/site/writeTools.ts`: tighten tool descriptions so token tools map from brief concepts into framework tokens.

### Local Sync Storage

Add a server-side target registry. Do not store GitHub credentials.

```ts
interface LocalSyncTarget {
  id: string
  label: string
  rootDir: string
  mode: 'editable' | 'static' | 'both'
  includeMedia: boolean
  includeDesignBrief: boolean
  deleteExtraneous: boolean
  createdAt: string
  updatedAt: string
  lastSyncedAt: string | null
  lastResult: LocalSyncResult | null
}
```

Security rule: local sync can write only inside configured allowlisted roots. Suggested config:

```text
INSTATIC_LOCAL_SYNC_ROOTS=/Users/me/Sites:/Users/me/Projects
```

If no allowlist is configured, the Local Sync UI is disabled with a server-side reason. This prevents an admin session from becoming arbitrary filesystem write access.

### Local Sync Output Layout

For `editable` mode:

```text
<target>/
├── DESIGN.md
├── README.md
├── .instatic/
│   ├── site-bundle.json
│   └── sync-manifest.json
└── media/
    └── <storagePath>
```

For `static` mode:

```text
<target>/
├── README.md
├── DESIGN.md
├── dist/
│   ├── index.html
│   ├── <route>.html
│   ├── _instatic/
│   │   ├── css/
│   │   └── assets/
│   └── uploads/
└── .instatic/
    └── sync-manifest.json
```

The editable snapshot should reuse the `SiteBundleArchiveManifest` shape without ZIP wrapping. The static output should reuse the publisher's baked artefacts rather than invent a second renderer.

### API Contract

New endpoints under the existing CMS admin surface:

```text
GET    /admin/api/cms/local-sync/targets
POST   /admin/api/cms/local-sync/targets
PATCH  /admin/api/cms/local-sync/targets/:id
DELETE /admin/api/cms/local-sync/targets/:id

POST   /admin/api/cms/local-sync/targets/:id/preview
POST   /admin/api/cms/local-sync/targets/:id/run
```

Responses should be typed with TypeBox and follow existing `jsonResponse({ error })` semantics.

Preview response:

```ts
interface LocalSyncPreview {
  targetId: string
  rootDir: string
  mode: 'editable' | 'static' | 'both'
  files: Array<{
    path: string
    action: 'create' | 'update' | 'delete' | 'unchanged'
    sizeBytes: number
  }>
  totals: {
    create: number
    update: number
    delete: number
    unchanged: number
    bytes: number
  }
  warnings: string[]
}
```

Run response:

```ts
interface LocalSyncResult extends LocalSyncPreview {
  syncedAt: string
  durationMs: number
}
```

### Safety Requirements

- Validate target path against allowlisted roots.
- Reject writes that escape target via `..`, symlinks, or absolute path tricks.
- Write into a temp directory first, then promote.
- Never include sessions, users, passwords, AI credentials, audit logs, or provider tokens.
- Do not run `git add`, `git commit`, `git push`, or store GitHub OAuth in v1.
- Make delete behavior explicit with `deleteExtraneous`; default false for early MVP.

## Implementation Plan

### Phase 0 — Spec And Boundaries

- [ ] Commit this plan.
- [ ] Add a short feature doc for "Design Brief" and "Local Sync" after implementation begins.
- [ ] Decide whether Local Sync targets live in a DB table or server config. Recommended: DB table for target metadata, server env allowlist for writable roots.

### Phase 1 — Design Brief MVP

- [ ] Add `src/core/designBrief` parser/digest module.
- [ ] Add tests with real-ish markdown fixtures inspired by getdesign.md section structure.
- [ ] Add store helpers for reading/writing a `DESIGN.md` `SiteFile`.
- [ ] Add UI in the AI/site workspace for paste/upload/edit/preview.
- [ ] Add AI prompt integration using a compact digest.
- [ ] Add an "Apply tokens" action that routes through existing AI token tools.
- [ ] Add tests:
  - parser extracts sections,
  - digest stays bounded,
  - `DESIGN.md` file round-trips in site shell,
  - AI prompt includes digest only when present.

### Phase 2 — Local Sync Editable Snapshot

- [ ] Extract export selection/manifest building from `server/handlers/cms/export.ts` into a reusable service.
- [ ] Add local sync target schemas and CRUD endpoints.
- [ ] Add allowlisted-root path validation and containment tests.
- [ ] Add `editable` writer:
  - `.instatic/site-bundle.json`,
  - `media/<storagePath>`,
  - `DESIGN.md`,
  - generated `README.md`,
  - `.instatic/sync-manifest.json`.
- [ ] Add preview endpoint that computes create/update/delete without writing.
- [ ] Add run endpoint that writes through a temp directory.
- [ ] Add Data workspace UI for target setup, preview, and run status.

### Phase 3 — Local Sync Static Output

- [ ] Reuse current published artefacts when available.
- [ ] Add "publish before sync" option, or require latest publish in v1.
- [ ] Copy baked HTML/CSS/JS/media into `dist/`.
- [ ] Add warnings for dynamic hole pages that still require an Instatic server.
- [ ] Add tests for static output path layout and missing-publish warnings.

### Phase 4 — Git-Aware Nice-To-Have

- [ ] Detect whether target folder has `.git`.
- [ ] Show read-only branch and dirty count, if cheap and safe.
- [ ] Offer copyable Git commands in docs, not automatic execution.
- [ ] Defer GitHub OAuth/App integration until local sync proves useful.

## Recommended PR Split

1. **Design Brief storage + parser**
   - Pure core module, no UI.
2. **Design Brief UI + AI prompt integration**
   - User-visible prototype.
3. **Local Sync target model + safety gate**
   - No writer yet; path validation and CRUD.
4. **Editable local sync writer**
   - Round-trippable source output.
5. **Static output mode**
   - Publish/deploy artifact export.
6. **Git-aware status**
   - Optional, read-only.

## Open Questions

- Should `DESIGN.md` be included in normal site-transfer ZIP by default? Recommended: yes, because it is a `doc` `SiteFile` in the site shell.
- Should Local Sync be enabled by default in hosted deployments? Recommended: no. Enable only when `INSTATIC_LOCAL_SYNC_ROOTS` is configured.
- Should static sync render draft or published site? Recommended: published site for v1, because it matches what visitors see and reuses existing publisher semantics.
- How should dynamic pages be represented in static output? Recommended: sync the shell with warnings; fully dynamic static export is a separate feature.
- Should Instatic ever push to GitHub directly? Recommended: only later via explicit GitHub App/OAuth, separate from local sync.

## Success Criteria

- A user can paste a `DESIGN.md`, see it in the app, and have future AI page generation follow it.
- A user can run local sync and get a folder that is meaningful in Git history.
- Editable sync can be re-imported through existing site-transfer semantics.
- Static sync can be inspected in a browser and deployed as ordinary files when the site has no server-only dynamic holes.
- No secrets are written to synced files.
