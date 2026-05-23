---
name: progressive-disclosure-learning
description: >
  Explains a topic, article, paper, website, or repository through progressive
  disclosure — layered teaching with generated diagrams and visuals, like a
  picture book for adult learners. Trigger when the user asks to "explain",
  "teach", "walk me through", "break down", or "help me understand" a concept,
  paper, codebase, or URL. Also trigger for /learn, /explain, /teach.
---

# Progressive Disclosure Learning Skill

Teach any topic through layered explanation and generated visuals. The goal is
clarity, not simplification — concepts stay accurate and complete, but the
hardest ideas are introduced only after their foundations are established.

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

| Diagram type             | Tool        |
|--------------------------|-------------|
| Concept maps, flow diagrams, DAGs, system architecture | **graphviz** |
| Data plots, curves, charts, numeric relationships      | **matplotlib** |
| Annotated step-by-step diagrams, before/after visuals  | **matplotlib** (with annotations) |
| Summary cards / reference sheets                       | **matplotlib** (text layout) |

ASCII art is a last resort only when code execution is unavailable.

### Setup — always run this first

```python
import matplotlib
matplotlib.use('Agg')
import matplotlib.pyplot as plt
import matplotlib.patches as mpatches
import numpy as np
import graphviz
import os

OUTPUT_DIR = '/tmp/learn_diagrams'
os.makedirs(OUTPUT_DIR, exist_ok=True)

# Consistent visual style across all diagrams
plt.rcParams.update({
    'figure.facecolor': '#FAFAFA',
    'axes.facecolor': '#FAFAFA',
    'font.family': 'monospace',
    'axes.spines.top': False,
    'axes.spines.right': False,
})

BLUE   = '#2563EB'
ORANGE = '#EA580C'
GREEN  = '#16A34A'
GRAY   = '#6B7280'
LIGHT  = '#E5E7EB'
```

---

### Pattern 1 — Graphviz: Concept Map

Use for: topic overview maps, system architecture, data flow, dependency trees.

```python
def concept_map(nodes, edges, title, filename):
    """
    nodes: list of (id, label, shape) — shape in {'box','ellipse','diamond'}
    edges: list of (from_id, to_id, label)
    """
    g = graphviz.Digraph(
        graph_attr={
            'label': title, 'labelloc': 't', 'fontsize': '16',
            'bgcolor': '#FAFAFA', 'rankdir': 'TB', 'splines': 'ortho',
            'nodesep': '0.6', 'ranksep': '0.8'
        },
        node_attr={
            'style': 'filled', 'fillcolor': '#EFF6FF',
            'color': '#2563EB', 'fontsize': '12', 'fontname': 'Helvetica'
        },
        edge_attr={'color': '#6B7280', 'fontsize': '10', 'fontname': 'Helvetica'}
    )
    for nid, label, shape in nodes:
        g.node(nid, label=label, shape=shape)
    for src, dst, label in edges:
        g.edge(src, dst, label=label)
    path = f'{OUTPUT_DIR}/{filename}'
    g.render(path, format='png', cleanup=True)
    return path + '.png'
```

### Pattern 2 — Matplotlib: Data / Curve Plots

Use for: loss landscapes, training curves, probability distributions, any
relationship between numeric quantities.

```python
def line_plot(x, y_series, labels, title, xlabel, ylabel, annotations, filename):
    """
    y_series: list of arrays (one per line)
    annotations: list of (x, y, text, arrow_dx, arrow_dy)
    """
    fig, ax = plt.subplots(figsize=(9, 5))
    colors = [BLUE, ORANGE, GREEN, GRAY]
    for i, (y, label) in enumerate(zip(y_series, labels)):
        ax.plot(x, y, color=colors[i % len(colors)], linewidth=2.5, label=label)
    for (ax_, ay, text, dx, dy) in annotations:
        ax.annotate(text, xy=(ax_, ay), xytext=(ax_ + dx, ay + dy),
                    arrowprops=dict(arrowstyle='->', color=ORANGE),
                    fontsize=10, color=ORANGE)
    ax.set_title(title, fontsize=14, pad=12)
    ax.set_xlabel(xlabel, fontsize=11)
    ax.set_ylabel(ylabel, fontsize=11)
    if len(labels) > 1:
        ax.legend()
    plt.tight_layout()
    path = f'{OUTPUT_DIR}/{filename}'
    fig.savefig(path, dpi=150, bbox_inches='tight')
    plt.close(fig)
    return path
```

