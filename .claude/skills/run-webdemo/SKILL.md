---
name: run-webdemo
description: Launch, verify and stop the MiniCPM-OCR Web Demo locally on macOS (Apple Silicon, no CUDA) - Ollama recognition service on :18499, FastAPI backend on :8966, Vite frontend on :5173. Use when asked to run/start/launch/stop the app or web demo, or to check a change in the real UI.
---

# Run the MiniCPM-OCR Web Demo on macOS

Verified on an Apple M4 Pro (macOS, 24 GB RAM), Ollama 0.40.16, Node 25, uv-managed Python 3.12.
`docs/webdemo_deploy.md` targets Linux + NVIDIA; this skill holds the macOS deltas.
Run everything from the repo root. Use **bash** for multi-step scripts (zsh does not word-split `$var`, which broke a download loop once).

Stack:

| Service | Port | Notes |
|---|---|---|
| Ollama, dedicated instance | `127.0.0.1:18499` | model `minicpm-ocr`, store in `models/ollama` |
| Backend (uvicorn `server:app`) | `:8966` | layout on CPU, `OCR_ENGINE_TYPE=ollama` |
| Frontend (Vite) | `:5173` | `VITE_OCR_TARGET=http://127.0.0.1:8966` |

Logs go to the session scratchpad (`$S` below); set `S` to it before starting.

## 1. Prerequisites (check, only fix what is missing)

```bash
ollama --version            # must be >= 0.40.12 (Modelfile uses DRAFT / REQUIRES)
ls .minicpm-ocr/bin/python  # SDK venv
ls models/MiniCPM-OCR-1.0-gguf/*.gguf models/PP-DocLayoutV3_infer models/paddle_upright
ls webdemo/frontend/node_modules >/dev/null
```

- **Ollama too old**: it is the desktop app (`/Applications/Ollama.app`). Ask the user to update it (menu bar icon → update, or ollama.com/download). Do not reinstall it yourself.
- **Venv missing** (CPU paddle, not `paddlepaddle-gpu`; the module imports as `minicpmvocr`):
  ```bash
  uv venv .minicpm-ocr --python 3.12 --seed
  .minicpm-ocr/bin/python -m pip install "minicpm-ocr[all]" "paddlepaddle==3.3.0" fastapi uvicorn python-multipart
  ```
- **GGUF missing** (~2 GB, fast from Hugging Face):
  ```bash
  bash -c 'd=models/MiniCPM-OCR-1.0-gguf; mkdir -p $d
  curl -fsSL --retry 3 -o $d/MiniCPM-OCR-1_0-Q8_0.gguf https://huggingface.co/openbmb/MiniCPM-OCR-1.0-gguf/resolve/main/MiniCPM-OCR-1_0-Q8_0.gguf
  curl -fsSL --retry 3 -o $d/mmproj-model-f16-4x.gguf https://huggingface.co/openbmb/MiniCPM-OCR-1.0-gguf/resolve/main/mmproj-model-f16-4x.gguf
  curl -fsSL --retry 3 -o $d/dspark-MiniCPM-OCR-1_0-Q8_0.gguf https://huggingface.co/openbmb/MiniCPM-OCR-1.0-DSpark-gguf/resolve/main/dspark-MiniCPM-OCR-1_0-Q8_0.gguf'
  ```
- **Paddle models missing** (bcebos server is slow, ~25-30 KB/s; run in background):
  ```bash
  bash webdemo/backend/prepare_upright_models.sh
  curl -fsSL --retry 3 https://paddle-model-ecology.bj.bcebos.com/paddlex/official_inference_model/paddle3.0.0/PP-DocLayoutV3_infer.tar | tar -x -C models
  ```
- **node_modules missing**: `(cd webdemo/frontend && corepack pnpm install)`. The "Ignored build scripts: esbuild" warning is harmless.

## 2. Start

