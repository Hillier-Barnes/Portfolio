# Scroll animation: GSAP Observer "continuous paged sections"

A reusable recipe for a full-screen, **one-section-per-gesture** experience: the
page doesn't scroll — wheel/trackpad/swipe/arrow intent is captured by GSAP's
**Observer** plugin and each gesture transitions to the next section with a
**masked reveal** (one section wipes in over the previous), a content parallax,
a **character-stagger** on the heading, and an optional **continuous loop**.

Based on GreenSock's "Animated Continuous Sections" demo, generalised with the
adaptations and bug-fixes learned from putting it on a real, content-rich site.

---

## When to use it

- A short, sectioned site (3–7 "slides": hero, work, about, contact…).
- You want a deliberate, presentation-like feel, not free scrolling.
- **Caveat:** each section must fit one viewport. If a section's content is
  taller than the screen (common on mobile), paging will crop it — so gate this
  to desktop and fall back to normal scroll on small screens (see below).

## Dependencies

```bash
npm i gsap   # gsap >= 3.12 includes Observer (free)
```

```js
import gsap from 'gsap';
import { Observer } from 'gsap/Observer';
gsap.registerPlugin(Observer);
```

No SplitText needed — the heading split is done with a tiny helper (below).

---

## 1. Markup

Each section is full-screen. The content lives inside **two nested mask
wrappers** (`.outer` / `.inner`). Put anything you want revealed — crucially the
**background** — inside `.inner`.

```html
<main>
  <section class="section" data-id="home">
    <div class="outer"><div class="inner">
      <div class="bg"><!-- bg image/colour lives here, see CSS -->
        <h2 class="section-heading">Home</h2>
        <!-- ...the rest of the section content... -->
      </div>
    </div></div>
  </section>
  <section class="section" data-id="work"> … </section>
  <!-- more sections -->
</main>
```

If your sections already exist with their own content, you can **wrap them in
JS** instead of editing every section (see "Adapting to existing markup").

## 2. CSS

```css
html, body { height: 100%; overflow: hidden; }     /* no native scroll */
main { position: fixed; inset: 0; }

.section { position: absolute; inset: 0; visibility: hidden; } /* gsap autoAlpha shows the active one */
.outer, .inner { position: absolute; inset: 0; width: 100%; height: 100%; overflow: hidden; }
.inner { display: flex; align-items: center; justify-content: center; }

/* ⚠️ The section BACKGROUND must be on a layer INSIDE the mask (.bg / .inner),
   NOT on .section. Otherwise the incoming section's opaque background covers the
   outgoing one instantly and it "disappears" instead of wiping in. */
.bg { position: absolute; inset: 0; width: 100%; height: 100%;
      background-size: cover; background-position: center; }

.pg-char { display: inline-block; white-space: pre; will-change: transform, opacity; }
```

## 3. The controller