### Pattern 3 — Matplotlib: Step-by-Step Annotated Diagram

Use for: showing a mechanism evolving across multiple states side by side
(e.g., one update of gradient descent, one step of an algorithm).

```python
def step_panels(steps, title, filename):
    """
    steps: list of dicts, each with:
      'title': str
      'draw': callable(ax) that draws onto a matplotlib Axes
    """
    n = len(steps)
    fig, axes = plt.subplots(1, n, figsize=(5 * n, 4))
    if n == 1:
        axes = [axes]
    for ax, step in zip(axes, steps):
        step['draw'](ax)
        ax.set_title(step['title'], fontsize=12, pad=8)
        ax.set_facecolor('#FAFAFA')
    fig.suptitle(title, fontsize=14, y=1.02)
    plt.tight_layout()
    path = f'{OUTPUT_DIR}/{filename}'
    fig.savefig(path, dpi=150, bbox_inches='tight')
    plt.close(fig)
    return path
```

### Pattern 4 — Matplotlib: Summary / Reference Card

Use for: the closing summary card that users can save as a reference.

```python
def summary_card(title, sections, filename):
    """
    sections: list of (heading, list_of_bullet_strings)
    Renders a clean typeset card as a PNG.
    """
    n_sections = len(sections)
    fig, axes = plt.subplots(1, n_sections, figsize=(5 * n_sections, 6))
    if n_sections == 1:
        axes = [axes]
    fig.patch.set_facecolor('#1E293B')
    for ax, (heading, bullets) in zip(axes, sections):
        ax.set_facecolor('#1E293B')
        ax.axis('off')
        ax.text(0.5, 1.0, heading, transform=ax.transAxes,
                fontsize=12, fontweight='bold', color='#93C5FD',
                ha='center', va='top')
        for i, bullet in enumerate(bullets):
            ax.text(0.05, 0.88 - i * 0.13, f'• {bullet}',
                    transform=ax.transAxes, fontsize=10, color='#E2E8F0',
                    va='top', wrap=True)
    fig.suptitle(title, fontsize=16, fontweight='bold',
                 color='white', y=1.03)
    plt.tight_layout()
    path = f'{OUTPUT_DIR}/{filename}'
    fig.savefig(path, dpi=150, bbox_inches='tight', facecolor='#1E293B')
    plt.close(fig)
    return path
```

---

## Step 0 — Understand the Input

Before teaching, determine what you're working with:

- **Topic/concept** (e.g. "gradient descent", "Fourier transforms"): Teach
  from first principles.
- **URL / website / article**: Fetch the content, extract the key ideas, then
  teach those.
- **Research paper (URL or title)**: Fetch or recall the paper; identify its
  thesis, method, results, and contribution. Teach those.
- **Repository**: Read the README and key source files; identify what the
  project does, its architecture, and how it works. Teach those.

Also identify:
- What the user already knows (ask if not stated — one quick question max,
  like "Any background in ML or starting fresh?")
- Whether they want depth (full course) or breadth (executive summary)

---

## Step 1 — The Big Picture Diagram

Generate a single overview image that answers three questions visually:
1. What is this?
2. Why does it matter?
3. What problem does it solve?

Use a graphviz diagram with three clearly labeled clusters or a matplotlib
annotated figure. It should be readable at a glance — no jargon, no detail,
just orientation.

After sending the image, write 2–3 sentences of prose reinforcing the big
picture. Then ask the user if they want to continue or adjust depth/direction.

---

## Step 2 — The Conceptual Map

Generate a graphviz concept map of all key concepts and their relationships
before diving into any of them. Label each node with one short phrase. Label
edges with the relationship ("feeds into", "requires", "produces").

Send the image. Then list the sections you'll cover as a numbered agenda —
this gives the user a mental scaffold before you begin.

---

## Step 3 — Layer-by-Layer Teaching

Teach each concept from the map one section at a time. Each section follows
this structure:

**[Section title]**

**The Intuition** — A concrete analogy or real-world scenario the user already
understands. No jargon yet.

**The Diagram** — Generate a targeted visual for this concept only. Choose the
right pattern:
- A sub-graph (graphviz) if it's structural or relational
- A line/curve plot (matplotlib) if it involves numeric relationships
- Step panels (matplotlib) if you're showing a mechanism in action
- An annotated diagram with arrows and callouts if you need to highlight parts

