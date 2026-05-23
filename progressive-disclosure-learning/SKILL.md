---
name: progressive-disclosure-learning
description: >
  Explains a topic, article, paper, website, or repository through progressive
  disclosure — layered teaching with high-quality infographic-style diagrams,
  like a picture book for adult learners. Trigger when the user asks to
  "explain", "teach", "walk me through", "break down", or "help me understand"
  a concept, paper, codebase, or URL. Also trigger for /learn, /explain, /teach.
---

# Progressive Disclosure Learning Skill

Teach any topic through layered explanation and high-quality generated visuals.
The goal is clarity, not simplification — concepts stay accurate and complete,
but the hardest ideas are introduced only after their foundations are established.

## Guiding Principles

- **Progressive disclosure**: Each layer builds on the last. Never introduce a
  term before defining it. Never show a full system before its parts.
- **One idea per beat**: Each section focuses on exactly one concept or
  mechanism. Don't cram.
- **Visuals first, prose second**: Lead with a generated diagram whenever
  possible. Explain it in prose afterward.
- **Calibrate to the user**: Use the user's stated expertise level, or infer it
  from their phrasing. Default to "no domain expertise" if uncertain.
- **Accuracy over approachability**: Never distort a concept for simplicity.
  If a concept is hard, say so and scaffold it carefully.
- **Complete the picture**: By the end, the user should understand the full
  topic — not just a simplified summary.

---

## Visual Generation Stack

Generate all diagrams as PNG files and send them with `SendUserFile`. Use the
right tool for each diagram type:

| Diagram type                                             | Tool           |
|----------------------------------------------------------|----------------|
| Concept maps, flow diagrams, DAGs, system architecture   | **graphviz**   |
| Data plots, curves, charts, numeric relationships        | **matplotlib** |
| Annotated step-by-step diagrams, before/after visuals    | **matplotlib** |
| Summary cards / reference sheets                         | **matplotlib** |

ASCII art is a last resort only when code execution is unavailable.

---

## Visual Style System

All diagrams must follow this unified style. This is non-negotiable — every
diagram in a session should feel like it came from the same publication.

The target aesthetic is high-quality data journalism (think Visual Capitalist,
Information is Beautiful): dark background, strong typographic hierarchy,
vivid accent colors, clean layout with generous spacing, infographic-style
callout badges for key insights.

### Install and register fonts (run once per session)

```python
import matplotlib
matplotlib.use('Agg')
import matplotlib.pyplot as plt
import matplotlib.patches as mpatches
import matplotlib.gridspec as gridspec
import matplotlib.font_manager as fm
import matplotlib.patheffects as pe
import numpy as np
import graphviz
import os

# Register Open Sans if available (preferred), else fall back gracefully
_FONT_DIR = '/usr/share/fonts/truetype/open-sans/'
if os.path.isdir(_FONT_DIR):
    for _f in os.listdir(_FONT_DIR):
        if _f.endswith('.ttf'):
            fm.fontManager.addfont(os.path.join(_FONT_DIR, _f))
    SANS = 'Open Sans'
else:
    SANS = 'DejaVu Sans'

OUTPUT_DIR = '/tmp/learn_diagrams'
os.makedirs(OUTPUT_DIR, exist_ok=True)
```

### Color palette

```python
# ── Backgrounds ────────────────────────────────────────────────────────────
BG      = '#0B1120'   # full-page background
CARD    = '#131D30'   # primary card / panel background
CARD2   = '#0F1929'   # alternate / darker card
BORDER  = '#1E2D45'   # subtle card border

# ── Accent colors ──────────────────────────────────────────────────────────
CYAN    = '#38BDF8'   # primary accent (titles, key data)
AMBER   = '#FBBF24'   # secondary accent (supporting data)
GREEN   = '#34D399'   # positive / correct / result
RED     = '#F87171'   # negative / warning / before-state
PURPLE  = '#A78BFA'   # tertiary accent (formulas, mechanisms)
TEAL    = '#2DD4BF'   # quaternary (alternative)

# ── Text ───────────────────────────────────────────────────────────────────
TXT_PRI = '#F1F5F9'   # primary body text
TXT_SEC = '#94A3B8'   # secondary / captions
TXT_DIM = '#475569'   # axis ticks, watermarks
```

### Global rcParams (set once at the top of every script)

```python
plt.rcParams.update({
    'figure.facecolor':      BG,
    'axes.facecolor':        CARD,
    'axes.edgecolor':        BORDER,
    'axes.labelcolor':       TXT_SEC,
    'axes.titlecolor':       TXT_PRI,
    'xtick.color':           TXT_DIM,
    'ytick.color':           TXT_DIM,
    'text.color':            TXT_PRI,
    'grid.color':            BORDER,
    'grid.linewidth':        0.6,
    'axes.spines.top':       False,
    'axes.spines.right':     False,
    'axes.spines.left':      True,
    'axes.spines.bottom':    True,
    'font.family':           SANS,
    'figure.dpi':            150,
})
```

---

## Helper Functions (copy into every script that needs them)

### `page_header(fig, title, subtitle)`