```js
function initPager() {
  const sections  = gsap.utils.toArray('.section');
  const outers    = sections.map(s => s.querySelector('.outer'));
  const inners    = sections.map(s => s.querySelector('.inner'));
  const bgs       = sections.map(s => s.querySelector('.bg'));        // for parallax
  const headings  = sections.map(s => s.querySelector('.section-heading'));

  // Split each heading into characters (preserves <br>).
  const splitChars = (el) => {
    if (!el) return [];
    const out = [];
    const walk = (node) => [...node.childNodes].forEach((c) => {
      if (c.nodeType === 3) {                                  // text node
        const frag = document.createDocumentFragment();
        for (const ch of c.textContent) {
          const s = document.createElement('span');
          s.className = 'pg-char'; s.textContent = ch;
          frag.appendChild(s); out.push(s);
        }
        node.replaceChild(frag, c);
      } else if (c.nodeType === 1 && c.tagName !== 'BR') walk(c);
    });
    walk(el);
    return out;
  };
  const charsBy = headings.map(splitChars);

  // initial mask offsets
  gsap.set(outers, { yPercent: 100 });
  gsap.set(inners, { yPercent: -100 });
  gsap.set(sections, { autoAlpha: 0 });

  let currentIndex = -1;
  let animating = false;
  const wrap = gsap.utils.wrap(0, sections.length);             // continuous loop

  function gotoSection(index, direction) {
    index = wrap(index);
    if (animating || index === currentIndex) return;
    animating = true;
    const dFactor = direction === -1 ? -1 : 1;                  // direction-aware
    const tl = gsap.timeline({
      defaults: { duration: 1.15, ease: 'power1.inOut' },
      onComplete: () => { animating = false; },
    });

    // animate the OUTGOING section out, then hide it
    if (currentIndex >= 0) {
      gsap.set(sections[currentIndex], { zIndex: 0 });
      tl.to(bgs[currentIndex], { yPercent: -15 * dFactor }, 0)
        .set(sections[currentIndex], { autoAlpha: 0 });
    }

    // reveal the INCOMING section: outer slides up, inner slides down (mask),
    // bg parallaxes in, heading chars stagger in
    gsap.set(sections[index], { autoAlpha: 1, zIndex: 1 });
    tl.fromTo([outers[index], inners[index]],
        { yPercent: (i) => (i ? -100 * dFactor : 100 * dFactor) },
        { yPercent: 0 }, 0)
      .fromTo(bgs[index], { yPercent: 15 * dFactor }, { yPercent: 0 }, 0)
      .fromTo(charsBy[index],
        { autoAlpha: 0, yPercent: 120 * dFactor },
        { autoAlpha: 1, yPercent: 0, duration: 1, ease: 'power2',
          stagger: { each: 0.02, from: 'random' } }, 0.15);

    currentIndex = index;
  }

  // ---- input ----
  Observer.create({
    type: 'wheel,touch',         // omit 'pointer' if the page has clickable UI (see gotchas)
    wheelSpeed: -1,
    onUp:   () => !animating && gotoSection(currentIndex + 1, 1),
    onDown: () => !animating && gotoSection(currentIndex - 1, -1),
    tolerance: 10,
    preventDefault: true,
  });

  window.addEventListener('keydown', (e) => {
    if (['ArrowDown', 'PageDown', ' '].includes(e.key)) { e.preventDefault(); gotoSection(currentIndex + 1, 1); }
    else if (['ArrowUp', 'PageUp'].includes(e.key))     { e.preventDefault(); gotoSection(currentIndex - 1, -1); }
  });

  gotoSection(0, 1);             // show the first section
}

initPager();
```

That's the whole effect. The rest is adaptation.

---

## Key mechanics

- **The mask reveal.** `outer` and `inner` start offset in *opposite* directions
  (`+100` / `-100`). Animating both to `0` keeps the content visually anchored
  while the clip-window wipes across — a clean reveal with no visible sliding of
  the content itself.
- **Direction-aware.** `dFactor` flips every offset so scrolling up reveals from
  the top and scrolling down from the bottom.
- **Continuous loop.** `gsap.utils.wrap(0, n)` makes `gotoSection(n)` → `0` and
  `gotoSection(-1)` → `n-1`. Drop it (clamp the index) if you don't want looping.
- **Gesture gating.** `animating` ignores input mid-transition; Observer's
  `tolerance` debounces a single gesture into one step.

---

## Adapting to existing / content-rich markup

You rarely have the demo's tidy `bg + heading` sections. To apply it to real
sections without rewriting them:

1. **Wrap in JS.** For each `.section`, create `.outer`/`.inner` and move the
   existing content node into `.inner`:
   ```js
   sections.forEach((sec) => {
     const content = sec.querySelector('.content');     // your existing wrapper
     const outer = Object.assign(document.createElement('div'), { className: 'outer' });
     const inner = Object.assign(document.createElement('div'), { className: 'inner' });
     outer.appendChild(inner); inner.appendChild(content);
     sec.prepend(outer);
   });
   ```