**The Precise Explanation** — Accurate, complete explanation using real
terminology. If there are equations, show them inline as text and explain
every symbol. Don't skip the math if it's central — just introduce it
piece by piece.

**The "Why This Works" Beat** — What insight does this concept encode? What
breaks if you remove it?

**A Concrete Example** — Walk through one specific, small, numeric or
code-based example end-to-end.

**Common Misconceptions** — 1–2 things people get wrong here. Be direct:
"This does NOT mean X. It means Y."

End each section with a one-sentence transition to the next concept.

---

## Step 4 — The Full System Diagram

After teaching all parts, generate a complete system diagram showing how
everything connects and flows. This is the same conceptual map from Step 2,
fully annotated with the mechanics and data flow. Use graphviz with edge
labels explaining what passes between each stage.

---

## Step 5 — Real-World Grounding

Show:
- **Where it's actually used**: 2–3 real applications
- **What it looks like in practice**: A code snippet, real result, or live
  example
- **Known limitations**: Honest about tradeoffs and failure modes

If the comparison across applications is rich, generate a matplotlib table
visualization or a grouped bar chart.

---

## Step 6 — The Summary Card

Generate a summary card image using the `summary_card` pattern. It should
be dense enough to serve as a reference — not a watered-down recap. Include:
- Core idea (one precise sentence)
- Key concepts (tight definitions)
- The mechanism (step sequence)
- Key tradeoffs (strengths and limitations)
- Where to go next (related topics)

---

## Example: Gradient Descent (No ML Background)

This shows what the output should look like for a user with no prior ML
background. Generate these diagrams in sequence.

### Diagram 1 — Big Picture (graphviz)

```python
g = graphviz.Digraph(
    graph_attr={
        'label': 'Gradient Descent — Big Picture', 'labelloc': 't',
        'fontsize': '18', 'bgcolor': '#FAFAFA', 'rankdir': 'LR',
        'splines': 'curved', 'nodesep': '0.8'
    },
    node_attr={'style': 'filled', 'fontname': 'Helvetica', 'fontsize': '12'}
)
g.node('problem', 'Model makes\nbad predictions',
       shape='box', fillcolor='#FEE2E2', color='#DC2626')
g.node('gd', 'Gradient\nDescent',
       shape='ellipse', fillcolor='#DBEAFE', color='#2563EB')
g.node('result', 'Model learns\nto predict well',
       shape='box', fillcolor='#DCFCE7', color='#16A34A')
g.edge('problem', 'gd', label='  feeds into  ')
g.edge('gd', 'result', label='  produces  ')
g.render(f'{OUTPUT_DIR}/01_big_picture', format='png', cleanup=True)
```

### Diagram 2 — Conceptual Map (graphviz)

```python
g = graphviz.Digraph(
    graph_attr={
        'label': 'Gradient Descent — Concept Map', 'labelloc': 't',
        'fontsize': '16', 'bgcolor': '#FAFAFA', 'rankdir': 'TB',
        'splines': 'ortho', 'nodesep': '0.7', 'ranksep': '0.9'
    },
    node_attr={'style': 'filled', 'fontname': 'Helvetica', 'fontsize': '11'}
)
g.node('gd',    'Gradient Descent',  shape='ellipse', fillcolor='#DBEAFE', color='#2563EB')
g.node('loss',  'Loss Function',     shape='box',     fillcolor='#EFF6FF', color='#3B82F6')
g.node('grad',  'Gradient (Slope)',  shape='box',     fillcolor='#EFF6FF', color='#3B82F6')
g.node('lr',    'Learning Rate α',   shape='box',     fillcolor='#EFF6FF', color='#3B82F6')
g.node('update','Weight Update Rule',shape='box',     fillcolor='#EFF6FF', color='#3B82F6')
g.node('loop',  'Training Loop',     shape='ellipse', fillcolor='#F0FDF4', color='#16A34A')

g.edge('gd',    'loss',   label='measures error via')
g.edge('gd',    'grad',   label='computes direction via')
g.edge('gd',    'lr',     label='controls step size via')
g.edge('loss',  'update', label='feeds')
g.edge('grad',  'update', label='feeds')
g.edge('lr',    'update', label='feeds')
g.edge('update','loop',   label='repeated in')
g.render(f'{OUTPUT_DIR}/02_concept_map', format='png', cleanup=True)
```