```bash
# Ollama (dedicated instance; the desktop app on :11434 stays untouched)
OLLAMA_HOST=127.0.0.1:18499 OLLAMA_MODELS="$PWD/models/ollama" OLLAMA_NUM_PARALLEL=1 OLLAMA_MAX_LOADED_MODELS=1 \
  nohup ollama serve > "$S/ollama.log" 2>&1 &
sleep 3
```

Register the model only if `OLLAMA_HOST=127.0.0.1:18499 ollama list` lacks `minicpm-ocr`:

```bash
cat > models/MiniCPM-OCR-1.0-gguf/Modelfile <<'EOF'
FROM ./MiniCPM-OCR-1_0-Q8_0.gguf
FROM ./mmproj-model-f16-4x.gguf
DRAFT ./dspark-MiniCPM-OCR-1_0-Q8_0.gguf
REQUIRES 0.40.12
PARAMETER num_ctx 8192
PARAMETER temperature 0
PARAMETER draft_num_predict 15
EOF
OLLAMA_HOST=127.0.0.1:18499 ollama create minicpm-ocr -f models/MiniCPM-OCR-1.0-gguf/Modelfile
OLLAMA_HOST=127.0.0.1:18499 ollama show minicpm-ocr | sed -n '/Capab/,/^$/p'   # must list "vision"
```

Backend (CPU layout, small crop pool, page-ordered streaming):

```bash
source .minicpm-ocr/bin/activate
OCRWEB_PYTHON=python OCR_ENGINE_TYPE=ollama OCR_SERVER_URL=http://127.0.0.1:18499 OCR_SERVED_NAME=minicpm-ocr \
OCR_LAYOUT_MODEL_DIR="$PWD/models/PP-DocLayoutV3_infer" OCR_LAYOUT_DEVICE=cpu OCR_LAYOUT_DEVICES=cpu OCR_LAYOUT_WORKERS=1 \
OCR_SHARED_CROP_POOL=0 OCR_CROP_WORKERS=8 PORT=8966 \
  nohup bash webdemo/backend/start_local_worker.sh > "$S/backend.log" 2>&1 &
for i in $(seq 1 40); do curl -sf http://127.0.0.1:8966/health && break; sleep 3; done   # {"status":"ok"}
```

Frontend:

```bash
(cd webdemo/frontend && VITE_OCR_TARGET=http://127.0.0.1:8966 \
  nohup corepack pnpm dev --host --port 5173 --strictPort > "$S/frontend.log" 2>&1 &)
for i in $(seq 1 20); do curl -sf -o /dev/null http://localhost:5173/playground/document-parse && echo UP && break; sleep 1; done
```

## 3. Verify end to end (don't stop at "it launched")

With the Playwright MCP tools:

1. Navigate to `http://localhost:5173/playground/document-parse`.
2. A "Choose a platform sample" tour dialog opens: click **Close**. The sample buttons load precomputed results and don't exercise the backend.
3. Click `section[aria-label="Original"]` to get a file chooser, then upload `assets/minicpm-ocr/french_eval_contract.jpg`.
4. Wait for "Parsing..." to disappear (~30 s on first request, the model loads). Expected text: *Comme indiqué dans les conditions particulières de mon contrat d'assurance*.
5. `grep -i draft "$S/ollama.log"` should show `draft-dspark` statistics (DSpark active).

Playwright writes `.playwright-mcp/` screenshots/logs into the working directory; delete them when done.

## 4. Stop

Kill by port, so the Ollama desktop app (`/Applications/Ollama.app/.../ollama serve` on :11434) keeps running. Never `pkill -f "ollama serve"`.

```bash
for p in 18499 8966 5173; do lsof -tiTCP:$p -sTCP:LISTEN | xargs -r kill; done
pkill -f "pnpm dev --host --port 5173" || true
lsof -nP -iTCP:18499 -iTCP:8966 -iTCP:5173 -sTCP:LISTEN || echo "ports free"
```
