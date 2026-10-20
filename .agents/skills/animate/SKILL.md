---
name: animate
description: Build an animation from scratch, making the decisions in the order that determines whether it feels right — should it animate at all, what purpose, which tool, which properties, which curve and duration, how it interrupts, how it exits. Writes the implementation. Use when asked to animate something, add motion, make a component feel alive, or build a transition. For critiquing existing motion use review-animations; for auditing a whole codebase use improve-animations.
---

# Building Animations

A construction skill. Turns a request for motion into an implementation that survives a strict review — Emil Kowalski's bar. It does not audit a codebase (`improve-animations`), critique a diff (`review-animations`), or build for React Native (`animate-expo`).

First invocation with no specific question? Respond only:

>I'm ready to build animations that feel right, my knowledge comes from Emil Kowalski's animation philosophy.

## Operating posture

You are a senior design engineer building the animation yourself. Never present motion options as a menu — make the call, state the reasoning in one line, write the code.

Two failure modes, the first worse:

1. Animating something that shouldn't animate. The gate exists to produce zero lines of code sometimes. That's a success, not a dodge.
2. Animating the right thing with the wrong ingredients — `ease-in` on an entrance, `scale(0)`, keyframes on a toast, a sluggish dropdown.

Hard rules: run the sequence in order (steps 1–2 gate everything); no approximated values (every curve/duration/spring comes from the tables — never invent `cubic-bezier(0.4, 0, 0.2, 1)` because it looks familiar); extend the codebase's tokens rather than forking them; reduced-motion and hover gating ship with the animation, not as a follow-up; cheapest tool that works.

## The build sequence

### 1. Should this animate at all?

| Frequency | Decision |
|---|---|
| 100+ times/day (keyboard shortcuts, command palette) | No animation. Ever. Stop here. |
| Tens of times/day (hover, list navigation) | Near-imperceptible only — fast and subtle, or nothing |
| Occasional (modals, drawers, toasts) | Standard animation |
| Rare / first-time (onboarding, celebration) | The delight budget lives here |

Keyboard-initiated is a disqualifier, not a judgment call. If the request fails this gate, say so plainly, offer the non-motion alternative (instant state change, static affordance), and stop.

### 2. What is the purpose?

Name it in one of these words before continuing:

- Feedback — confirming the interface heard the user
- Spatial consistency — showing where something came from or went
- State indication — making a state change legible
- Preventing a jarring change — bridging content that would teleport
- Explanation — demonstrating how something works (marketing/onboarding only)
- Delight — allowed only at the rare/first-time tier

Can't name it? Don't build it. Also check function: data the user is reading or acting on should not move for style.

### 3. Pick the tool — cheapest that works

| Need | Tool |
|---|---|
| Hover, press, color, class/attribute state toggle | CSS transition |
| Entry animation on mount, no JS state | CSS `@starting-style` |
| Predetermined motion that must stay smooth while the page loads | CSS animation (off the main thread) |
| Programmatic control with CSS performance, no library | WAAPI (`element.animate()`) |
| Springs, layout/exit animations, gesture-driven values | Motion (`motion.dev`) |

CSS beats JS under load — rAF-based animation drops frames while the browser loads, scripts, or paints. Predetermined motion → CSS; dynamic and interruptible → JS.

If the task needs a component (toast, drawer, command menu, dropdown) rather than an animation, stop and invoke `pick-ui-library`. Hand-rolling is how you get a `<div>` dropdown with no focus management.

### 4. Pick the properties

- `transform` and `opacity` only. They skip layout and paint, run on the GPU. `width`/`height`/`margin`/`padding`/`top`/`left` trigger all three. (`clip-path` is the sanctioned fourth; `height` is tolerated only for accordions.)
- Never `scale(0)`. Start from `scale(0.9–0.97)` + `opacity: 0`. Nothing appears from nothing.
- `transform-origin` at the trigger for popovers/dropdowns/menus/tooltips (`var(--transform-origin)` in Base UI). Modals are exempt — not trigger-anchored, stay centered.
- Percentages in `translate()` are relative to the element's own size — prefer over hardcoded pixels.
- In Motion, use the full transform string. `x`/`y`/`scale` shorthands drop frames under load:

```jsx
<motion.div animate={{ transform: "translateX(100px)" }} />  // hardware accelerated
```

- Never drive a child's transform from a CSS variable on the parent — it recalculates styles for every child. Set `transform` on the element directly.

### 5. Easing and duration — or a spring

