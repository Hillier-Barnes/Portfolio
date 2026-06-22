# Portfolio — Sosaiah Hillier-Barnes

An editorial portfolio site built with [Astro](https://astro.build/), translated from a Figma design.

Five scrolling sections — Landing, Work, About, Personal, Contact — with a paper-grain texture, horizontal slide-in parallax, a typewriter hero, and a loader-curtain intro (GSAP).

## Develop

```bash
npm install
npm run dev      # http://localhost:4321
```

## Build

```bash
npm run build    # outputs to dist/
npm run preview  # preview the production build
```

## Stack

- Astro 5
- GSAP + ScrollTrigger (scroll effects, intro)
- Vanilla CSS (design tokens in `src/styles/global.css`)
