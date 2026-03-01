# RKLLama: Getting Started Guide

A practical guide covering installation, first run, and adding models — covering everything the README assumes you already know.

## Requirements

- Rockchip RK3588(S) or RK3576 device (Orange Pi 5, Rock 5, etc.)
- Ubuntu 24.04 arm64 or Armbian (Debian Bookworm)
- Python 3.9 – 3.12
- Internet connection (for first-time tokenizer download)

---

## 1. Clone and Install

Clone the repository **anywhere you like** — it does not need to be in your home folder:

```bash
git clone https://github.com/notpunchnox/rkllama
cd rkllama
```

Install the package (a virtual environment is recommended but not required):

```bash
# Optional but recommended: create a venv first
python3 -m venv .venv
source .venv/bin/activate

# Install
pip install .
```

After installation, two commands become available system-wide:

| Command | Purpose |
|---|---|
| `rkllama_server` | Starts the inference server |
| `rkllama_client` | CLI client to talk to the server |

---

## 2. Where Models Are Stored

After installation, RKLLama stores models in:

```
~/.local/share/rkllama/models/
```

This directory is created automatically the first time the server starts. **It does not matter where you cloned the repo** — models always live in your home directory.

You can override this with the `--models` flag or an environment variable:

```bash
# Use a custom models directory for this session
rkllama_server --models /mnt/ssd/my-models

# Or set it permanently via environment variable
export RKLLAMA_PATHS_MODELS=/mnt/ssd/my-models
```

---

## 3. Start the Server

The server must run as root to set NPU frequencies (for best performance):

```bash
sudo rkllama_server
```

Without root it still works, but NPU clock speeds won't be optimised.

Useful flags:

```bash
# Specify models directory
sudo rkllama_server --models /path/to/models

# Change port (default: 8080)
sudo rkllama_server --port 8081

# Enable verbose debug logging
sudo rkllama_server --debug

# Specify processor explicitly if auto-detection fails
sudo rkllama_server --processor rk3588   # or rk3576
```

You should see output like:

```
Start the API at http://localhost:8080
```

Leave this terminal open. Open a second terminal for the client.

---

## 4. Add a Model

### Option A — Pull from Hugging Face (easiest)

With the server running:

```bash
rkllama_client pull username/repo_id/model_file.rkllm/my-model-name
```

Example:

```bash
rkllama_client pull c01zaut/Qwen2.5-3B-Instruct-RK3588-1.1.4/Qwen2.5-3B-Instruct-rk3588-w8a8-opt-0-hybrid-ratio-0.5.rkllm/qwen2.5:3b
```

This downloads the model and creates the Modelfile automatically.

### Option B — Manual installation

1. Download a `.rkllm` file from [Hugging Face](https://huggingface.co).

2. Create a folder for the model inside `~/.local/share/rkllama/models/`:

```bash
mkdir -p ~/.local/share/rkllama/models/tinyllama:1.1b
```

3. Copy the `.rkllm` file into that folder:

```bash
cp TinyLlama-1.1B-Chat-v1.0.rkllm ~/.local/share/rkllama/models/tinyllama:1.1b/
```

4. Create a `Modelfile` in the same folder:

```bash
nano ~/.local/share/rkllama/models/tinyllama:1.1b/Modelfile
```

Minimum required content:

```env
FROM="TinyLlama-1.1B-Chat-v1.0.rkllm"
HUGGINGFACE_PATH="TinyLlama/TinyLlama-1.1B-Chat-v1.0"
```

Full Modelfile options:

```env
FROM="your-model-file.rkllm"
HUGGINGFACE_PATH="huggingface/repo-id"
SYSTEM="You are a helpful assistant."
TEMPERATURE=0.7
NUM_CTX=4096
MAX_NEW_TOKENS=1024
TOP_K=7
TOP_P=0.9
REPEAT_PENALTY=1.1
```

> **Why `HUGGINGFACE_PATH`?** RKLLama downloads the tokenizer and chat template from this HuggingFace repo on the first load. An internet connection is required once. The tokenizer is then cached inside the model folder. You can point to any compatible tokenizer repo — it does not have to be the exact model source.

The resulting folder structure should look like this:

```
~/.local/share/rkllama/models/
    └── tinyllama:1.1b/
        ├── Modelfile
        └── TinyLlama-1.1B-Chat-v1.0.rkllm
```

---

## 5. Verify and Run

List available models:

```bash
rkllama_client list
```

Run a model interactively:

```bash
rkllama_client run tinyllama:1.1b
```

---

## 6. Multimodal Models (Vision)

For models with a vision encoder (e.g. Qwen2-VL, MiniCPMV4):

1. Place both the `.rkllm` and `.rknn` encoder files in the model folder.
2. Add the vision properties to the `Modelfile`:

```env
FROM="Qwen2-VL-2B-Instruct.rkllm"
HUGGINGFACE_PATH="Qwen/Qwen2-VL-2B-Instruct"
IMAGE_WIDTH=392
IMAGE_HEIGHT=392
N_IMAGE_TOKENS=196
IMG_START=<|vision_start|>
IMG_END=<|vision_end|>
IMG_CONTENT=<|image_pad|>
```

Folder structure:

```
~/.local/share/rkllama/models/
    └── qwen2-vision:2b/
        ├── Modelfile
        ├── Qwen2-VL-2B-Instruct.rkllm
        └── Qwen2-VL-2B-Instruct.rknn
```

---

## 7. Uninstall

```bash
pip uninstall rkllama
```

Model files in `~/.local/share/rkllama/models/` are **not** removed — delete that folder manually if you want to clean up completely.

---

## Troubleshooting

**`rkllama_client list` shows no models**
- Check the server is running: `rkllama_client list` needs the server up first.
- Check the models directory the server is using. Start the server with `--debug` to see the resolved paths in logs.
- Make sure each model has its own subfolder containing both the `.rkllm` file and a `Modelfile`.

**Server fails to set NPU frequency**
- Run with `sudo`. Without root the frequency script is skipped but inference still works.

**"tokenizer not found" on first load**
- The server needs internet access to download the tokenizer the first time. After that it's cached locally.

**Want to use a different models directory without moving files**
- Start the server with `--models /your/path` or set `export RKLLAMA_PATHS_MODELS=/your/path` before running.
