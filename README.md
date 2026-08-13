# llama.cpp Build and Run

Instructions for building [llama.cpp](https://github.com/ggml-org/llama.cpp) from source, running a model via `llama-server`, and wiring it into VS Code.

## Prerequisites

Install the required system packages:

```bash
sudo apt update
sudo apt install -y git build-essential cmake
```

## Build

Clone the repository and build with CMake:

```bash
git clone https://github.com/ggml-org/llama.cpp.git main
cd main
mkdir build
cd build
cmake ..
cmake --build . --config Release
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
| `-hf <repo>:<quant>` | Fetch a model from Hugging Face by repo and quant tag. |
| `--jinja` | Use the model's Jinja chat template for prompt formatting. |
| `--port 8080` | Serve the HTTP API on port 8080. |
| `-ngl 99` | Offload up to 99 layers to the GPU (use a large number to offload all). |

Once running, the OpenAI-compatible API is available at `http://localhost:8080/v1`.

> **Security note:** `llama-server` defaults to CORS-open and no API key. Fine on localhost — do not expose port 8080 to your network without setting `--api-key`.

## Use from VS Code

Opening this repo in VS Code will prompt you to install the recommended extensions listed in [`.vscode/extensions.json`](.vscode/extensions.json) — Continue (chat) and llama.vscode (autocomplete). You can also install them manually with the commands below.

### Chat — via Continue

[Continue](https://marketplace.visualstudio.com/items?itemName=Continue.continue) is a general chat + inline-edit extension that speaks the OpenAI Chat Completions protocol, which `llama-server` implements.

**Install:**

```bash
code --install-extension Continue.continue
```

**Configure per-project.** This repo ships a [`.continuerc.json`](.continuerc.json) that points Continue at `http://localhost:8080/v1` and merges on top of your global `~/.continue/config.yaml`:

```json
{
  "models": [
    {
      "title": "Qwen3.6-27B (local llama.cpp)",
      "provider": "openai",
      "model": "qwen3",
      "apiBase": "http://localhost:8080/v1",
      "apiKey": "dummy",
      "contextLength": 32768
    }
  ],
  "mergeBehavior": "merge"
}
```

- `apiKey` must be non-empty even though `llama-server` ignores it.
- `model` can be any string — `llama-server` serves whatever it loaded.
- `mergeBehavior: "merge"` layers this on top of the global config; use `"overwrite"` to replace top-level keys entirely.

**Use:** reload VS Code (`Ctrl+Shift+P` → "Reload Window"), then:
- `Ctrl+L` — chat panel
- `Ctrl+I` — inline edit on selected code
- `Ctrl+Shift+L` — add selection to chat context

### Code completion — via llama.vscode

[llama.vscode](https://marketplace.visualstudio.com/items?itemName=ggml-org.llama-vscode) is the official extension for fill-in-middle (FIM) autocomplete. It requires a **FIM-trained coder model** — a general chat model like Qwen3.6-27B will not work well.

**Install:**

```bash
code --install-extension ggml-org.llama-vscode
```

**Run a coder model on port 8012** (the extension's default), in parallel with your chat server:

```bash
cd ~/workspaces/llama_cpp/main/build/bin

# <8GB VRAM
./llama-server -hf ggml-org/Qwen2.5-Coder-1.5B-Q8_0-GGUF \
  --port 8012 -ub 1024 -b 1024 --ctx-size 0 --cache-reuse 256 -ngl 99

# 8–16GB VRAM: Qwen2.5-Coder-3B-Q8_0-GGUF
# 16GB+ VRAM:  Qwen2.5-Coder-7B-Q8_0-GGUF
```

Type in any code file — completions should appear inline. Click the `llama-vscode` status bar item (or press `Ctrl+Shift+M`) for the menu.