2. **Move the background onto the masked layer.** If your section colour/texture
   is on `.section`, make `.section` transparent in paged mode and put the colour
   on `.inner` (or a `.bg`). This is the #1 thing people get wrong — see gotchas.
3. **Parallax target.** Use whatever full-bleed element you have as the `bg`
   parallax target; skip that tween if there isn't one.

## Desktop-only + mobile fallback (recommended)

Paging crops tall content, so only page on wide screens:

```js
// set <html class="paged"> before paint when wide & motion-OK
if (matchMedia('(min-width:1024px)').matches &&
    !matchMedia('(prefers-reduced-motion: reduce)').matches) {
  document.documentElement.classList.add('paged');
}
```
Guard the CSS (`html.paged …`) and `if (!document.documentElement.classList.contains('paged')) return;`
at the top of `initPager`. On narrow screens / reduced-motion, leave the page as
a normal scrolling document. Reload on crossing the breakpoint:
```js
addEventListener('resize', () => { /* debounce */ if (!matchMedia('(min-width:1024px)').matches) location.reload(); });
```

## Wiring navigation

With no scroll, a navbar/anchor links must page instead of jump:

```js
const ids = sections.map(s => s.dataset.id);
document.querySelectorAll('a[href^="#"]').forEach((a) => {
  const idx = ids.indexOf(a.getAttribute('href').slice(1));
  if (idx === -1) return;
  a.addEventListener('click', (e) => {
    e.preventDefault();
    if (!animating) gotoSection(idx, idx > currentIndex ? 1 : -1);
  });
});
```
Drive any "active link" / scroll-spy state from inside `gotoSection` instead of
scroll position.

## Optional: intro pre-roll

Run a loader/curtain first, then call `gotoSection(0, 1)` from its `onComplete`.
If you add a safety timeout to force-start, **guard it** (`if (currentIndex === -1) …`)
or it will yank the user back to section 0 later (real bug I hit).

---

## Gotchas (learned the hard way)

1. **Background must be inside the mask.** If the section's opaque background is
   on the outer `.section`, the incoming section instantly hides the outgoing one
   ("the page being left disappears"). Put the background on `.inner`/`.bg`.
2. **Curtain/intro safety timer.** Any `setTimeout` that calls `gotoSection(0)`
   as a fallback must check `currentIndex === -1` first — otherwise it fires
   seconds later and snaps you back to the first section mid-navigation.
3. **`type: 'pointer'` vs clickable UI.** Observer's `pointer` type with
   `preventDefault` can swallow clicks on links/buttons. If the page has
   interactive elements, use `type: 'wheel,touch'` and rely on keys for the rest.
4. **ScrollTrigger doesn't mix well here.** If a child also uses ScrollTrigger to
   reveal on scroll, note that ScrollTrigger **mis-measures elements inside a
   CSS-`transform`-scaled container** (and there's no scroll anyway). Prefer
   driving child reveals from `gotoSection` directly.
5. **Stagger lengthens the timeline.** `onComplete` (where `animating=false`)
   fires after the *last* staggered char, which can be well past `duration`.
   That's fine for users, but account for it if you script/QA timing.
6. **reduced-motion.** Don't enter paged mode under
   `prefers-reduced-motion: reduce`; leave a normal scrolling document.
7. **Testing with synthetic events.** Observer often ignores a single
   JS-dispatched `WheelEvent`; drive real `Input.dispatchMouseEvent`
   (`type:'mouseWheel'`) / key events when automating, and space samples beyond
   the full (stagger-extended) transition time.

---

## Framework notes

- **Astro / Vite:** put the controller in a processed (non-`is:inline`)
  `<script>` so `import gsap` works; it runs deferred (DOM ready).
- **React:** run `initPager()` in a `useEffect` (empty deps); return a cleanup
  that `Observer.getAll().forEach(o => o.kill())` and `gsap` kills if you remount.
- **Char split:** swap the helper for GSAP **SplitText** (`new SplitText(el,
  {type:'chars'}).chars`) if you already use it — identical result.
