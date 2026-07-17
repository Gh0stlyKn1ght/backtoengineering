# Build Process & Architecture

This document tracks the collaborative build process with Claude and the architectural decisions made for the Back To Engineering tech tree learning site.

## Maintainer & Credits

- **Maintainer**: [Gh0stlyKn1ght](https://github.com/Gh0stlyKn1ght) — this repo lives at [Gh0stlyKn1ght/backtoengineering](https://github.com/Gh0stlyKn1ght/backtoengineering)
- **Upstream**: forked from [iuliaferoli/backtoengineering](https://github.com/iuliaferoli/backtoengineering) by Iulia Feroli, creator of the Back To Engineering YouTube channel
- Credit appears on the site itself: header repo button + footer copyright (MkDocs Material via `mkdocs.yml`), a GitHub button in the tree homepage HUD (`docs/index.html`), and the README Credits section

## Project Genesis

**Goal**: Build a learning website for robotics and AI enthusiasts featuring:
- A visual dependency graph ("tech tree") showing prerequisite relationships between topics
- Hands-on projects mapped to required knowledge
- Single-person maintainability with minimal time investment
- Easy content addition/editing

**Key Requirements**:
- Simplicity in architecture and software stack
- Static site (no servers, databases, or backend complexity)
- Python-first tooling (maintainer's strongest language)
- Future-proof for phase 2 features (user progress tracking)

## Technical Stack Selection

### Final Stack
- **MkDocs with Material theme**: Content pages and static site generation
- **Python build script** (`scripts/build_graph.py`): Frontmatter parser and graph validator
- **Standalone tree page** (`docs/index.html`): Custom renderer using ELK (elkjs) for layered layout, HTML cards for nodes, one SVG layer for edges, custom pan/zoom — served as the homepage
- **MkDocs hooks** (`scripts/hooks.py`): YouTube embeds + auto-generated "Next in the tree" links on every content page
- **GitHub Pages + GitHub Actions**: Zero-cost hosting with auto-deploy on push, custom domain `www.backtoengineering.com` (CNAME)
- **Umami analytics** (optional): injected at deploy time from repo variables; local builds need no credentials

### Why Not Hugo?
Hugo is written in Go, but users write Go templates (a quirky mini-language), not Go code. Since the maintainer knows Python best, MkDocs with Jinja2-like templates is more accessible. Hugo's popularity doesn't translate to editability for this use case.

### Why Not Pelican?
Pelican works but has a small, semi-abandoned plugin ecosystem. MkDocs Material has a massive community, excellent documentation, and active maintenance.

### Why Not Astro?
Astro's content collections would handle frontmatter validation natively, and interactive islands would simplify the graph page. However, it requires the Node ecosystem. Python comfort was prioritized for long-term maintainability.

### Renderer history: Cytoscape → custom ELK page
The first version rendered the tree with Cytoscape.js + dagre inside a normal MkDocs page (`docs/js/tech-tree.js`). It was replaced by a standalone full-page renderer (`docs/index.html`, no MkDocs theme) because the site needed: group boxes around ecosystems (ROS 2, NVIDIA, Python foundations, hardware…), era/tier partitioning, HTML node cards with icons and video counts, an info side panel, and a Civ-style HUD (brand, legend, zoom, horizontal scrollbar). ELK's layered algorithm with partitions handles all of that; Cytoscape/dagre did not.

**`docs/js/tech-tree.js` is legacy and no longer loaded anywhere** — nothing in `mkdocs.yml` or any page references it. Safe to delete whenever.

## Architecture

### Content as Data
Every topic and project is a Markdown file with YAML frontmatter:

```yaml
---
id: slam                       # stable unique identifier — never rename
title: SLAM
type: topic                    # topic | project
category: ai-ml                # drives node color (see categories below)
tree_icon: map-2               # Tabler icon name (tabler.io/icons), optional
group: ros-ecosystem           # optional: draws a labeled box around members
prerequisites: [ros2-basics, linear-algebra]
videos:                        # optional; single `video:` key also works
  - https://youtu.be/...
thumbnail: https://...         # optional
---
```

**Critical design decision**: IDs are the stable identity. Never rename an ID after publishing; progress tracking (phase 2) will reference these IDs. Changing an ID breaks user progress.

### Build Pipeline

1. **Content scan**: `build_graph.py` reads all `.md` files in `docs/topics/` and `docs/projects/`
2. **Frontmatter parsing**: PyYAML
3. **Validation**: required fields, no duplicate IDs, all prerequisites exist, no cycles (iterative DFS), valid types. Build fails loudly on any error
4. **Graph generation**: writes `docs/assets/graph.json` — per node: id, label, type, category, icon, group, blurb (first body paragraph), videoCount, tier (longest prerequisite chain), prereqCount, url; plus source→target edges
5. **Static site build**: `mkdocs build --strict` (fails on broken links/warnings)

### MkDocs Hooks (`scripts/hooks.py`)
Runs on every page at build time:
- **Video embeds**: `videos:`/`video:` frontmatter renders privacy-friendly `youtube-nocookie` players, at a `<!-- videos -->` marker or in an appended "Watch" section. Playlists supported
- **"See in tree" link**: every node page links back to the homepage with its node selected (`/?node=<id>`)
- **"Next in the tree" section**: successor links computed from `graph.json`, sorted by tier — the tree structure drives page-to-page navigation automatically
- All injected links carry Umami event attributes

### Tree Renderer (`docs/index.html` — the homepage)
Standalone HTML page, copied verbatim by MkDocs (listed in `not_in_nav`). `docs/tree/index.html` is just a redirect to `/` kept for old links.

- **Layout**: ELK (elkjs 0.9.3 from CDN) layered algorithm, left-to-right; `tier` values become ELK partitions ("eras"); `group` values become ELK hierarchy nodes rendered as labeled boxes (labels in `GROUP_LABELS` — add an entry when introducing a new group id)
- **Nodes**: HTML cards with Tabler icons, category color, video count; projects styled distinctly
- **Edges**: single SVG layer with orthogonal routes from ELK
- **Interaction**: custom pan/zoom, horizontal scrollbar, hover highlights prerequisite chains, click opens an info panel (blurb, prereq/unlock chips, "Read" button), `?node=<id>` deep-links
- **HUD**: brand badge, GitHub button (maintainer credit), About link, category legend, zoom controls; responsive rules for <720px
- Category colors live in `:root` CSS variables at the top of the file

### Deployment (`.github/workflows/deploy.yml`)
On every push to `main` (plus manual dispatch):
1. Setup Python 3.13 + uv (cached)
2. `uv sync`
3. `uv run python scripts/build_graph.py` — CI fails if the graph is invalid
4. `uv run python scripts/write_analytics_config.py` — writes `docs/js/analytics-config.js` from `UMAMI_SCRIPT_URL` / `UMAMI_WEBSITE_ID` / `UMAMI_DOMAINS` repo variables (empty locally; analytics simply stay off)
5. `uv run mkdocs build --strict`
6. Upload + deploy to GitHub Pages

Zero servers, zero maintenance, zero cost.

## Content Structure (As Built)

**43 nodes** (36 topics, 7 projects), **71 edges**, single root (`curiosity`), deepest path 14 tiers (Curiosity → Open Duck Mini).

By category: AI & ML 17 · Programming 10 · Electronics 8 · Mechanical 4 · Data Science 3 · Core 1.

**Grouped ecosystems** (rendered as boxes on the tree):
- `python-foundations`: python-variables, python-loops, python-classes, python-basics
- `ros-ecosystem`: ros2-basics, ros2-communication, ros2-packages-launch
- `nvidia-ecosystem`: gpu, cuda-basics, tensorrt, deepstream, simulation (Isaac Sim)
- `hardware`: power-systems, microcontrollers, sensors, motor-control
- `data-science`: data-fundamentals, statistics-modelling, ml-fundamentals, neural-networks, computer-vision
- `agentic-AI`: llms, rag, ai-agents

**7 projects**: blink-led, sensor-station, robot-arm (7-episode LeRobot series), so101, robot-car, robot-dog (quadruped), open-duck (Open Duck Mini).

### Design Notes on Dependencies
- **Granularity**: small concepts are checklists inside topic pages ("You can move on when you can..."), not individual nodes. Civilization has ~70 chunky nodes, not 500 tiny ones
- **Projects as prerequisites**: e.g. sensor-station requires the blink-led project; robot-arm requires sensor-station. Hands-on experience gates advanced work
- **Category colors**: distinct color per branch, in both the legend and node styling

## Phase 2 Upgrade Path

**Goal**: Track user progress (mark topics completed, show what's unlocked).

### Phase 2a: localStorage (still fully static)
- Checkbox on each topic page stores the topic ID in `localStorage`
- Tree page reads stored IDs and highlights completed/unlocked nodes
- Zero backend; no migration needed (same stable IDs, same graph.json)

### Phase 2b: Real accounts (minimal backend)
- Swap localStorage for a tiny hosted backend (Supabase, Pocketbase) storing "set of completed IDs per user"
- Site stays static; only progress sync talks to the backend

**Nothing built in phase 1 gets thrown away.**

## Developer Workflow

### Adding Content
1. Create `docs/topics/my-topic.md` or `docs/projects/my-project.md` (copy frontmatter from an existing file)
2. Add the page to `nav:` in `mkdocs.yml`
3. If introducing a **new** `group:` id, add a label to `GROUP_LABELS` in `docs/index.html`
4. Run `uv run python scripts/build_graph.py` (validates immediately)
5. Optional: `uv run mkdocs serve` to preview at `http://127.0.0.1:8000`
6. Push to GitHub → auto-deploy

**The tree lays itself out.** No manual node placement.

### Local Testing
```bash
uv sync                                # install dependencies
uv run python scripts/build_graph.py   # validate and write graph.json
uv run mkdocs build --strict           # what CI runs
uv run mkdocs serve                    # live preview
```

## Validation Features

`build_graph.py` catches before deployment: missing frontmatter fields, duplicate IDs, typo'd prerequisite IDs, circular dependencies, invalid node types. `mkdocs build --strict` catches broken links.

**Philosophy**: Fail loudly at build time, never ship a broken graph.

## Known Limitations & Future Considerations

- **MkDocs 2.0**: the Material team has flagged MkDocs 2.0 as a breaking rewrite with no migration path (theme overrides break, closed contribution model). `pyproject.toml` pins `mkdocs==1.6.1` and `mkdocs-material==9.7.6`. Do not upgrade without reading their analysis: https://squidfunk.github.io/mkdocs-material/blog/2026/02/18/mkdocs-2.0/
- **Manual nav ordering**: pages must be listed in `mkdocs.yml` `nav:`; chosen over auto-nav plugins for explicitness
- **CDN dependencies**: elkjs and Tabler icons load from CDNs. Vendor into `docs/js/` if offline builds are ever needed
- **Mobile**: the tree page has responsive HUD/info-panel rules and touch pan/zoom, but 43 nodes remain dense on phones
- **Search doesn't index frontmatter**: searching "requires kinematics" won't find dependents
- **Legacy file**: `docs/js/tech-tree.js` (old Cytoscape renderer) is unused and can be deleted

## Testing Strategy

- **Build-time validation is the test suite**: if `build_graph.py` writes `graph.json` and `mkdocs build --strict` passes, the site is shippable — CI runs both on every push
- Smoke test: `uv run python scripts/build_graph.py && uv run mkdocs build --strict`

## Inspiration & Design References

**Civilization VI tech tree**: left-to-right eras, category colors, chunky nodes, projects as prerequisites, group boxes as "districts" of related techs.

## File Structure Summary

```
backtoengineering/
├── .github/workflows/deploy.yml       # CI: uv sync → validate graph → analytics config → strict build → Pages
├── docs/
│   ├── index.html                     # THE tree page (homepage): ELK layout, HTML cards, custom pan/zoom, HUD
│   ├── tree/index.html                # redirect to / (kept for old links)
│   ├── about.md                       # "how this works" page
│   ├── CNAME                          # www.backtoengineering.com
│   ├── assets/graph.json              # generated by build_graph.py — do not edit by hand
│   ├── assets/branding/               # logos
│   ├── css/custom.css                 # Material theme tweaks (next-node buttons, video embeds…)
│   ├── js/analytics-config.js         # generated at deploy; empty values locally
│   ├── js/site-analytics.js           # Umami loader + custom events
│   ├── js/tech-tree.js                # LEGACY (unused Cytoscape renderer)
│   ├── topics/*.md                    # 36 topic pages with frontmatter
│   └── projects/*.md                  # 7 project pages with frontmatter
├── scripts/
│   ├── build_graph.py                 # frontmatter → graph.json with validation
│   ├── hooks.py                       # MkDocs hooks: video embeds, "Next in the tree", "See in tree"
│   └── write_analytics_config.py      # env vars → analytics-config.js
├── mkdocs.yml                         # theme, nav, repo_url/copyright credit, hooks, extras
├── pyproject.toml + uv.lock           # pinned deps (Python ≥3.13, mkdocs 1.6.1, material 9.7.6)
└── README.md                          # setup/usage guide + credits
```

## Lessons Learned

1. **SSG is interchangeable; unique features are not**: the tree page is the complexity budget, and it eventually outgrew off-the-shelf graph libraries — the custom ELK renderer earns its ~1000 lines
2. **Stable IDs are the skeleton**: titles, content, even URLs can change; IDs cannot. They're the foreign key for progress tracking, the hooks, and the graph
3. **Validate early, validate loud**: build-time validation catches content errors before CI deploys them
4. **Derive navigation from data**: "Next in the tree" links are computed from graph.json at build time — adding an edge updates every affected page automatically
5. **Future-proofing is cheap when the data model is right**: phase 2 progress tracking requires zero changes to content files

## Next Steps (Proposed)

1. **Phase 2a**: localStorage progress tracking (checkbox per topic, completed/unlocked states on the tree)
2. **Flesh out stub topics**: most topic pages are short checklists; expand explanations and link videos via `videos:` frontmatter
3. **Delete legacy `docs/js/tech-tree.js`** and drop the stale renderer mentions from README
4. **Clean up `docs/assets/`**: stray `Iulia's YouTube library_.md` files and duplicate logo copies
5. **Mobile polish**: collapsible groups or a simplified phone view for the tree

## Claude's Role in This Build

Claude generated: the original MkDocs/Cytoscape skeleton, the build/validation script, content stubs, CI workflow, the hardware topic branch, GitHub credit integration, and this documentation. The custom ELK tree renderer, analytics pipeline, video/next-node hooks, and the full content expansion (ROS 2, NVIDIA, Python foundations, agentic AI branches, 7 projects) were built on top of that skeleton.

Human provided: requirements (static, Python, single maintainer), UX reference (Civ VI), domain expertise (robotics/AI dependency structure), content, and branding.

## Build Metadata

- **Initial build**: 2026-07-12 · **Last doc update**: 2026-07-16
- **Nodes**: 43 (36 topics, 7 projects) · **Edges**: 71 · **Depth**: 14 tiers (Curiosity → Open Duck Mini)
- **Groups**: python-foundations, ros-ecosystem, nvidia-ecosystem, hardware, data-science, agentic-AI
- **Technologies**: Python 3.13, uv, MkDocs 1.6.1, Material 9.7.6, elkjs 0.9.3, Tabler Icons
- **Custom code**: `build_graph.py` ~240 lines · `hooks.py` ~120 lines · `docs/index.html` ~1000 lines (renderer + styles) · `write_analytics_config.py` + `site-analytics.js`
