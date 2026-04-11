# Vulpine Mark — Visual Element Labeling for AI Browser Agents

## What It Is

A standalone tool that annotates browser screenshots with numbered labels on interactive elements, returning both the annotated image and a structured element map. Enables AI agents to say "click @14" instead of guessing coordinates.

## Why It's Needed

AI agents using screenshots for visual grounding have 20-30% lower accuracy than agents with element-level labels (Microsoft SoM paper, 2023). Current approaches require heavy ML models (OmniParser needs 11GB VRAM, runs on A100 GPUs) or are tightly coupled to specific frameworks (WebArena, Vercel agent-browser). Nobody has shipped a fast, standalone, browser-agnostic tool.

## Competitive Landscape

| Tool | Approach | GPU Required | Standalone | Browser Support |
|------|----------|-------------|------------|-----------------|
| Microsoft SoM | SAM segmentation | Yes (heavy) | No (research demo) | Any (image-based) |
| OmniParser | YOLOv8 + Florence-2 | Yes (11GB VRAM) | Partially | Any (image-based) |
| SeeClick | 7B VLM | Yes (GPU) | No | Any (image-based) |
| WebArena SoM | JS injection | No | No (embedded in benchmark) | Chrome only |
| Anthropic CU | Raw coordinate prediction | No (cloud) | No (proprietary) | Any (screenshot) |
| **Vulpine Mark** | DOM layout tree + compositor | **No** | **Yes** | **Any via CDP/Juggler** |

## Our Unique Advantage

VulpineOS controls the browser engine. We can:
- Read the live layout tree (element positions) with zero ML overhead
- Render labels natively via the compositor (no JS injection, works on cross-origin iframes)
- Map labels directly to Juggler element handles — no coordinate guessing
- Run in <10ms (just reading existing layout + painting overlays)
- Handle dynamic content naturally (re-reads live DOM on each call)

## Architecture

```
Browser Page
     │
     ▼
Vulpine Mark Engine
├── 1. Enumerate interactive elements (buttons, links, inputs, selects)
│      via accessibility tree or DOM querySelectorAll
├── 2. Get bounding rects (getBoundingClientRect for each)
├── 3. Filter: only visible, in viewport, not obscured
├── 4. Assign labels (@1, @2, @3...)
├── 5. Render annotated screenshot:
│      - Take base screenshot
│      - Draw numbered badges at element centers
│      - Color-code by role (green=button, blue=link, purple=input)
├── 6. Return:
│      - Annotated PNG image
│      - JSON map: { "@1": {tag, role, text, x, y, w, h}, "@2": ... }
└── 7. Labels persist for subsequent click/type actions
```

## Implementation Options

### Option A: Juggler-Level (VulpineOS-native, fastest)
New Juggler method `Page.getAnnotatedScreenshot` that:
1. Reads the accessibility tree (already filtered by Phase 1)
2. Gets bounding rects for each interactive node
3. Takes a screenshot via the compositor
4. Draws labels using Gecko's canvas painting APIs
5. Returns PNG + element map in one call

**Pros:** Fastest (<10ms), works on cross-origin iframes, no JS injection.
**Cons:** Tied to Camoufox/VulpineOS. Needs Juggler JS code (not C++).

### Option B: Standalone Go Library (browser-agnostic)
A Go library that:
1. Takes any CDP connection (foxbridge, Chrome, etc.)
2. Calls `Runtime.evaluate` to get element positions
3. Takes screenshot via `Page.captureScreenshot`
4. Draws labels using Go image library
5. Returns annotated PNG + element map

**Pros:** Works with any browser. Can be a standalone repo.
**Cons:** Slightly slower (JS eval + image processing in Go). Can't handle cross-origin iframes.

### Option C: Both
Implement Option A in VulpineOS for maximum speed, and Option B as an open-source standalone library for the community. The standalone version drives adoption; the native version is the competitive advantage.

## Recommended Approach

**Build both. Open-source the standalone Go library. Keep the Juggler-native version in VulpineOS.**

The standalone `vulpine-mark` repo gets community adoption and positions VulpineOS as the ecosystem leader. The native version in VulpineOS is faster and more capable (cross-origin iframe support, no JS injection).

## Open Source: YES

**Repo:** `PopcornDev1/vulpine-mark` (public)

Reasons:
- The concept (SoM) is already published research — no proprietary secret
- Community adoption drives traffic to VulpineOS
- The Go library works with Chrome too — wider audience
- The competitive advantage is the *native* Juggler implementation, not the standalone library

## MVP Scope (1-2 weeks)

1. Go library that connects via CDP WebSocket
2. Enumerates interactive elements via `Runtime.evaluate`
3. Gets bounding rects
4. Takes screenshot
5. Draws numbered badges using Go `image/draw`
6. Returns annotated PNG + JSON element map
7. CLI tool: `vulpine-mark --cdp ws://localhost:9222 --output annotated.png`
8. MCP tool integration: `vulpine_annotated_screenshot` in VulpineOS

## API Design

```go
// Standalone library
mark := vulpinemark.New(cdpWebSocketURL)
result, err := mark.Annotate()
// result.Image - annotated PNG bytes
// result.Elements - map of label → {tag, role, text, x, y, w, h}
// result.Labels - ordered list of labels

// Click by label
mark.Click("@3") // clicks the element labeled @3
```

```bash
# CLI
vulpine-mark --cdp ws://localhost:9222 --output screenshot.png --json elements.json
```

## Integration with VulpineOS MCP

New tool `vulpine_annotated_screenshot`:
- Returns annotated image as `image` content block
- Returns element map as `text` content block
- Agent sees labeled screenshot + structured data
- Can then call `vulpine_click` with coordinates from the map

## Success Metrics

- Accuracy improvement on WebArena-style benchmarks (target: 20%+ vs non-annotated)
- Latency: <100ms for standalone, <20ms for native Juggler
- GitHub stars / adoption by other agent frameworks
