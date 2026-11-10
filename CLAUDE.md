# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

**WebAI / "sala.company Web AI Chat"** (https://webai.sala.company) is a privacy-first chat app that runs open-source
LLMs entirely in the browser with [WebLLM](https://github.com/mlc-ai/web-llm). The real app is a **single
self-contained `index.html`**: inline CSS plus one inline `<script type="module">` that imports
`https://esm.run/@mlc-ai/web-llm@0.2.52`. Model weights are downloaded by WebLLM from Hugging Face on first load;
chats never leave the browser.

Important: the repo also contains a Lovable-generated Vite + React + TypeScript + shadcn/ui + Tailwind scaffold
(`package.json`, `src/`, `vite.config.ts`, `tailwind.config.ts`, `components.json`, etc.). **It is unused by the live
site**: `index.html` has no `#root` element and does not load `src/main.tsx`, and `src/pages/Index.tsx` is still the
"Welcome to Your Blank App" placeholder. Make app changes in `index.html` unless deliberately migrating to React.

## Commands

The live app needs no build. Serve the repo root statically:

```bash
python3 -m http.server 8000   # then open http://localhost:8000
```

npm scripts that exist for the (unused) React scaffold (`package-lock.json` and `bun.lockb` are both committed):

```bash
npm install
npm run dev        # vite dev server on port 8080 (host "::")
npm run build      # vite build -> dist/ (gitignored)
npm run build:dev  # development-mode build (enables lovable-tagger)
npm run lint       # eslint .
npm run preview
```

There are no tests. Note `vite build` would process the same `index.html`; it is not part of deployment.

## Architecture (index.html)

- UI: model `<select id="model">`, `Load` button, progress bar/phase/error status, `#chat` message list, textarea
  `#input` (Enter sends, Shift+Enter newline), `Send`/`Stop` buttons. Responsive rule at `max-width: 640px`.
- Model registry: `webllm.prebuiltAppConfig` (handled whether it is an object or a function), filtered by
  `getRepresentativeModels()` to one model per family (Llama, Phi, Mistral, Qwen, Gemma, TinyLlama) using per-family
  priority lists of model IDs, with regex fallback, then a "first 6" fallback.
- Loading: `webllm.CreateMLCEngine(modelId, { appConfig, initProgressCallback })`, first with `device: 'wasm'` in the
  app config, falling back to `device: 'webgpu'`. Switching models unloads the previous engine.
- Chat: `engine.chat.completions.create({ messages, stream: true })` with an `AbortController` for Stop; streams
  tokens into the DOM via `textContent` (no HTML injection) and shows tok/s.
- `localStorage` keys: `webllm.model.v1` (last loaded model) and `webllm.chat.v1` (full message history).
- `<meta http-equiv>` Cross-Origin-Opener-Policy / Cross-Origin-Embedder-Policy tags are present in `<head>`.

## Deployment

There is no `.github/workflows` directory and no `.nojekyll`; GitHub Pages serves the repository branch directly
(`main` is the only branch), so the root `index.html` is what is live. Custom domain via `CNAME`
(`webai.sala.company`). **Pushing to `main` publishes live.**

## Conventions / gotchas

- **GoatCounter** analytics snippet at the end of `index.html` (endpoint `https://salacompany.goatcounter.com/count`,
  skipped on localhost, paths prefixed with `location.host`) – keep it intact. The README's privacy section describes
  exactly what goes over the network (library from jsDelivr/esm.run, weights from Hugging Face, GoatCounter visit
  count); keep it accurate if network behavior changes.
- The WebLLM version is pinned in the import URL (`@0.2.52`); model IDs in `getRepresentativeModels()` must exist in
  that version's prebuilt registry. Changing the version may change available IDs.
- Bumping the localStorage key suffix (`.v1`) is the way to invalidate stored chats/model if their shape changes.
- MIT licensed; keep the copyright header comment in `index.html`.
