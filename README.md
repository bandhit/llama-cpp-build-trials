# llama.cpp Build and Run

Instructions for building [llama.cpp](https://github.com/ggml-org/llama.cpp) from source, running a model via `llama-server`, and wiring it into VS Code.

## Prerequisites

Install the required system packages:

```bash
sudo apt update
sudo apt install -y git build-essential cmake
```

## Build

The upstream [llama.cpp](https://github.com/ggml-org/llama.cpp) source lives in [`main/`](main) as a git submodule pinned to a known-good commit. Clone this repo with submodules, then build with CMake:

```bash
# On first clone
git clone --recurse-submodules git@github.com:bandhit/llama-cpp-build-trials.git
cd llama-cpp-build-trials

# Or, if you already cloned without --recurse-submodules
git submodule update --init --recursive
```

```bash
cd main
mkdir build
cd build
cmake ..
cmake --build . --config Release
```

To bump to a newer llama.cpp commit later:

```bash
cd main && git fetch && git checkout <commit-or-tag>
cd .. && git add main && git commit -m "Bump llama.cpp submodule"
```

### GPU acceleration (optional)

The default build above is **CPU-only** — `-ngl` will be ignored at runtime and large models will be very slow. Rebuild with GPU support if you have one:

```bash
cd ~/workspaces/llama_cpp/main
rm -rf build && mkdir build && cd build

# NVIDIA (needs the CUDA toolkit installed)
cmake .. -DGGML_CUDA=ON

# AMD (needs ROCm)
# cmake .. -DGGML_HIP=ON

# Cross-vendor via Vulkan (works on most modern GPUs)
# cmake .. -DGGML_VULKAN=ON

cmake --build . --config Release -j
```

## Run

Launch `llama-server` with a Hugging Face model:

```bash
cd bin
./llama-server -hf unsloth/Qwen3.6-27B-MTP-GGUF:UD-Q4_K_XL --jinja --port 8080 -ngl 99
```

### Flags

| Flag | Description |
| --- | --- |
| `-hf <repo>:<quant>` | Fetch a model from Hugging Face by repo and quant tag. Shorthand for `--hf-repo <repo> --hf-file <auto-picked-file>`. |
| `--hf-repo <repo>` / `--hf-file <file>` | Explicit repo + file pair. Use this when the repo has multiple GGUFs and `-hf` can't guess, e.g. community fine-tunes. |
| `--jinja` | Use the model's Jinja chat template for prompt formatting. |
| `--port 8080` | Serve the HTTP API on port 8080. |
| `-ngl 99` | Offload up to 99 layers to the GPU (use a large number to offload all). |
| `-c 65536` | Set context window to 65536 tokens (default is model-dependent; larger `-c` uses more VRAM). |
| `CUDA_VISIBLE_DEVICES=0` | Env var: pin the process to GPU 0 on multi-GPU systems. Use `0,1` for multiple, or leave unset to use all visible GPUs. |

Once running, the OpenAI-compatible API is available at `http://localhost:8080/v1`.

> **Security note:** `llama-server` defaults to CORS-open and no API key. Fine on localhost — do not expose port 8080 to your network without setting `--api-key`.

### Examples

**Qwen3.6-27B on GPU 0 with a 64K context window:**

```bash
CUDA_VISIBLE_DEVICES=0 llama-server \
  -hf unsloth/Qwen3.6-27B-MTP-GGUF:UD-Q4_K_XL \
  --jinja --port 8080 -ngl 99 -c 65536
```

**Fable Fusion 27B (community fine-tune, uncensored) — explicit file selection:**

```bash
CUDA_VISIBLE_DEVICES=0 llama-server \
  --hf-repo DavidAU/Qwen3.6-27B-Fable-Fusion-711-Uncensored-Heretic-NM-DAU-NEO-MAX-MTP-GGUF \
  --hf-file Qwen3.6-27B-Fable-Fus-711-UnHeretic-NM-DAU-NEO-MAX-NEO-MTP-Q4_K_M.gguf \
  --jinja --port 8080 -ngl 99 -c 65536
```

These call `llama-server` directly (from `PATH` if installed system-wide) instead of `./llama-server` from `build/bin/` — swap in whichever path applies to your setup.

## Use from VS Code — Chat via Continue

Opening this repo in VS Code will prompt you to install the recommended extension listed in [`.vscode/extensions.json`](.vscode/extensions.json) — [Continue](https://marketplace.visualstudio.com/items?itemName=Continue.continue), a general chat + inline-edit extension that speaks the OpenAI Chat Completions protocol, which `llama-server` implements.

**Install:**

```bash
code --install-extension Continue.continue
```

**Configure.** Continue reads its config from `~/.continue/config.yaml` (workspace-level `.continuerc.json` is legacy and unreliable in current versions — the global config is the source of truth). Replace the file contents with:

```yaml
name: Main Config
version: 1.0.0
schema: v1
models:
  - name: Qwen3.6-27B (local llama.cpp)
    provider: openai
    model: qwen3
    apiBase: http://localhost:8080/v1
    apiKey: dummy
    defaultCompletionOptions:
      contextLength: 32768
    roles:
      - chat
      - edit
      - apply
```

- `apiKey` must be non-empty even though `llama-server` ignores it.
- `model` can be any string — `llama-server` serves whatever it loaded.
- Continue auto-reloads on save; pick the model from the assistant selector at the top of the chat panel.

**Use:** reload VS Code (`Ctrl+Shift+P` → "Reload Window"), then:
- `Ctrl+L` — chat panel
- `Ctrl+I` — inline edit on selected code
- `Ctrl+Shift+L` — add selection to chat context
