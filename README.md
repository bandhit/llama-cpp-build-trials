# llama.cpp Build and Run

Instructions for building [llama.cpp](https://github.com/ggml-org/llama.cpp) from source and running a model via `llama-server`.

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

Once running, the OpenAI-compatible API is available at `http://localhost:8080`.
