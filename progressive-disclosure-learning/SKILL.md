---
name: progressive-disclosure-learning
description: >
  Explains a topic, article, paper, website, or repository through progressive
  disclosure — layered teaching with ASCII diagrams and visuals, like a picture
  book for adult learners. Trigger when the user asks to "explain", "teach",
  "walk me through", "break down", or "help me understand" a concept, paper,
  codebase, or URL. Also trigger for /learn, /explain, /teach.
---

# Progressive Disclosure Learning Skill

Teach any topic through layered explanation and visuals. The goal is clarity,
not simplification — concepts stay accurate and complete, but the hardest ideas
are introduced only after their foundations are established.

## Guiding Principles

- **Progressive disclosure**: Each layer builds on the last. Never introduce a
  term before defining it. Never show a full system before its parts.
- **One idea per beat**: Each section focuses on exactly one concept or
  mechanism. Don't cram.
- **Visuals first, prose second**: Lead with a diagram or visual whenever
  possible. Explain the visual in prose afterward.
- **Calibrate to the user**: Use the user's stated expertise level, or infer it
  from their phrasing. Default to "no domain expertise" if uncertain.
- **Accuracy over approachability**: Never distort a concept for simplicity.
  If a concept is hard, say so and scaffold it carefully.
- **Complete the picture**: By the end, the user should understand the full
  topic — not just a simplified summary.

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

If the user provides a URL, fetch its content before beginning. If it's a
paper, extract: problem being solved, proposed solution, key results, and
why it matters.

Also identify:
- What the user already knows (ask if not stated, but make this feel light —
  one quick question max, like "Any background in ML or are we starting fresh?")
- Whether they want depth (full course) or breadth (executive summary)

---

## Step 1 — The Big Picture Card

Begin every explanation with a "Big Picture Card": a single-screen overview
that answers three questions in a visual way:

1. What is this?
2. Why does it matter?
3. What problem does it solve?

Format it like this — adapt the ASCII art and layout to the topic:

```
╔══════════════════════════════════════════════════════════════╗
║  [TOPIC NAME]                                                ║
╠══════════════════════════════════════════════════════════════╣
║                                                              ║
║  WHAT IT IS:   [One clear sentence]                          ║
║                                                              ║
║  WHY IT MATTERS: [One clear sentence about real impact]      ║
║                                                              ║
║  THE CORE PROBLEM IT SOLVES:                                 ║
║                                                              ║
║    [Before state]    →→→  [This technique]  →→→  [After]    ║
║                                                              ║
╚══════════════════════════════════════════════════════════════╝
```

After the card, write 2–3 sentences max of plain prose reinforcing the
big picture. Then pause: ask the user if they want to continue or adjust
the depth/direction before proceeding to Step 2.

---

## Step 2 — The Conceptual Map

Show the full landscape of the topic before diving into any part of it.
This prevents the "lost in the weeds" problem — the user always knows where
they are.

Create a visual map of all the key concepts and how they connect:

```
                    ┌─────────────────────────────────┐
                    │         [MAIN CONCEPT]          │
                    └────────────┬────────────────────┘
                                 │
            ┌────────────────────┼────────────────────┐
            ▼                    ▼                    ▼
    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
    │  Concept A   │    │  Concept B   │    │  Concept C   │
    └──────┬───────┘    └──────┬───────┘    └──────┬───────┘
           │                  │                    │
          ...                ...                  ...
```

Label each node with one phrase. Annotate arrows with the relationship
("feeds into", "requires", "produces").

Then tell the user: "We'll go through each of these in order. Each one
builds on the last."

List the sections you'll cover as a numbered agenda — this gives the user a
mental scaffold before you begin.

---

## Step 3 — Layer-by-Layer Teaching

Teach each concept from the map one section at a time. Each section follows
this exact structure:

### Section Template

**[Section title with a clear, specific name]**

**[The Intuition]** — Start with a concrete analogy or real-world scenario
that the user already understands. No jargon yet.

**[The Visual]** — Draw the concept using ASCII art, a diagram, a table, or a
before/after comparison. Examples of visual types to use:

- Flow diagrams for processes:
  ```
  Input → [Step 1] → [Step 2] → [Step 3] → Output
              ↓
         [If condition]
              ↓
         [Branch path]
  ```