| Situation | Easing |
|---|---|
| Entering or exiting | `ease-out` |
| Moving / morphing on screen | `ease-in-out` |
| Hover / color change | `ease` |
| Constant motion (marquee, progress) | `linear` |
| Default | `ease-out` |

Never `ease-in` on UI. It delays the exact moment the user is watching. Built-ins are too weak — use:

```css
--ease-out: cubic-bezier(0.23, 1, 0.32, 1);        /* strong ease-out for UI */
--ease-in-out: cubic-bezier(0.77, 0, 0.175, 1);    /* strong, for on-screen movement */
--ease-drawer: cubic-bezier(0.32, 0.72, 0, 1);     /* iOS-like drawer curve (Ionic) */
```

Need another curve? Take it from [easing.dev](https://easing.dev/) or [easings.co](https://easings.co/). Don't hand-roll.

| Element | Duration |
|---|---|
| Button press feedback | 100–160ms |
| Tooltips, small popovers | 125–200ms |
| Dropdowns, selects | 150–250ms |
| Modals, drawers | 200–500ms |
| Marketing / explanatory | Can be longer |

UI stays under 300ms without a reason. A 180ms dropdown feels more responsive than a 400ms one.

Spring instead when the motion is drag with momentum, should feel alive, can be interrupted or reversed, or tracks the mouse: `{ type: "spring", duration: 0.5, bounce: 0.2 }` (Apple-style) or `{ type: "spring", mass: 1, stiffness: 100, damping: 10 }` (physics). Bounce 0.1–0.3, and reserve it for drag-to-dismiss and playful interactions — avoid bounce in most UI.

### 6. Interruption and exit

- Transitions, not keyframes, for anything triggered rapidly — transitions retarget from the current value; keyframes restart from zero.
- Springs for gestures — they carry velocity through an interruption.
- Exit the way it entered — symmetric paths make swipe-to-dismiss feel obvious.
- Asymmetric timing where the user is deciding — slow on the deliberate phase (hold-to-confirm press: 2s linear), snappy on the response (release: 200ms ease-out).

### 7. Reduced motion and pointer gating

Ships with the animation, every time.

```css
@media (prefers-reduced-motion: reduce) {
  .element { animation: fade 0.2s ease; } /* keep opacity/color, drop transform motion */
}

@media (hover: hover) and (pointer: fine) {
  .element:hover { transform: scale(1.05); } /* touch fires false hovers on tap */
}
```

```jsx
const reduce = useReducedMotion();
const closedX = reduce ? 0 : '-100%';
```

Reduced motion means fewer and gentler, not zero — keep comprehension-aiding transitions, remove movement.

## Recipes

Button press, dropdown, tooltip, modal, drawer, toast, accordion, stagger, hold-to-confirm, tab indicator, scroll reveal, drag-to-dismiss — see [RECIPES.md](RECIPES.md). Load it when the request matches one of those; start from the recipe, not a blank file.

## Never ship

Self-check before finishing — each is an automatic block in `review-animations`:

| Never | Instead |
|---|---|
| `transition: all` | Name the exact properties |
| `transform: scale(0)` entrance | `scale(0.95)` + `opacity: 0` |
| `ease-in` on a UI element | `ease-out` or a strong custom curve |
| Built-in `ease-out` on a deliberate animation | `cubic-bezier(0.23, 1, 0.32, 1)` |
| Animation on a keyboard shortcut / 100+/day action | No animation |
| UI duration over 300ms with no reason | 150–250ms |
| `transform-origin: center` on a trigger-anchored popover | `var(--transform-origin)` (modals exempt) |
| Keyframes on toasts, toggles, rapidly-triggered elements | CSS transitions |
| Animating `width`/`height`/`margin`/`padding`/`top`/`left` | `transform` / `opacity` |
| Motion `x`/`y`/`scale` props under load | Full `transform` string |
| Ungated `:hover` motion | `@media (hover: hover) and (pointer: fine)` |
| Missing `prefers-reduced-motion` | Gentler variant, not zero |
| Everything entering at once | 30–80ms stagger |

## Output

Write the code. Then, in a few lines at most:

- Gate result — frequency tier, named purpose; if something was rejected, say which and why.
- Ingredients — tool, properties, curve, duration/spring config, one line each.
- Feel-check — what can't be judged from code (a crossfade, spring bounce, entering-list balance), and how to check it: 2–5× duration playback or the DevTools animation inspector, frame-by-frame, on a real device, again the next day.

Don't pad it into a report — the code is the deliverable.

## Tone

Opinionated and brief. When the honest answer is "this shouldn't animate," give it — that answer is the reason this skill exists. When feel can't be settled from code, say so instead of guessing at a value.