Draws a bold title + subtitle + decorative accent line at the top of any figure.
Call this after creating the figure, before adding axes.

```python
def page_header(fig, title, subtitle='', source=''):
    fig.text(0.5, 0.965, title,
             ha='center', va='top', fontsize=22, fontweight='bold',
             color=TXT_PRI, fontfamily=SANS)
    if subtitle:
        fig.text(0.5, 0.928, subtitle,
                 ha='center', va='top', fontsize=11, color=TXT_SEC,
                 fontfamily=SANS)
    line = plt.Line2D([0.15, 0.85], [0.912, 0.912],
                      transform=fig.transFigure,
                      color=CYAN, linewidth=1.5, alpha=0.7)
    fig.add_artist(line)
    if source:
        fig.text(0.5, 0.022, source,
                 ha='center', fontsize=7.5, color=TXT_DIM, fontfamily=SANS)
```

### `card_header(ax, label, color, subtitle='')`

Draws a tinted header band + bold label at the top of a panel/card axes.

```python
def card_header(ax, label, color=CYAN, subtitle=''):
    ax.axhspan(0.875, 1.0, color=color, alpha=0.13, zorder=0,
               transform=ax.transAxes)
    ax.text(0.5, 0.953, label,
            ha='center', va='center', fontsize=12, fontweight='bold',
            color=color, fontfamily=SANS, transform=ax.transAxes)
    if subtitle:
        ax.text(0.5, 0.896, subtitle,
                ha='center', va='center', fontsize=8.5, color=TXT_SEC,
                fontfamily=SANS, transform=ax.transAxes)
```

### `badge(ax, x, y, w, h, headline, subline, bg, border_color)`

Creates an inset callout badge — the Visual Capitalist "stat box" element.
Use for key insights, statistics, or "so what" moments.

```python
def badge(ax, x, y, w, h, headline, subline='', bg='#0D1F30', border_color=CYAN):
    b = ax.inset_axes([x, y, w, h])
    b.set_facecolor(bg)
    b.set_xlim(0, 1); b.set_ylim(0, 1); b.axis('off')
    for sp in b.spines.values():
        sp.set_edgecolor(border_color); sp.set_linewidth(1.6)
    b.text(0.5, 0.64 if subline else 0.5, headline,
           ha='center', va='center', fontsize=9.5, fontweight='bold',
           color=border_color, fontfamily=SANS)
    if subline:
        b.text(0.5, 0.22, subline,
               ha='center', va='center', fontsize=7.5,
               color=TXT_SEC, fontfamily=SANS)
    return b
```

### `style_axes(ax)`

Applies the standard dark-theme spine and tick styling to any axes.

```python
def style_axes(ax):
    ax.set_facecolor(CARD)
    for spine in ax.spines.values():
        spine.set_edgecolor(BORDER)
        spine.set_linewidth(1.0)
    ax.tick_params(colors=TXT_DIM, labelsize=9.5)
    ax.xaxis.label.set_color(TXT_SEC)
    ax.yaxis.label.set_color(TXT_SEC)
```

### `section_divider(ax, y, label, color=CYAN)`

Draws a thin horizontal rule with a centred label — used to separate sections
within a tall single-panel diagram.

```python
def section_divider(ax, y, label, color=CYAN):
    ax.axhline(y, color=BORDER, linewidth=0.8, zorder=1)
    ax.text(0.5, y + 0.01, label,
            ha='center', va='bottom', fontsize=8, color=color,
            fontfamily=SANS, transform=ax.transAxes)
```

---

## Diagram Patterns

### Pattern 1 — Graphviz: Concept Map / Flow Diagram

Use for: topic overview maps, system architecture, data flow, dependency trees.

The graphviz stylesheet mirrors the matplotlib dark theme.

```python
def make_graph(directed=True, title='', rankdir='TB'):
    G = graphviz.Digraph if directed else graphviz.Graph
    g = G(
        graph_attr={
            'label':     title,
            'labelloc':  't',
            'fontsize':  '16',
            'fontname':  'Open Sans Bold',
            'fontcolor': '#F1F5F9',
            'bgcolor':   '#0B1120',
            'rankdir':   rankdir,
            'splines':   'curved',
            'nodesep':   '0.7',
            'ranksep':   '1.0',
            'pad':       '0.5',
        },
        node_attr={
            'style':     'filled,rounded',
            'shape':     'box',
            'fillcolor': '#131D30',
            'color':     '#1E2D45',
            'fontcolor': '#F1F5F9',
            'fontname':  'Open Sans',
            'fontsize':  '11',
            'margin':    '0.2,0.12',
        },
        edge_attr={
            'color':     '#38BDF8',
            'fontcolor': '#94A3B8',
            'fontname':  'Open Sans',
            'fontsize':  '9',
            'penwidth':  '1.4',
            'arrowsize': '0.8',
        }
    )
    return g

# Node color presets — use these for semantic meaning:
NODE_PRIMARY   = {'fillcolor': '#0F2947', 'color': '#38BDF8', 'fontcolor': '#38BDF8'}
NODE_SECONDARY = {'fillcolor': '#2A1D0A', 'color': '#FBBF24', 'fontcolor': '#FBBF24'}
NODE_POSITIVE  = {'fillcolor': '#0A2A1E', 'color': '#34D399', 'fontcolor': '#34D399'}
NODE_NEGATIVE  = {'fillcolor': '#2A0F0F', 'color': '#F87171', 'fontcolor': '#F87171'}
NODE_NEUTRAL   = {'fillcolor': '#1A1F2E', 'color': '#94A3B8', 'fontcolor': '#94A3B8'}
NODE_PURPLE    = {'fillcolor': '#1A0F2A', 'color': '#A78BFA', 'fontcolor': '#A78BFA'}
EDGE_AMBER     = {'color': '#FBBF24', 'fontcolor': '#FBBF24'}
EDGE_GREEN     = {'color': '#34D399', 'fontcolor': '#34D399'}
EDGE_DIM       = {'color': '#475569', 'fontcolor': '#475569', 'style': 'dashed'}
```