- State diagrams for transitions:
  ```
  State A ──(trigger)──→ State B ──(trigger)──→ State C
     ↑                                              │
     └──────────────(reset)────────────────────────┘
  ```

- Graphs/charts for data relationships (ASCII plot):
  ```
  Loss
   │ ×
   │  ×
   │   ×
   │     ×
   │        ×  ×  ×
   └─────────────────→ Training steps
  ```

- Tables for comparisons:
  ```
  ┌──────────────┬───────────────┬──────────────────┐
  │              │  Approach A   │   Approach B     │
  ├──────────────┼───────────────┼──────────────────┤
  │ Speed        │ Fast          │ Slow             │
  │ Accuracy     │ Lower         │ Higher           │
  │ Use when...  │ Prototyping   │ Production       │
  └──────────────┴───────────────┴──────────────────┘
  ```

- Code annotation diagrams for algorithms:
  ```
  for each weight w:              ← iterate over parameters
      gradient = ∂Loss/∂w         ← how much w affects loss
      w = w - α × gradient        ← take a step downhill
               ↑
           learning rate
  ```

**[The Precise Explanation]** — Now explain the concept accurately and
completely. Use real terminology. If there are equations, show them and
explain every symbol. Don't skip the math if it's central; just introduce
it piece by piece.

**[The "Why This Works" or "Why This Matters" Beat]** — Explain the insight
behind the concept. What problem does this solve? What breaks if you remove it?

**[A Concrete Example]** — Walk through one specific, small example end-to-end.
Preferably numeric or code-based. Don't abstract away the numbers.

**[Common Misconceptions]** — Call out 1–2 things people commonly get wrong
here. Be direct: "This does NOT mean X. It means Y."

After each section, offer a brief transition: "Now that we understand [A],
we can see why [B] works the way it does. Let's look at [B]."

---

## Step 4 — The Full System View

After teaching all the parts, reassemble them into a complete diagram showing
how everything connects and flows together. This is the same conceptual map
from Step 2, but now fully annotated with the mechanics.

```
┌────────────────────────────────────────────────────────────────────┐
│                      FULL SYSTEM: [Topic]                          │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  [Input/Problem]                                                   │
│       │                                                            │
│       ▼                                                            │
│  ┌─────────┐   [mechanism]   ┌─────────┐   [mechanism]            │
│  │ Stage 1 │ ─────────────→  │ Stage 2 │ ─────────────→  ...      │
│  └─────────┘                 └─────────┘                          │
│       ↑                           │                               │
│       │       [feedback loop]     │                               │
│       └───────────────────────────┘                               │
│                                                                    │
│  [Output/Result]                                                   │
└────────────────────────────────────────────────────────────────────┘
```

---

## Step 5 — Real-World Grounding

For the topic, show:
- **Where it's actually used**: 2–3 real applications with brief notes
- **What it looks like in practice**: A snippet of code, a real dataset result,
  an example from a paper, a live system that uses it
- **Known limitations**: What does this approach fail at? What are the
  tradeoffs? Be honest.

Format as a compact table when there are multiple examples:

```
┌──────────────────┬──────────────────────┬───────────────────────┐
│ Application      │ How it's used        │ Why this approach     │
├──────────────────┼──────────────────────┼───────────────────────┤
│ Neural networks  │ Train model weights  │ Scalable, GPU-friendly│
│ Logistics        │ Route optimization   │ Fast approximation    │
│ Finance          │ Portfolio balancing  │ Handles high-dim data │
└──────────────────┴──────────────────────┴───────────────────────┘
```

---

## Step 6 — The Summary Card

End with a closing card that the user could screenshot or save. It should
be a complete, dense reference — not a watered-down recap:

```
╔══════════════════════════════════════════════════════════════╗
║  [TOPIC NAME] — Summary                                      ║
╠══════════════════════════════════════════════════════════════╣
║                                                              ║
║  CORE IDEA:                                                  ║
║    [One precise sentence]                                    ║
║                                                              ║
║  KEY CONCEPTS:                                               ║
║    • [Concept 1]: [tight definition]                         ║
║    • [Concept 2]: [tight definition]                         ║
║    • [Concept 3]: [tight definition]                         ║
║    • [Concept 4]: [tight definition]                         ║
║                                                              ║
║  THE MECHANISM:                                              ║
║    [Step 1] → [Step 2] → [Step 3] → [Result]                ║
║                                                              ║
║  KEY TRADEOFFS:                                              ║
║    ✓ [Strength 1]           ✗ [Limitation 1]                ║
║    ✓ [Strength 2]           ✗ [Limitation 2]                ║
║                                                              ║
║  WHERE TO GO NEXT:                                           ║
║    → [Related topic A]                                       ║
║    → [Related topic B]                                       ║
╚══════════════════════════════════════════════════════════════╝
```

---

## Example Walkthrough — Gradient Descent (No ML Background)

Below is how this skill should look in practice, for a user with no ML
background. Use this as a reference for tone, visual density, and pacing.

---

### BIG PICTURE CARD

```
╔══════════════════════════════════════════════════════════════╗
║  GRADIENT DESCENT                                            ║
╠══════════════════════════════════════════════════════════════╣
║                                                              ║
║  WHAT IT IS:   An algorithm that finds the minimum of a     ║
║                mathematical function by repeatedly taking   ║
║                small steps downhill.                        ║
║                                                              ║
║  WHY IT MATTERS: It's how nearly all modern neural networks  ║
║                  learn — training a model IS running this   ║
║                  algorithm thousands of times.              ║
║                                                              ║
║  THE CORE PROBLEM IT SOLVES:                                ║
║                                                              ║
║   "Model is wrong"  →  [Gradient Descent]  →  "Model works" ║
║                                                              ║
╚══════════════════════════════════════════════════════════════╝
```

Gradient descent is the engine behind machine learning. When a model makes
bad predictions, gradient descent is what guides it to make better ones —
automatically, without anyone hand-tuning the model's internal settings.

---

### CONCEPTUAL MAP

```
                    ┌────────────────────────┐
                    │    GRADIENT DESCENT    │
                    └───────────┬────────────┘
                                │
          ┌─────────────────────┼─────────────────────┐
          ▼                     ▼                     ▼
  ┌──────────────┐    ┌──────────────────┐    ┌─────────────┐
  │  Loss        │    │  Gradient        │    │  Learning   │
  │  Function    │    │  (Slope)         │    │  Rate       │
  └──────────────┘    └──────────────────┘    └─────────────┘
         │                     │                     │
         └──────────┬──────────┘                     │
                    ▼                                │
          ┌──────────────────┐                       │
          │  Weight Update   │ ←─────────────────────┘
          └──────────────────┘
```

We'll cover: (1) Loss Functions, (2) Gradients, (3) The Update Rule,
(4) Learning Rate, (5) Full Training Loop, (6) Variants and Tradeoffs.

---

### SECTION 1 — Loss Functions

**The Intuition**

Imagine you're blindfolded, standing somewhere on a hilly landscape. Your
goal is to reach the lowest valley. You can't see, but you can feel the
slope under your feet.

The "hilly landscape" in machine learning is the **loss function** — a
mathematical way to measure how wrong the model is. The valley floor is
perfect predictions. Every hill is error.

**The Visual**

```
Loss (how wrong the model is)
  │
  │ ╲                         ╱
  │  ╲                       ╱
  │   ╲                     ╱
  │    ╲         ___       ╱
  │     ╲       ╱   ╲     ╱
  │      ╲_____╱     ╲___╱
  │
  │              ↑
  │         Global minimum
  │         (best model)
  └──────────────────────────→ Model parameters (weights)
```

Each point on the x-axis is a different set of model weights — different
configurations of the model. The y-axis is how bad the model is with those
weights. We want to find the x that gives the lowest y.

**The Precise Explanation**

A loss function L(w) takes the model's current weights w and outputs a
single number: the error. Common choices:

- **Mean Squared Error** (regression): L = (1/n) Σ (predicted − actual)²
- **Cross-Entropy Loss** (classification): L = −Σ y · log(ŷ)

The specific formula matters less than the idea: loss is a score of
wrongness, and we want to minimize it.

**Why This Works**

If we can express "how wrong the model is" as a smooth mathematical
function of its weights, we can use calculus to find which direction
to adjust those weights to make the model less wrong.

