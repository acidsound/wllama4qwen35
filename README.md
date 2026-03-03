# wllama4qwen35

This repository includes a recent `wllama` build output that works with Qwen3.5 GGUF models, plus a single-page verification app.

Date baseline: `2026-03-03`.

## Live demo (Vercel)

- Demo URL: `https://wllama4qwen35.vercel.app`

This is the recommended way to test the web app in browser.

## What is included

- `index.html`
- `vendor/wllama.local.esm.js`
- `single-thread/wllama.wasm`
- `multi-thread/wllama.wasm`
- `serve.mjs`

## Quick local run

```bash
node serve.mjs
```

Open:

```text
http://localhost:8080/
```

## Recommended integration (copy files)

For real usage, copy these 3 artifacts into your own project and use them from the same origin:

- `vendor/wllama.local.esm.js`
- `single-thread/wllama.wasm`
- `multi-thread/wllama.wasm`

Suggested layout in your app:

```text
your-app/
  vendor/wllama.local.esm.js
  single-thread/wllama.wasm
  multi-thread/wllama.wasm
```

Minimal usage:

```html
<script type="module">
  import { Wllama } from "./vendor/wllama.local.esm.js";

  const wllama = new Wllama({
    "single-thread/wllama.wasm": "./single-thread/wllama.wasm",
    "multi-thread/wllama.wasm": "./multi-thread/wllama.wasm",
  });

  await wllama.loadModelFromUrl(
    "https://huggingface.co/unsloth/Qwen3.5-0.8B-GGUF/resolve/main/Qwen3.5-0.8B-Q3_K_S.gguf",
    {
      n_ctx: 512,
      n_batch: 64,
      n_threads: Math.max(1, Math.min(8, navigator.hardwareConcurrency || 4)),
    }
  );

  const messages = [{ role: "user", content: "Hello Qwen3.5" }];
  const chunks = await wllama.createChatCompletion(messages, {
    stream: true,
    nPredict: 128,
    sampling: { temp: 0.7, top_p: 0.9, top_k: 40 },
  });

  for await (const chunk of chunks) {
    console.log(chunk.currentText || "");
  }
</script>
```

## Why copy is recommended

- Stable version pinning
- No cross-repo URL breakage
- Fewer CORS and MIME surprises
- Easier debugging and release management

## GitHub Pages (artifact reference only)

GitHub Pages is kept as a static reference source for build artifacts:

```text
https://acidsound.github.io/wllama4qwen35/
```

Direct file links:

```text
https://acidsound.github.io/wllama4qwen35/vendor/wllama.local.esm.js
https://acidsound.github.io/wllama4qwen35/single-thread/wllama.wasm
https://acidsound.github.io/wllama4qwen35/multi-thread/wllama.wasm
```

Use it mainly for download/reference, not as the primary interactive demo host.

## Model URL notes

Recommended model URLs:

- Fast/low RAM:
  `https://huggingface.co/unsloth/Qwen3.5-0.8B-GGUF/resolve/main/Qwen3.5-0.8B-Q3_K_S.gguf`
- Balanced:
  `https://huggingface.co/unsloth/Qwen3.5-0.8B-GGUF/resolve/main/Qwen3.5-0.8B-Q4_K_M.gguf`
- Higher quality:
  `https://huggingface.co/unsloth/Qwen3.5-0.8B-GGUF/resolve/main/Qwen3.5-0.8B-Q5_K_M.gguf`

Always use `.../resolve/main/...` URLs, not `.../blob/...`.

## Threading and runtime notes

- Multi-thread may fall back to single-thread depending on browser/environment.
- Expected warning:
  `Multi-threads are not supported in this environment, falling back to single-thread`
- On load error like `Invalid typed array length ...`, try:
  - Smaller quant (start with `Q3_K_S`)
  - Lower `n_ctx`
  - Lower `n_batch`