### Pattern 2 — Matplotlib: Line / Curve Plot

Use for: loss landscapes, training curves, distributions, numeric relationships.

```python
def line_plot(title, subtitle, xlabel, ylabel, series, annotations, filename, source=''):
    """
    series: list of dicts with keys:
        x, y           — data arrays
        label          — legend label
        color          — line color from palette
        linewidth      — default 2.5
        linestyle      — default '-'
        fill           — True/False, shade under curve
    annotations: list of dicts with keys:
        x, y           — arrow tip position
        dx, dy         — text offset from tip
        text           — annotation string
        color          — annotation color
        connectionstyle — e.g. 'arc3,rad=0.2' (optional)
    """
    fig, ax = plt.subplots(figsize=(11, 6))
    page_header(fig, title, subtitle, source)
    style_axes(ax)

    for s in series:
        lw = s.get('linewidth', 2.5)
        ls = s.get('linestyle', '-')
        line, = ax.plot(s['x'], s['y'], color=s['color'], linewidth=lw,
                        linestyle=ls, label=s.get('label', ''), zorder=4)
        if s.get('fill'):
            ax.fill_between(s['x'], s['y'], alpha=0.08, color=s['color'])

    for ann in annotations:
        cs = ann.get('connectionstyle', 'arc3,rad=0.15')
        ax.annotate(
            ann['text'],
            xy=(ann['x'], ann['y']),
            xytext=(ann['x'] + ann['dx'], ann['y'] + ann['dy']),
            fontsize=9, color=ann['color'], fontfamily=SANS,
            arrowprops=dict(arrowstyle='->', color=ann['color'],
                            lw=1.3, connectionstyle=cs),
        )

    ax.set_xlabel(xlabel, fontsize=10)
    ax.set_ylabel(ylabel, fontsize=10)
    if any(s.get('label') for s in series):
        ax.legend(facecolor=CARD2, edgecolor=BORDER, labelcolor=TXT_PRI,
                  fontsize=9)
    ax.grid(True, alpha=0.4)

    plt.tight_layout(rect=[0, 0.04, 1, 0.88])
    path = f'{OUTPUT_DIR}/{filename}'
    fig.savefig(path, dpi=150, bbox_inches='tight', facecolor=BG)
    plt.close(fig)
    return path
```

### Pattern 3 — Matplotlib: Side-by-Side Step Panels

Use for: showing a mechanism evolving across multiple states — one update of
an algorithm, before/after comparisons, A-vs-B contrasts.

Each panel is a full mini-diagram with its own header band, drawn by a
callable you provide.

```python
def step_panels(title, subtitle, steps, filename, source=''):
    """
    steps: list of dicts:
        label   — panel header text
        color   — panel header accent color
        draw    — callable(ax) that draws onto the axes
    """
    n = len(steps)
    fig = plt.figure(figsize=(5.5 * n, 6.5))
    page_header(fig, title, subtitle, source)

    gs = gridspec.GridSpec(1, n, figure=fig,
                           left=0.04, right=0.96,
                           top=0.82, bottom=0.10,
                           wspace=0.06)
    for i, step in enumerate(steps):
        ax = fig.add_subplot(gs[i])
        style_axes(ax)
        card_header(ax, step['label'], step.get('color', CYAN))
        step['draw'](ax)

    path = f'{OUTPUT_DIR}/{filename}'
    fig.savefig(path, dpi=150, bbox_inches='tight', facecolor=BG)
    plt.close(fig)
    return path
```

### Pattern 4 — Matplotlib: Summary / Reference Card

Use for the closing card of every topic. This card should be saveable as a
standalone reference. Dark background, strong typographic hierarchy,
color-coded bullet points.