**Concrete Example**

Say our model predicts house prices. It predicts $300k but the real price
is $400k. MSE loss for this one example: (300 − 400)² = 10,000. The
model is $10,000 units wrong. We need to adjust its weights to push
this number toward zero.

**Common Misconceptions**

- Loss is NOT accuracy. Accuracy is discrete (right/wrong). Loss is
  continuous and differentiable — that's why we can take its gradient.
- The loss landscape is not always a smooth bowl. It can be jagged, flat,
  or have many local minima. That's why gradient descent has failure modes.

---

### SECTION 2 — Gradients

**The Intuition**

Back to the blindfolded hiker. To find downhill, you stomp your foot and
feel which direction tilts steepest. That direction is the **gradient** —
the slope of the loss function with respect to each weight.

**The Visual**

```
Loss
  │   ●  ← current position
  │    ╲
  │     ╲  ← slope here is steep and negative
  │      ╲         (gradient points left = uphill)
  │       ╲
  │        ╲___
  │             ╲____ ← minimum
  └─────────────────────→ Weight value

  gradient = slope = ∂L/∂w

  If gradient > 0: loss increases as w increases → move w LEFT (decrease w)
  If gradient < 0: loss decreases as w increases → move w RIGHT (increase w)
```

**The Precise Explanation**

The gradient ∂L/∂w is the partial derivative of the loss with respect to
a weight w. For a model with many weights (a neural network can have
billions), we compute one gradient per weight. Together they form the
**gradient vector** — a direction in weight-space pointing toward steeper
loss.

**Backpropagation** is the algorithm that efficiently computes all these
gradients using the chain rule of calculus. You don't need to derive it
by hand — PyTorch and TensorFlow do this automatically (called autograd).

**Why This Works**

The derivative tells us the exact direction that increases loss most
steeply. So moving in the opposite direction decreases loss most steeply.
That's it. That's gradient descent.

**Concrete Example**

Suppose L = w² (simple 1-weight model). The gradient is ∂L/∂w = 2w.
At w = 3: gradient = 6. Loss is increasing at w = 3, so we should
decrease w. After the update: w = 3 − α × 6. With α = 0.1: w = 2.4.
We moved closer to the minimum at w = 0.

**Common Misconceptions**

- The gradient points **uphill** (toward higher loss), not downhill.
  We move in the **negative** gradient direction. This confuses many people.
- Gradient ≠ the loss value. It's the rate of change of loss.

---

### SECTION 3 — The Update Rule

**The Core Equation**

```
  w ← w − α · ∂L/∂w

  w         = current weight
  α         = learning rate (a small number like 0.01)
  ∂L/∂w    = gradient (slope of loss at current w)
  w ← ...   = new weight after one step
```

**The Visual**

```
  One step of gradient descent:

  BEFORE:                     AFTER:
  Loss                        Loss
    │  ●                        │
    │   ╲                       │    ●
    │    ╲                      │     ╲
    │     ╲___                  │      ╲___
    └──────────→ w              └──────────→ w

        w                          w - α·gradient
```

We move the weight a little bit in the direction that reduces loss.
Repeat thousands of times. The model converges to good weights.

---

### SECTION 4 — Learning Rate

**The Intuition**

α (alpha) controls the size of each step. Too large: you overshoot the
minimum and bounce around. Too small: you inch forward and training
takes forever.

```
  TOO LARGE (α = 1.0):         TOO SMALL (α = 0.00001):
  Loss                         Loss
    │●                           │●
    │  ╲      ●                  │ ●
    │   ╲    ╱ ╲                 │  ●
    │    ╲  ╱   ╲                │   ●
    │    minimum ●               │    ●
    │     ↑ overshot!            │     ... (very slow)
    └──────→ w                  └──────→ w

  JUST RIGHT (α = 0.01):
  Loss
    │●
    │ ●
    │  ●
    │   ●●●──→ minimum
    └──────→ w
```

**Common Misconceptions**

- There is no universally correct learning rate. It depends on the problem,
  model, and data. You typically tune it.
- Learning rate schedules (reducing α over time) often work better than a
  fixed α. Start fast, refine slowly.

---

### THE FULL SYSTEM VIEW

