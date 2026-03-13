# Plan: Interactive "Which AI Coding Agent?" Decision Website

## Context
The repo contains a Mermaid-based decision flowchart for choosing an AI coding agent (`docs/ai-coding-agent-decision-flow.mermaid`). The goal is to transform this into an interactive website where users click through questions one-by-one to reach a personalized tool recommendation, then optionally explore the full decision tree.

**The user will provide the `flow-data.json` file directly** — we do not need to generate it from the Mermaid source. The app just needs to consume whatever JSON is placed at that path.

## Architecture

**Static site, zero build tools, zero dependencies.** Three files at the repo root:

| File | Purpose |
|------|---------|
| `index.html` | App shell, structure, minimal inline JS |
| `style.css` | All styling (modern CSS: container queries, nesting, `@layer`, `color-mix()`, scroll-driven animations, view transitions) |
| `flow-data.json` | Decision tree data — the single source of truth, designed for future LLM generation |

Why separate files instead of a single HTML: keeps the data layer (`flow-data.json`) cleanly decoupled so an LLM or API can swap it without touching markup. The site still works as plain static files — any HTTP server, Vercel, Netlify, S3, `python -m http.server`, etc.

## Data Schema (`flow-data.json`)

```jsonc
{
  "meta": {
    "title": "Which AI Coding Agent?",
    "description": "Find the right AI coding tool for your workflow",
    "version": "1.0",
    "lastUpdated": "2026-03-13"
  },
  "nodes": {
    "START": {
      "type": "question",
      "text": "Need deep reasoning?",
      "subtitle": "Multi-file refactoring, edge cases",
      "options": [
        { "label": "Yes, complex refactoring & edge cases", "next": "DR_PRIV" },
        { "label": "No, something else", "next": "Q2" }
      ]
    },
    // ... all question nodes
    "CLAUDE": {
      "type": "result",
      "name": "Claude Code",
      "category": "closed",        // "oss" | "closed" | "commercial"
      "icon": "🔵",
      "tagline": "80.9% SWE-bench · cloud",
      "description": "Top-tier reasoning for complex refactoring tasks...",
      "tags": ["Paid (API)", "Anthropic", "200k context", "Terminal"],
      "features": ["🏆 80.9% SWE-bench", "☁️ Cloud", "🔌 MCP + Agents"]
    }
    // ... all result nodes
  }
}
```

Every question node has `options[]` with `label` + `next` (node ID). Every result node has rich metadata. This is the only file an LLM needs to update.

**Note:** The user will supply this file. The app should document the expected schema (in a comment or this plan) but not generate the data itself.

## HTML Structure (`index.html`)

```
<body>
  <header>  — title, tagline
  <main>
    <section id="quiz">        — the interactive card area
      <div class="progress">   — breadcrumb trail of answers so far
      <div class="card">       — current question or result
    </section>
    <section id="explore">     — full decision tree (hidden until user opts in)
      <div class="tree">       — CSS-based interactive flow diagram
    </section>
  </main>
  <footer>  — legend, attribution
</body>
```

## Interactive Flow (Minimal JS)

JS is needed for exactly 3 things that CSS alone cannot do:
1. **Fetch and parse** `flow-data.json`
2. **Navigate** between nodes (update DOM with next question/result based on user click)
3. **Build the explore-all-paths tree** from the JSON data

This will be ~80-100 lines of vanilla JS, inline in `index.html`. No framework, no module bundler.

### Flow:
1. On load: fetch `flow-data.json`, render first question as a card
2. User clicks an option → animate card transition (CSS), render next node
3. If next node is a `result` → show the recommendation card with full details + "Start Over" and "Explore All Paths" buttons
4. "Explore All Paths" → renders the full tree from JSON as a CSS-styled nested structure with the user's taken path highlighted

## CSS Approach (`style.css`)

### Design Language: "Funky but Accessible"
- **Dark base** (#0f1117) with vibrant neon accents — green (#7ED321), blue (#6ab0de), orange (#F5C563)
- **Large, chunky option buttons** with hover/focus glow effects (box-shadow with `color-mix()`)
- **Card-based UI** with smooth slide/fade transitions using CSS `@starting-style` + `transition-behavior: allow-discrete`
- **Gradient borders** on cards using `border-image` or pseudo-element technique
- **Playful but readable typography**: system font stack, generous sizing (clamp-based fluid type)
- **Progress breadcrumb** showing path taken, with colored dots matching category

### Mobile-First Responsive
- Single column layout, full-width cards
- Options stacked vertically with large touch targets (min 48px)
- The "explore" tree view uses horizontal scroll with snap points on mobile
- The tree diagram uses CSS `details/summary` for collapsible branches — no JS needed

### Accessibility
- All interactive elements are `<button>` elements (keyboard navigable)
- Focus-visible outlines with the accent color
- `prefers-reduced-motion` media query to disable animations
- `prefers-color-scheme: light` support (light mode variant)
- Sufficient color contrast (WCAG AA) — text on dark backgrounds verified
- `aria-live` region on the card area so screen readers announce new questions
- Semantic HTML throughout

## The "Explore All Paths" Diagram

Instead of Mermaid (heavy, not mobile-friendly), build a **CSS tree** using nested `<details>` elements:

```html
<details open>
  <summary>Need deep reasoning?</summary>
  <div class="branch">
    <details>
      <summary>Yes → Data privacy critical?</summary>
      <div class="branch">
        <details>
          <summary>Yes → Full offline needed?</summary>
          ...
        </details>
      </div>
    </details>
    <details>
      <summary>No → Need autonomous execution?</summary>
      ...
    </details>
  </div>
</details>
```

- Generated from `flow-data.json` by JS
- Each `<summary>` styled as a question card, each leaf styled as a result card
- Tree lines drawn with CSS `border-left` + `::before` pseudo-elements
- The user's own path gets a highlight class (glowing border)
- Fully collapsible/expandable — works great on mobile
- `<details>` is native HTML — no JS for open/close behavior

## Files to Create

1. **`/home/user/which-agent/index.html`** — app shell with inline `<script>` (~80-100 lines JS)
2. **`/home/user/which-agent/style.css`** — all styling

The user will provide **`/home/user/which-agent/flow-data.json`** themselves, following the schema documented above.

Existing `docs/` files are left untouched (they're source reference material).

## Implementation Order

1. Create `style.css` — full styling with responsive design, animations, dark/light themes
2. Create `index.html` — HTML structure + minimal JS for navigation and tree rendering (consumes whatever `flow-data.json` is present)
3. Test & verify — open in browser with user-provided JSON, check mobile responsiveness, keyboard nav, all paths work

## Verification

- Open `index.html` via any static server and click through every path to verify all branches lead to correct results
- Resize browser to mobile widths to verify responsive layout
- Tab through with keyboard to verify focus management
- Check that "Explore All Paths" renders the complete tree with the user's path highlighted
- Verify any valid `flow-data.json` following the schema is picked up on reload