```python
def summary_card(title, sections, filename, source=''):
    """
    sections: list of dicts:
        heading — section title string
        color   — accent color for heading
        bg      — card background (optional, default CARD2)
        items   — list of strings; prefix '+' GREEN, '-' RED, '>' PURPLE,
                  '=' AMBER, '#' CYAN; no prefix = TXT_PRI
              Use '->' in text for arrows (avoids Open Sans missing-glyph warning)
    """
    n = len(sections)
    fig = plt.figure(figsize=(3.8 * n, 8))
    fig.patch.set_facecolor(BG)
    fig.text(0.5, 0.975, title,
             ha='center', va='top', fontsize=18, fontweight='bold',
             color=TXT_PRI, fontfamily=SANS)
    line = plt.Line2D([0.1, 0.9], [0.958, 0.958],
                      transform=fig.transFigure,
                      color=CYAN, linewidth=1.5, alpha=0.6)
    fig.add_artist(line)
    if source:
        fig.text(0.5, 0.012, source, ha='center',
                 fontsize=7.5, color=TXT_DIM, fontfamily=SANS)

    gs = gridspec.GridSpec(1, n, figure=fig,
                           left=0.02, right=0.98,
                           top=0.935, bottom=0.05,
                           wspace=0.04)

    PREFIX_COLORS = {'+': GREEN, '-': RED, '>': PURPLE, '=': AMBER, '#': CYAN}

    for i, sec in enumerate(sections):
        ax = fig.add_subplot(gs[i])
        ax.set_facecolor(sec.get('bg', CARD2))
        ax.axis('off')
        for sp in ax.spines.values():
            sp.set_edgecolor(BORDER); sp.set_linewidth(1.2)

        acc = sec.get('color', CYAN)
        # Heading band
        ax.axhspan(0.935, 1.0, color=acc, alpha=0.15,
                   transform=ax.transAxes, zorder=0)
        ax.text(0.5, 0.967, sec['heading'],
                ha='center', va='center',
                fontsize=11, fontweight='bold', color=acc,
                fontfamily=SANS, transform=ax.transAxes)
        # Divider
        ax.plot([0.05, 0.95], [0.928, 0.928], color=BORDER,
                linewidth=0.8, transform=ax.transAxes)

        for j, item in enumerate(sec['items']):
            prefix = item[0] if item and item[0] in PREFIX_COLORS else None
            color  = PREFIX_COLORS.get(prefix, TXT_PRI)
            ax.text(0.06, 0.905 - j * 0.078, item,
                    ha='left', va='top', fontsize=9, color=color,
                    fontfamily=SANS, transform=ax.transAxes,
                    wrap=True)

    path = f'{OUTPUT_DIR}/{filename}'
    fig.savefig(path, dpi=150, bbox_inches='tight', facecolor=BG)
    plt.close(fig)
    return path
```

---

## Teaching Structure

### Step 0 — Understand the Input

Before teaching, determine what you're working with:

- **Topic/concept**: Teach from first principles.
- **URL / website / article**: Fetch the content, extract key ideas, teach those.
- **Research paper**: Fetch or recall; identify thesis, method, results,
  contribution. Teach in order: problem → prior art → new idea → mechanism
  → results → limitations.
- **Repository**: Read README and key source files; identify what the project
  does, its architecture, data flow. Teach in order: problem → architecture →
  key abstractions → data flow → how to use → how to extend.

Ask one question about expertise level if not stated
("Any background in X, or starting fresh?"). Default to no domain expertise.

### Step 1 — The Big Picture

Generate a graphviz diagram answering: What is this? Why does it matter?
What problem does it solve? Keep it to 3–5 nodes. No jargon.

Send it. Write 2–3 sentences of prose. Ask user if they want to continue
or adjust depth before proceeding.

### Step 2 — The Conceptual Map

Generate a complete graphviz map of all key concepts and their relationships.
Label edges with the relationship ("feeds into", "requires", "produces").

Send it. List the sections as a numbered agenda.

### Step 3 — Layer-by-Layer Teaching

For each concept in the map, one section at a time:

1. **The Intuition** — a real-world analogy using nothing the user doesn't
   already know. No jargon.
2. **The Diagram** — generate a targeted visual for this concept only.
   Choose: graphviz for structure, line plot for numeric relationships,
   step panels for mechanisms, horizontal bar chart for comparisons.
   Use `badge()` for the single most important insight in each diagram.
3. **The Precise Explanation** — accurate, complete, full terminology. Equations
   shown as readable strings with every symbol explained.
4. **The "Why This Works" Beat** — what insight does this encode? What breaks
   if you remove it?
5. **A Concrete Example** — one small, numeric, end-to-end example.
6. **Common Misconceptions** — 1–2 explicit "this does NOT mean X, it means Y".

End each section with a one-sentence transition to the next.

### Step 4 — The Full System Diagram

Reassemble all parts into one graphviz diagram annotated with mechanics and
data flow. Every edge should be labeled.

### Step 5 — Real-World Grounding

2–3 real applications with honest limitations.
Use a horizontal bar or grouped bar chart when comparing results numerically.

### Step 6 — The Summary Card

Generate a `summary_card()` with 4–5 sections. Make it dense enough that a
user could screenshot it as a standalone reference. Standard sections:

- **Core Idea** — one precise sentence + mechanism sequence
- **Key Concepts** — tight definitions, one per bullet
- **The Formula / Mechanism** — equations or step-by-step (use `=` prefix)
- **Tradeoffs** — `+` for strengths, `-` for limitations
- **Where Next** — `>` prefix for follow-on topics (renders purple)

---

## Example: Gradient Descent (No ML Background)

Below is the complete diagram code for the gradient descent example.
Use it as a reference for style and quality level.

