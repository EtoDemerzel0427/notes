---
title: React Concurrent Rendering
slug: react-concurrent-rendering
date: 2025-12-21
tags: [React]
category: Dev
draft: false
---

Today I optimized the [Live Preview](https://github.com/EtoDemerzel0427/RectoWiki/commit/4e884ff841a829c569bdd6c14c464967e3be502c) of this wiki using a powerful capability introduced in **React 18**: **concurrent rendering**, specifically the hook `useDeferredValue`.

## TL;DR

* `useDeferredValue` marks updates as **non-urgent (lower priority)**
* React may delay, interrupt, or restart rendering based on user input
* This greatly reduces perceived jank, but does not magically remove main-thread costs

If you need *both* responsiveness and freshness, `useDeferredValue` is often the cleanest tool for the job.

## The Problem: “Janky Typing”

In a live markdown editor, we have two competing goals:

1. **Input Responsiveness**
   The user expects typed characters to appear *immediately*.

2. **Preview Freshness**
   The user also wants to see the rendered result (HTML, Math, Music) *as soon as possible*.

However, rendering the preview can be expensive — parsing markdown, processing ABC notation, or initializing audio synthesizers are all synchronous operations that run on the **main thread**. If we do too much work eagerly, typing becomes laggy and unpleasant. This becomes a non-negligible issue when I built a music wiki and started using `abc.js` to input and render standard staff notation in real time.

---

## The Old Solution: Debouncing

Traditionally, we solve this with debouncing using `setTimeout`.

```js
// Old Way
const [debouncedText, setDebouncedText] = useState(text);

useEffect(() => {
  const timer = setTimeout(() => {
    setDebouncedText(text); // Wait 300ms AFTER typing stops
  }, 300);
  return () => clearTimeout(timer);
}, [text]);
```

**Pros**

* Prevents expensive rendering from running on every keystroke
* Avoids completely freezing the UI

**Cons**

* Introduces artificial latency
* The preview is always “behind”
* If the user keeps typing, the preview may never update

Debouncing optimizes for safety, but at the cost of freshness.

---

## The Modern Solution: `useDeferredValue`

React 18 introduces **concurrent rendering**, which allows React to work on rendering updates with different priorities.

`useDeferredValue` lets us tell React:

> “This value is important, but not urgent. Update it when you have time.”

```js
import { useDeferredValue } from 'react';

const Preview = ({ text }) => {
  // Mark this value as non-urgent
  const deferredText = useDeferredValue(text);

  return (
    <>
      {/* High-priority: updates immediately */}
      <Editor text={text} />

      {/* Low-priority: may lag slightly behind */}
      <HeavyPreview text={deferredText} />
    </>
  );
};
```

---

## How It Works (Conceptually)

1. **Urgent update**
   When the user types, React updates `text` immediately. The input remains responsive.

2. **Deferred update**
   React notices that `deferredText` depends on `text`, but schedules updates that use it at a **lower priority**.

3. **Concurrent rendering**
   React starts rendering `HeavyPreview` in an interruptible way.

4. **Interruption & restart**
   If the user types again before the deferred render finishes, React **abandons the in-progress render** and retries with the newer value.

This makes the preview feel as fresh as possible *without compromising typing responsiveness*.

---

## What This Does *Not* Do (Important!)

It’s worth clarifying a few common misconceptions:

* ❌ `useDeferredValue` does **not** run work on a background thread
* ❌ It does **not** use Web Workers
* ❌ It does **not** eliminate main-thread work entirely

All rendering still happens on the main thread.
What React concurrency gives us is **prioritization and interruptibility**, not parallel JavaScript execution.

If `HeavyPreview` performs very expensive synchronous work, it can still block the main thread once it runs. In such cases, further optimization or offloading (e.g. Web Workers) may still be necessary.

---

## The Result

* **Responsive typing**
  Urgent updates are prioritized, so keystrokes remain smooth.

* **Best-possible freshness**
  The preview updates as quickly as the scheduler allows, without artificial delays.

* **Graceful degradation**
  On fast machines, updates may appear almost instant.
  On slower devices, the preview may lag slightly — but the UI remains usable.

You can think of this as:

> **Debouncing driven by render priority instead of timers.**

---

## Further Reading

* [React Docs: `useDeferredValue`](https://react.dev/reference/react/useDeferredValue)
* [A Visual Guide to React 18 Concurrent Rendering](https://react.dev/blog/2022/03/29/react-v18)