```
┌───────────────────────────────────────────────────────────────────┐
│                   GRADIENT DESCENT — FULL LOOP                    │
├───────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌──────────┐                                                     │
│  │ Training │ → feed batch of examples into model                 │
│  │   Data   │                                                     │
│  └──────────┘                                                     │
│       │                                                           │
│       ▼                                                           │
│  ┌──────────┐                                                     │
│  │  Model   │ → produces predictions with current weights         │
│  │ (weights)│                                                     │
│  └──────────┘                                                     │
│       │                                                           │
│       ▼                                                           │
│  ┌──────────┐                                                     │
│  │   Loss   │ → L = compare predictions to true labels           │
│  │ Function │                                                     │
│  └──────────┘                                                     │
│       │                                                           │
│       ▼                                                           │
│  ┌──────────┐                                                     │
│  │  Backprop│ → compute ∂L/∂w for every weight                   │
│  │ (autodiff│                                                     │
│  └──────────┘                                                     │
│       │                                                           │
│       ▼                                                           │
│  ┌──────────┐                                                     │
│  │  Update  │ → w ← w − α · ∂L/∂w  (for all weights)            │
│  │  Weights │                                                     │
│  └──────────┘                                                     │
│       │                                                           │
│       └──────────→  loop back (repeat until loss is low enough)  │
│                                                                   │
└───────────────────────────────────────────────────────────────────┘
```

---

### SUMMARY CARD

```
╔══════════════════════════════════════════════════════════════╗
║  GRADIENT DESCENT — Summary                                  ║
╠══════════════════════════════════════════════════════════════╣
║                                                              ║
║  CORE IDEA:                                                  ║
║    Minimize a loss function by repeatedly nudging weights    ║
║    in the direction that reduces error, guided by the slope  ║
║    (gradient) of the loss surface.                           ║
║                                                              ║
║  KEY CONCEPTS:                                               ║
║    • Loss function: measures how wrong the model is         ║
║    • Gradient: slope of the loss w.r.t. each weight         ║
║    • Learning rate (α): step size per update                ║
║    • Backprop: efficient algorithm to compute all gradients  ║
║    • Epoch: one full pass through the training data         ║
║                                                              ║
║  THE MECHANISM:                                              ║
║    Data → Model → Loss → Gradients → Update → (repeat)      ║
║                                                              ║
║  KEY TRADEOFFS:                                              ║
║    ✓ Scalable to billions of params  ✗ Can get stuck in     ║
║    ✓ Works with any differentiable     local minima          ║
║      loss function                  ✗ Sensitive to α choice ║
║    ✓ Parallelizable on GPUs         ✗ Requires smooth loss  ║
║                                                              ║
║  WHERE TO GO NEXT:                                           ║
║    → Stochastic Gradient Descent (SGD) and mini-batches      ║
║    → Adam, RMSProp (adaptive learning rate optimizers)       ║
║    → Backpropagation in depth                                ║
║    → Loss landscape geometry (saddle points, plateaus)       ║
╚══════════════════════════════════════════════════════════════╝
```

---

## Handling Special Input Types

### Paper / Article
1. Fetch the content.
2. Identify: problem statement, proposed method, key results, and contribution.
3. Teach in this order: Why the problem matters → Prior art and why it fell
   short → The new idea → How it works mechanically → Key results and what
   they prove → Limitations and open questions.

### Repository / Codebase
1. Read README and key source files.
2. Identify: what it does, its architecture, key modules, data flow.
3. Teach in this order: What problem it solves → High-level architecture →
   Key abstractions and their roles → Data/control flow through the system →
   How to use it → How to extend or modify it.

### URL / Website
1. Fetch the page content.
2. If it's a tutorial: follow its structure but add visuals and intuition.
3. If it's documentation: identify the core API or concepts; teach those.
4. If it's an article: extract the argument and evidence; teach those.

---

## Quality Checks

Before finishing each section, verify:
- [ ] Every term is defined before it's used
- [ ] Every diagram is explained in prose
- [ ] Every claim is accurate (do not round off concepts that matter)
- [ ] Each section ends with a clear transition to the next
- [ ] The summary card captures the full topic at a density suitable for
      review/reference
- [ ] No misconceptions were introduced in the name of simplicity