### Diagram 3 — Loss Landscape (matplotlib)

```python
fig, ax = plt.subplots(figsize=(9, 5), facecolor='#FAFAFA')
ax.set_facecolor('#FAFAFA')
w = np.linspace(-3, 3, 300)
loss = w**2 + 0.3 * np.sin(5 * w)  # slightly bumpy to show local minima

ax.plot(w, loss, color=BLUE, linewidth=2.5, label='Loss L(w)')
ax.fill_between(w, loss, alpha=0.08, color=BLUE)

# Mark global minimum
min_idx = np.argmin(loss)
ax.scatter([w[min_idx]], [loss[min_idx]], color=GREEN, s=80, zorder=5)
ax.annotate('Global minimum\n(best model weights)', xy=(w[min_idx], loss[min_idx]),
            xytext=(w[min_idx] + 1.0, loss[min_idx] + 1.5),
            arrowprops=dict(arrowstyle='->', color=GREEN, lw=1.5),
            fontsize=10, color=GREEN)

# Mark current position
ax.scatter([2.0], [2.0**2 + 0.3 * np.sin(5 * 2.0)], color=ORANGE, s=80, zorder=5)
ax.annotate("Current\nmodel weights", xy=(2.0, 2.0**2 + 0.3*np.sin(10)),
            xytext=(1.2, 6.5),
            arrowprops=dict(arrowstyle='->', color=ORANGE, lw=1.5),
            fontsize=10, color=ORANGE)

ax.set_title('The Loss Landscape — our terrain to navigate', fontsize=13, pad=12)
ax.set_xlabel('Weight value (w)', fontsize=11)
ax.set_ylabel('Loss — how wrong the model is', fontsize=11)
ax.spines['top'].set_visible(False)
ax.spines['right'].set_visible(False)
plt.tight_layout()
fig.savefig(f'{OUTPUT_DIR}/03_loss_landscape.png', dpi=150, bbox_inches='tight')
plt.close(fig)
```

### Diagram 4 — One Gradient Descent Step (matplotlib, step panels)

```python
fig, axes = plt.subplots(1, 3, figsize=(14, 4), facecolor='#FAFAFA')
w_vals = np.linspace(-2, 3, 300)
loss_fn = lambda w: w**2

for ax in axes:
    ax.set_facecolor('#FAFAFA')
    ax.plot(w_vals, loss_fn(w_vals), color=BLUE, linewidth=2)
    ax.set_xlabel('w', fontsize=10)
    ax.set_ylabel('L(w)', fontsize=10)
    ax.spines['top'].set_visible(False)
    ax.spines['right'].set_visible(False)

# Panel 1 — current position
w0 = 2.2
axes[0].scatter([w0], [loss_fn(w0)], color=ORANGE, s=80, zorder=5)
axes[0].annotate(f'w = {w0}\nL = {loss_fn(w0):.2f}', xy=(w0, loss_fn(w0)),
                 xytext=(w0 - 1.5, loss_fn(w0) + 1.0),
                 arrowprops=dict(arrowstyle='->', color=ORANGE),
                 fontsize=9, color=ORANGE)
axes[0].set_title('Step 1: Current weights', fontsize=11)

# Panel 2 — gradient (tangent line)
grad = 2 * w0
w_tan = np.linspace(w0 - 0.8, w0 + 0.8, 10)
axes[1].plot(w_tan, loss_fn(w0) + grad * (w_tan - w0), color=ORANGE,
             linewidth=2, linestyle='--', label=f'slope = {grad:.1f}')
axes[1].scatter([w0], [loss_fn(w0)], color=ORANGE, s=80, zorder=5)
axes[1].annotate(f'∂L/∂w = {grad:.1f}\n(slope at w={w0})',
                 xy=(w0, loss_fn(w0)), xytext=(w0 - 1.8, loss_fn(w0) + 0.5),
                 arrowprops=dict(arrowstyle='->', color=ORANGE),
                 fontsize=9, color=ORANGE)
axes[1].set_title('Step 2: Compute gradient (slope)', fontsize=11)

# Panel 3 — after update
alpha = 0.3
w1 = w0 - alpha * grad
axes[2].scatter([w0], [loss_fn(w0)], color=GRAY, s=60, zorder=5, alpha=0.4, label='before')
axes[2].scatter([w1], [loss_fn(w1)], color=GREEN, s=80, zorder=5, label='after')
axes[2].annotate('', xy=(w1, loss_fn(w1)), xytext=(w0, loss_fn(w0)),
                 arrowprops=dict(arrowstyle='->', color=GREEN, lw=2))
axes[2].annotate(f'w = {w1:.2f}\nL = {loss_fn(w1):.2f}',
                 xy=(w1, loss_fn(w1)), xytext=(w1 - 1.5, loss_fn(w1) + 1.0),
                 arrowprops=dict(arrowstyle='->', color=GREEN),
                 fontsize=9, color=GREEN)
axes[2].set_title(f'Step 3: Update  w ← w − α·grad\n= {w0} − {alpha}×{grad} = {w1:.2f}',
                  fontsize=11)

fig.suptitle('One Gradient Descent Step', fontsize=13, y=1.02)
plt.tight_layout()
fig.savefig(f'{OUTPUT_DIR}/04_one_step.png', dpi=150, bbox_inches='tight')
plt.close(fig)
```