### Setup (paste at top of every script in this session)

```python
import matplotlib
matplotlib.use('Agg')
import matplotlib.pyplot as plt
import matplotlib.patches as mpatches
import matplotlib.gridspec as gridspec
import matplotlib.font_manager as fm
import numpy as np
import graphviz, os

_FONT_DIR = '/usr/share/fonts/truetype/open-sans/'
if os.path.isdir(_FONT_DIR):
    for _f in os.listdir(_FONT_DIR):
        if _f.endswith('.ttf'):
            fm.fontManager.addfont(os.path.join(_FONT_DIR, _f))
    SANS = 'Open Sans'
else:
    SANS = 'DejaVu Sans'

OUTPUT_DIR = '/tmp/learn_diagrams'
os.makedirs(OUTPUT_DIR, exist_ok=True)

BG='#0B1120'; CARD='#131D30'; CARD2='#0F1929'; BORDER='#1E2D45'
CYAN='#38BDF8'; AMBER='#FBBF24'; GREEN='#34D399'; RED='#F87171'
PURPLE='#A78BFA'; TXT_PRI='#F1F5F9'; TXT_SEC='#94A3B8'; TXT_DIM='#475569'

plt.rcParams.update({
    'figure.facecolor': BG, 'axes.facecolor': CARD,
    'axes.edgecolor': BORDER, 'axes.labelcolor': TXT_SEC,
    'xtick.color': TXT_DIM, 'ytick.color': TXT_DIM,
    'text.color': TXT_PRI, 'grid.color': BORDER, 'grid.linewidth': 0.6,
    'axes.spines.top': False, 'axes.spines.right': False,
    'font.family': SANS,
})

def page_header(fig, title, subtitle='', source=''):
    fig.text(0.5, 0.965, title, ha='center', va='top', fontsize=22,
             fontweight='bold', color=TXT_PRI, fontfamily=SANS)
    if subtitle:
        fig.text(0.5, 0.928, subtitle, ha='center', va='top', fontsize=11,
                 color=TXT_SEC, fontfamily=SANS)
    fig.add_artist(plt.Line2D([0.15, 0.85], [0.912, 0.912],
                   transform=fig.transFigure, color=CYAN, linewidth=1.5, alpha=0.7))
    if source:
        fig.text(0.5, 0.022, source, ha='center', fontsize=7.5,
                 color=TXT_DIM, fontfamily=SANS)

def card_header(ax, label, color=CYAN, subtitle=''):
    ax.axhspan(0.875, 1.0, color=color, alpha=0.13, zorder=0,
               transform=ax.transAxes)
    ax.text(0.5, 0.953, label, ha='center', va='center', fontsize=12,
            fontweight='bold', color=color, fontfamily=SANS,
            transform=ax.transAxes)
    if subtitle:
        ax.text(0.5, 0.896, subtitle, ha='center', va='center', fontsize=8.5,
                color=TXT_SEC, fontfamily=SANS, transform=ax.transAxes)

def badge(ax, x, y, w, h, headline, subline='', bg='#0D1F30', color=CYAN):
    b = ax.inset_axes([x, y, w, h])
    b.set_facecolor(bg); b.set_xlim(0,1); b.set_ylim(0,1); b.axis('off')
    for sp in b.spines.values():
        sp.set_edgecolor(color); sp.set_linewidth(1.6)
    b.text(0.5, 0.64 if subline else 0.5, headline, ha='center', va='center',
           fontsize=9.5, fontweight='bold', color=color, fontfamily=SANS)
    if subline:
        b.text(0.5, 0.22, subline, ha='center', va='center', fontsize=7.5,
               color=TXT_SEC, fontfamily=SANS)
    return b

def style_axes(ax):
    ax.set_facecolor(CARD)
    for sp in ax.spines.values():
        sp.set_edgecolor(BORDER); sp.set_linewidth(1.0)
    ax.tick_params(colors=TXT_DIM, labelsize=9.5)
```

### Diagram 1 — Big Picture (graphviz)

```python
g = graphviz.Digraph(graph_attr={
    'label': 'Gradient Descent — Big Picture', 'labelloc': 't',
    'fontsize': '18', 'fontname': 'Open Sans Bold', 'fontcolor': '#F1F5F9',
    'bgcolor': '#0B1120', 'rankdir': 'LR', 'splines': 'curved',
    'nodesep': '1.0', 'ranksep': '1.4', 'pad': '0.5',
}, node_attr={
    'style': 'filled,rounded', 'fontname': 'Open Sans', 'fontsize': '12',
    'margin': '0.25,0.15',
}, edge_attr={
    'fontname': 'Open Sans', 'fontsize': '9', 'penwidth': '1.6',
    'arrowsize': '0.8',
})

g.node('problem', 'Model makes\nbad predictions',
       fillcolor='#2A0F0F', color='#F87171', fontcolor='#F87171')
g.node('gd', 'Gradient\nDescent',
       fillcolor='#0F2947', color='#38BDF8', fontcolor='#38BDF8',
       fontsize='14', shape='ellipse')
g.node('result', 'Model learns\nto predict well',
       fillcolor='#0A2A1E', color='#34D399', fontcolor='#34D399')

g.edge('problem', 'gd', color='#475569', label='solved by')
g.edge('gd', 'result', color='#34D399', label='produces')
g.render(f'{OUTPUT_DIR}/01_big_picture', format='png', cleanup=True)
```

