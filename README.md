# wllama4qwen35

This repository includes a recent `wllama` build output that works with Qwen3.5 GGUF models, plus a single-page verification app with a Transformers.js/ONNX Runtime path for compatible Qwen3.5 and causal-language-model ONNX repositories.

Date baseline: `2026-03-03`.

## Live demo (Vercel)

- Demo URL: `https://wllama4qwen35.vercel.app`

This is the recommended way to test the web app in browser.
The Vercel deployment is configured with:

- `Cross-Origin-Opener-Policy: same-origin`
- `Cross-Origin-Embedder-Policy: require-corp`

## What is included

- `index.html`
- `vendor/wllama.local.esm.js`
- `single-thread/wllama.wasm`
- `multi-thread/wllama.wasm`
- `vendor/onnxruntime-web/*`
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

For GGUF/wllama usage, copy these 3 artifacts into your own project and use them from the same origin:

- `vendor/wllama.local.esm.js`
- `single-thread/wllama.wasm`
- `multi-thread/wllama.wasm`

For the ONNX backend, also copy `vendor/onnxruntime-web/*` and preserve that directory layout.

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

### ONNX URLs in the wllama Chat tab

The **Model URL** field in the `wllama Chat` tab automatically selects the ONNX backend when either:

- the Hugging Face model repository name ends in `-ONNX`, or
- the URL points directly to a `.onnx` file inside a Hugging Face model repository.

Examples:

```text
https://huggingface.co/onnx-community/Qwen3.5-0.8B-ONNX
https://huggingface.co/onnx-community/Qwen3.5-0.8B-ONNX/resolve/main/onnx/decoder_model_merged_q4.onnx
https://huggingface.co/acidsound/LFM2.5-2.6B-Uncensored-ONNX
https://huggingface.co/acidsound/LFM2.5-2.6B-Uncensored-ONNX/resolve/main/onnx/model_q4f16.onnx
```

A direct `.onnx` URL is normalized to its parent model repository because chat inference also requires the repository's tokenizer, config, and companion graph files. Do not enter the `.onnx_data` URL separately; Transformers.js downloads external data files according to `transformers.js_config.use_external_data_format` in `config.json`. Arbitrary standalone `.onnx` files and non-Hugging-Face URLs are not supported.

The loader reads `model_type` and uses the Qwen3.5 conditional-generation class or generic `AutoModelForCausalLM` as appropriate. Decoder-only models try the explicitly requested dtype, WebGPU `q4f16`, WebGPU `q4`, and then WASM `q4`. The `n_ctx` and `n_batch` controls apply only to GGUF/wllama models. **Max Output Tokens** is shared by both backends: it maps to `max_new_tokens` for ONNX and `nPredict` for GGUF, and can be adjusted from 1 to 32768 tokens (subject to the model context window and available browser memory).

For one external data file named `model_q4f16.onnx_data`, its `config.json` entry must declare one chunk:

```json
{
  "transformers.js_config": {
    "use_external_data_format": {
      "model_q4f16.onnx": 1
    }
  }
}
```

### Thinking toggle with LFM2.5

`LFM2.5-2.6B` is an always-thinking checkpoint: its original template unconditionally opens `<think>` and does not implement `enable_thinking`. When **Enable thinking mode** is unchecked, this app renders the original conversation without a generation prompt and appends a short, already-closed thinking block before generation:

```text
<|im_start|>assistant
<think>
No unnecessary reasoning. Close thinking and answer immediately.
</think>
```

This is a template-level compatibility mode, not a change to the model weights. It encourages a direct answer but cannot guarantee that the model will never emit additional reasoning. When the toggle is checked, the original always-thinking template is used unchanged.

## Threading and runtime notes

- Multi-thread may fall back to single-thread depending on browser/environment.
- The app shows runtime info in `envText` after model load:
  - `runtime=multi-thread (N)` or `runtime=single-thread (N)`
  - plus `crossOriginIsolated` and hardware thread count.
- Expected warning:
  `Multi-threads are not supported in this environment, falling back to single-thread`
- On load error like `Invalid typed array length ...`, try:
  - Smaller quant (start with `Q3_K_S`)
  - Lower `n_ctx`
  - Lower `n_batch`