### Diagram 5 — Learning Rate Comparison (matplotlib)

```python
fig, axes = plt.subplots(1, 3, figsize=(14, 4), facecolor='#FAFAFA')
titles = ['Too large (α = 1.2) — overshoots', 'Too small (α = 0.005) — slow',
          'Just right (α = 0.3) — converges']
alphas = [1.2, 0.005, 0.3]
colors_path = [ORANGE, GRAY, GREEN]

for ax, alpha, title, col in zip(axes, alphas, titles, colors_path):
    ax.set_facecolor('#FAFAFA')
    w_range = np.linspace(-3, 3, 300)
    ax.plot(w_range, w_range**2, color=BLUE, linewidth=2, alpha=0.4)
    ax.set_xlabel('w', fontsize=10)
    ax.set_ylabel('Loss', fontsize=10)
    ax.set_title(title, fontsize=10)
    ax.spines['top'].set_visible(False)
    ax.spines['right'].set_visible(False)

    w = 2.5
    ws = [w]
    for _ in range(12):
        w = w - alpha * 2 * w
        ws.append(w)
        if abs(w) > 5:
            break

    ls = [wi**2 for wi in ws]
    ax.plot(ws, ls, 'o-', color=col, linewidth=1.5, markersize=5, alpha=0.85)
    ax.scatter([ws[0]], [ls[0]], color=col, s=80, zorder=5)

fig.suptitle('Learning Rate: How Step Size Affects Convergence', fontsize=13, y=1.02)
plt.tight_layout()
fig.savefig(f'{OUTPUT_DIR}/05_learning_rate.png', dpi=150, bbox_inches='tight')
plt.close(fig)
```

### Diagram 6 — Full System Flow (graphviz)

```python
g = graphviz.Digraph(
    graph_attr={
        'label': 'Gradient Descent — Full Training Loop', 'labelloc': 't',
        'fontsize': '16', 'bgcolor': '#FAFAFA', 'rankdir': 'TB',
        'splines': 'ortho', 'nodesep': '0.6', 'ranksep': '0.7'
    },
    node_attr={'style': 'filled', 'fontname': 'Helvetica', 'fontsize': '11'}
)
g.node('data',   'Training Data',         shape='cylinder', fillcolor='#F0FDF4', color='#16A34A')
g.node('model',  'Model\n(current weights)', shape='box',   fillcolor='#DBEAFE', color='#2563EB')
g.node('pred',   'Predictions',           shape='box',      fillcolor='#EFF6FF', color='#3B82F6')
g.node('loss',   'Loss  L(w)',            shape='ellipse',  fillcolor='#FEF3C7', color='#D97706')
g.node('grad',   'Gradients  ∂L/∂w',     shape='ellipse',  fillcolor='#FEF3C7', color='#D97706')
g.node('update', 'Update weights\nw ← w − α·∂L/∂w', shape='box', fillcolor='#DCFCE7', color='#16A34A')
g.node('check',  'Loss low\nenough?',     shape='diamond',  fillcolor='#F5F3FF', color='#7C3AED')
g.node('done',   'Trained Model',         shape='ellipse',  fillcolor='#DCFCE7', color='#15803D')

g.edge('data',   'model',  label='batch of examples')
g.edge('model',  'pred',   label='forward pass')
g.edge('pred',   'loss',   label='compare to labels')
g.edge('loss',   'grad',   label='backpropagation')
g.edge('grad',   'update', label='apply update rule')
g.edge('update', 'check',  '')
g.edge('check',  'model',  label='No — loop back', style='dashed')
g.edge('check',  'done',   label='Yes — done')

g.render(f'{OUTPUT_DIR}/06_full_system', format='png', cleanup=True)
```