### Diagram 2 — Concept Map (graphviz)

```python
g = graphviz.Digraph(graph_attr={
    'label': 'Gradient Descent — Concept Map', 'labelloc': 't',
    'fontsize': '16', 'fontname': 'Open Sans Bold', 'fontcolor': '#F1F5F9',
    'bgcolor': '#0B1120', 'rankdir': 'TB',
    'nodesep': '0.75', 'ranksep': '1.0', 'pad': '0.5',
}, node_attr={
    'style': 'filled,rounded', 'fontname': 'Open Sans', 'fontsize': '11',
    'margin': '0.2,0.12',
}, edge_attr={
    'fontname': 'Open Sans', 'fontsize': '9',
    'penwidth': '1.4', 'arrowsize': '0.8',
})

g.node('gd',     'Gradient Descent',      fillcolor='#0F2947', color='#38BDF8', fontcolor='#38BDF8', shape='ellipse', fontsize='13')
g.node('loss',   'Loss Function',          fillcolor='#2A1D0A', color='#FBBF24', fontcolor='#FBBF24')
g.node('grad',   'Gradient  ∂L/∂w',       fillcolor='#1A0F2A', color='#A78BFA', fontcolor='#A78BFA')
g.node('lr',     'Learning Rate  α',       fillcolor='#0F2040', color='#38BDF8', fontcolor='#94A3B8')
g.node('update', 'Weight Update Rule',     fillcolor='#0A2A1E', color='#34D399', fontcolor='#34D399')
g.node('loop',   'Training Loop',          fillcolor='#0A1F10', color='#34D399', fontcolor='#34D399', shape='ellipse')

g.edge('gd',     'loss',   color='#FBBF24', label='measures error via')
g.edge('gd',     'grad',   color='#A78BFA', label='finds direction via')
g.edge('gd',     'lr',     color='#475569', label='controls step size via')
g.edge('loss',   'update', color='#475569')
g.edge('grad',   'update', color='#475569')
g.edge('lr',     'update', color='#475569')
g.edge('update', 'loop',   color='#34D399', label='repeated in')
g.render(f'{OUTPUT_DIR}/02_concept_map', format='png', cleanup=True)
```

### Diagram 3 — Loss Landscape (line plot)

```python
fig, ax = plt.subplots(figsize=(11, 6))
page_header(fig, 'The Loss Landscape',
            'Our terrain to navigate — gradient descent finds the lowest valley',
            'Illustrative — L(w) = w² + 0.3·sin(5w)')
style_axes(ax)

w = np.linspace(-3.2, 3.2, 400)
loss = w**2 + 0.3 * np.sin(5 * w)
ax.plot(w, loss, color=CYAN, linewidth=2.5, zorder=4)
ax.fill_between(w, loss, alpha=0.07, color=CYAN)

min_i = np.argmin(loss)
ax.scatter([w[min_i]], [loss[min_i]], s=100, color=GREEN, zorder=6)
ax.annotate('Global minimum\n(best model weights)',
            xy=(w[min_i], loss[min_i]), xytext=(w[min_i]+0.9, loss[min_i]+1.8),
            fontsize=9, color=GREEN, fontfamily=SANS,
            arrowprops=dict(arrowstyle='->', color=GREEN, lw=1.3,
                            connectionstyle='arc3,rad=-0.2'))

ax.scatter([2.2], [2.2**2 + 0.3*np.sin(11)], s=100, color=AMBER, zorder=6)
ax.annotate('Where the model\nstarts (random)',
            xy=(2.2, 2.2**2 + 0.3*np.sin(11)), xytext=(1.0, 7.0),
            fontsize=9, color=AMBER, fontfamily=SANS,
            arrowprops=dict(arrowstyle='->', color=AMBER, lw=1.3,
                            connectionstyle='arc3,rad=0.3'))

badge(ax, 0.02, 0.06, 0.32, 0.15,
      'Each point = a different model',
      'We want the lowest loss point',
      bg='#0D1829', color=CYAN)

ax.set_xlabel('Weight value  w', fontsize=10)
ax.set_ylabel('Loss  L(w)  — model wrongness', fontsize=10)
ax.grid(True, alpha=0.3)
plt.tight_layout(rect=[0, 0.04, 1, 0.88])
fig.savefig(f'{OUTPUT_DIR}/03_loss_landscape.png', dpi=150, bbox_inches='tight', facecolor=BG)
plt.close(fig)
```

### Diagram 4 — One Gradient Descent Step (step panels)

```python
fig = plt.figure(figsize=(16, 6.5))
page_header(fig, 'One Step of Gradient Descent',
            'Repeat this thousands of times and the model converges')
gs = gridspec.GridSpec(1, 3, figure=fig,
                       left=0.04, right=0.96, top=0.82, bottom=0.12, wspace=0.07)

w0, alpha = 2.2, 0.3
grad0 = 2 * w0
w1 = w0 - alpha * grad0
w_arr = np.linspace(-2.5, 3.0, 300)
loss_fn = lambda w: w**2

panels = [
    ('1   CURRENT POSITION', AMBER),
    ('2   COMPUTE GRADIENT',  PURPLE),
    ('3   UPDATE WEIGHTS',    GREEN),
]
for i, (label, col) in enumerate(panels):
    ax = fig.add_subplot(gs[i])
    style_axes(ax)
    card_header(ax, label, col)
    ax.plot(w_arr, loss_fn(w_arr), color=CYAN, linewidth=2.2, zorder=3)
    ax.set_xlabel('w', fontsize=10); ax.set_ylabel('L(w)', fontsize=10)
    ax.set_xlim(-2.5, 3.0); ax.set_ylim(-0.3, 8.5)
    ax.set_facecolor(CARD)

    if i == 0:
        ax.scatter([w0], [loss_fn(w0)], s=100, color=AMBER, zorder=6)
        ax.annotate(f'w = {w0}\nL = {loss_fn(w0):.2f}',
                    xy=(w0, loss_fn(w0)), xytext=(w0-1.8, loss_fn(w0)+1.0),
                    fontsize=9, color=AMBER, fontfamily=SANS,
                    arrowprops=dict(arrowstyle='->', color=AMBER, lw=1.2))

    elif i == 1:
        w_tan = np.linspace(w0-0.9, w0+0.9, 20)
        ax.plot(w_tan, loss_fn(w0) + grad0*(w_tan-w0),
                color=PURPLE, linewidth=2.0, linestyle='--', zorder=4)
        ax.scatter([w0], [loss_fn(w0)], s=100, color=AMBER, zorder=6)
        ax.annotate(f'∂L/∂w = {grad0:.1f}\n(slope at w = {w0})',
                    xy=(w0, loss_fn(w0)), xytext=(w0-2.2, loss_fn(w0)-0.5),
                    fontsize=9, color=PURPLE, fontfamily=SANS,
                    arrowprops=dict(arrowstyle='->', color=PURPLE, lw=1.2))
        badge(ax, 0.50, 0.10, 0.46, 0.20,
              'Gradient > 0',
              '-> move w left (decrease)',
              bg='#170F2A', color=PURPLE)

    else:
        ax.scatter([w0], [loss_fn(w0)], s=80, color=TXT_DIM, zorder=4, alpha=0.5)
        ax.scatter([w1], [loss_fn(w1)], s=110, color=GREEN, zorder=6)
        ax.annotate('', xy=(w1, loss_fn(w1)), xytext=(w0, loss_fn(w0)),
                    arrowprops=dict(arrowstyle='->', color=GREEN, lw=2.0))
        ax.annotate(f'w = {w1:.2f}\nL = {loss_fn(w1):.2f}  (lower)',
                    xy=(w1, loss_fn(w1)), xytext=(w1-1.8, loss_fn(w1)+1.5),
                    fontsize=9, color=GREEN, fontfamily=SANS,
                    arrowprops=dict(arrowstyle='->', color=GREEN, lw=1.2))
        ax.set_title(f'w ← {w0} − {alpha} × {grad0:.1f} = {w1:.2f}',
                     fontsize=9, color=TXT_SEC, pad=4, fontfamily=SANS)

fig.savefig(f'{OUTPUT_DIR}/04_one_step.png', dpi=150, bbox_inches='tight', facecolor=BG)
plt.close(fig)
```

### Diagram 5 — Learning Rate Comparison (step panels)

```python
fig = plt.figure(figsize=(16, 6.5))
page_header(fig, 'Learning Rate  α — How Step Size Affects Convergence',
            'The single most important hyperparameter to tune')
gs = gridspec.GridSpec(1, 3, figure=fig,
                       left=0.04, right=0.96, top=0.82, bottom=0.12, wspace=0.07)

configs = [
    (1.4, 'TOO LARGE  (α = 1.4)', RED,    'Overshoots — bounces forever'),
    (0.005, 'TOO SMALL  (α = 0.005)', AMBER, 'Converges but painfully slow'),
    (0.3, 'JUST RIGHT  (α = 0.3)', GREEN,  'Smooth convergence'),
]
w_arr = np.linspace(-3, 3.5, 300)
for i, (alpha, label, col, note) in enumerate(configs):
    ax = fig.add_subplot(gs[i])
    style_axes(ax)
    card_header(ax, label, col, note)
    ax.plot(w_arr, w_arr**2, color=CYAN, linewidth=2.0, alpha=0.5, zorder=2)
    ax.set_xlabel('w', fontsize=10); ax.set_ylabel('L(w)', fontsize=10)
    ax.set_xlim(-3, 3.5); ax.set_ylim(-0.3, 9)
    ax.set_facecolor(CARD)

    w = 2.5; ws = [w]
    for _ in range(14):
        w = w - alpha * 2 * w
        ws.append(w)
        if abs(w) > 4: break
    ls = [wi**2 for wi in ws]
    ax.plot(ws, ls, 'o-', color=col, linewidth=1.8, markersize=5, alpha=0.9, zorder=4)
    ax.scatter([ws[0]], [ls[0]], s=90, color=col, zorder=6)

fig.savefig(f'{OUTPUT_DIR}/05_learning_rate.png', dpi=150, bbox_inches='tight', facecolor=BG)
plt.close(fig)
```