### Diagram 7 — Summary Card (matplotlib)

```python
fig = plt.figure(figsize=(14, 7), facecolor='#0F172A')
fig.suptitle('GRADIENT DESCENT — Reference Card',
             fontsize=16, fontweight='bold', color='white', y=0.97)

sections = [
    ('Core Idea', [
        'Minimize loss by stepping in the',
        'direction that reduces error most.',
        '',
        'Mechanism:',
        'Data → Model → Loss → Gradients',
        '→ Update → (repeat)',
    ], '#1E3A5F'),
    ('Key Concepts', [
        'Loss function: measures wrongness',
        'Gradient ∂L/∂w: slope of loss',
        'Learning rate α: step size',
        'Backprop: computes all gradients',
        'Epoch: one full pass over data',
    ], '#1E3A5F'),
    ('Tradeoffs', [
        '✓ Scales to billions of params',
        '✓ Works with any smooth loss',
        '✓ GPU-parallelizable',
        '✗ Can get stuck (local minima)',
        '✗ Sensitive to learning rate α',
        '✗ Requires differentiable loss',
    ], '#1A3A2F'),
    ('Where Next', [
        '→ SGD and mini-batches',
        '→ Adam / RMSProp optimizers',
        '→ Backpropagation in depth',
        '→ Loss landscape geometry',
        '→ Batch normalization',
    ], '#2D1B4E'),
]

axes = fig.subplots(1, len(sections))
for ax, (heading, lines, bg) in zip(axes, sections):
    ax.set_facecolor(bg)
    ax.axis('off')
    ax.text(0.5, 0.97, heading, transform=ax.transAxes,
            fontsize=12, fontweight='bold', color='#93C5FD',
            ha='center', va='top', fontfamily='monospace')
    ax.axhline(y=0.90, color='#334155', linewidth=0.8,
               xmin=0.05, xmax=0.95, transform=ax.transAxes)
    for i, line in enumerate(lines):
        color = '#E2E8F0' if not line.startswith('✓') else '#86EFAC'
        if line.startswith('✗'):
            color = '#FCA5A5'
        if line.startswith('→'):
            color = '#C4B5FD'
        ax.text(0.06, 0.86 - i * 0.11, line, transform=ax.transAxes,
                fontsize=9.5, color=color, va='top', fontfamily='monospace')

plt.tight_layout(rect=[0, 0, 1, 0.94])
fig.savefig(f'{OUTPUT_DIR}/07_summary_card.png', dpi=150, bbox_inches='tight',
            facecolor='#0F172A')
plt.close(fig)
```

---

## Handling Special Input Types

### Paper / Article
1. Fetch the content.
2. Identify: problem statement, proposed method, key results, contribution.
3. Teach in this order: Why the problem matters → Prior work and gaps →
   The new idea → How it works mechanically → Key results and what they prove
   → Limitations and open questions.

### Repository / Codebase
1. Read README and key source files.
2. Identify: what it does, its architecture, key modules, data flow.
3. Teach in this order: Problem it solves → High-level architecture diagram
   (graphviz) → Key abstractions and roles → Data/control flow through the
   system → How to use it → How to extend it.

### URL / Website
1. Fetch the page content.
2. If a tutorial: follow its structure but add generated diagrams and intuition.
3. If documentation: identify the core API or concepts; teach those.
4. If an article: extract the argument and evidence; teach those.

---

## Quality Checks

Before finishing each section, verify:
- [ ] Every term is defined before it's used
- [ ] Every diagram is explained in prose after being sent
- [ ] Every claim is accurate — never round off concepts that matter
- [ ] Each section ends with a clear transition to the next
- [ ] The summary card captures the full topic at reference density
- [ ] No misconceptions were introduced for the sake of simplicity
- [ ] All generated PNGs were sent with `SendUserFile` before prose references them