### Diagram 6 — Summary Card

```python
sections = [
    {
        'heading': 'Core Idea', 'color': CYAN, 'bg': '#0B1829',
        'items': [
            'Minimize loss by stepping in the',
            'direction that reduces error most.',
            '',
            '= Mechanism:',
            '= Data -> Model -> Loss',
            '= -> Gradients -> Update -> repeat',
        ]
    },
    {
        'heading': 'Key Concepts', 'color': AMBER, 'bg': '#120E05',
        'items': [
            '# Loss function: measures wrongness',
            '# Gradient ∂L/∂w: slope of loss',
            '# Learning rate α: step size',
            '# Backprop: computes all gradients',
            '# Epoch: one full pass over data',
        ]
    },
    {
        'heading': 'Tradeoffs', 'color': GREEN, 'bg': '#081510',
        'items': [
            '+ Scales to billions of params',
            '+ Works with any smooth loss fn',
            '+ GPU-parallelizable',
            '- Can get stuck (local minima)',
            '- Sensitive to learning rate α',
            '- Needs differentiable loss',
        ]
    },
    {
        'heading': 'Where Next', 'color': PURPLE, 'bg': '#0D0818',
        'items': [
            '> SGD and mini-batches',
            '> Adam / RMSProp optimizers',
            '> Backpropagation derivation',
            '> Loss landscape geometry',
            '> Batch normalization',
            '> Learning rate schedules',
        ]
    },
]

fig = plt.figure(figsize=(15.2, 8.5))
fig.patch.set_facecolor(BG)
fig.text(0.5, 0.975, 'GRADIENT DESCENT — Reference Card',
         ha='center', va='top', fontsize=20, fontweight='bold',
         color=TXT_PRI, fontfamily=SANS)
fig.add_artist(plt.Line2D([0.08, 0.92], [0.956, 0.956],
               transform=fig.transFigure, color=CYAN, linewidth=1.5, alpha=0.6))

gs = gridspec.GridSpec(1, 4, figure=fig,
                       left=0.02, right=0.98, top=0.935, bottom=0.05, wspace=0.04)
PREFIX_COLORS = {'+': GREEN, '-': RED, '>': PURPLE, '=': AMBER, '#': CYAN}

for i, sec in enumerate(sections):
    ax = fig.add_subplot(gs[i])
    ax.set_facecolor(sec['bg']); ax.axis('off')
    for sp in ax.spines.values():
        sp.set_edgecolor(BORDER); sp.set_linewidth(1.2)
    acc = sec['color']
    ax.axhspan(0.935, 1.0, color=acc, alpha=0.15, transform=ax.transAxes)
    ax.text(0.5, 0.967, sec['heading'], ha='center', va='center',
            fontsize=11, fontweight='bold', color=acc,
            fontfamily=SANS, transform=ax.transAxes)
    ax.plot([0.05, 0.95], [0.928, 0.928], color=BORDER,
            linewidth=0.8, transform=ax.transAxes)
    for j, item in enumerate(sec['items']):
        prefix = item[0] if item and item[0] in PREFIX_COLORS else None
        color = PREFIX_COLORS.get(prefix, TXT_PRI if item else TXT_DIM)
        ax.text(0.06, 0.905 - j*0.082, item, ha='left', va='top',
                fontsize=9, color=color, fontfamily=SANS,
                transform=ax.transAxes)

fig.savefig(f'{OUTPUT_DIR}/06_summary_card.png', dpi=150, bbox_inches='tight', facecolor=BG)
plt.close(fig)
```

---

## Handling Special Input Types

### Paper / Article
1. Fetch the content.
2. Identify: problem statement, proposed method, key results, contribution.
3. Teach: why problem matters → prior work and gaps → the new idea →
   how it works → key results and what they prove → limitations.

### Repository / Codebase
1. Read README and key source files.
2. Identify: what it does, architecture, key modules, data flow.
3. Teach: problem it solves → high-level architecture (graphviz) →
   key abstractions → data/control flow → how to use → how to extend.

### URL / Website
1. Fetch the page.
2. Tutorial: follow its structure but add generated visuals and intuition.
3. Documentation: identify core API or concepts, teach those.
4. Article: extract argument and evidence, teach those.

---

## Quality Checks

Before finishing each section, verify:
- [ ] Every term is defined before it's used
- [ ] Every diagram uses the unified dark theme and palette
- [ ] Every diagram is explained in prose after being sent
- [ ] `badge()` is used for the single most important insight per diagram
- [ ] `page_header()` is called on every figure
- [ ] Every claim is accurate — never round off concepts that matter
- [ ] Each section ends with a one-sentence transition to the next
- [ ] The summary card is dense enough to serve as a standalone reference
- [ ] All PNGs sent via `SendUserFile` before prose references them
- [ ] No misconceptions were introduced in the name of simplicity
