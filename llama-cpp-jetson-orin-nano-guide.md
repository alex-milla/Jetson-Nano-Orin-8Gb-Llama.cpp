# Running Local LLMs on Jetson Orin Nano 8GB with llama.cpp

A complete step-by-step guide to compile llama.cpp with CUDA support, download GGUF models, and run a local AI chat server on the NVIDIA Jetson Orin Nano 8GB.

## Tested Environment

| Component | Version |
|-----------|---------|
| **Device** | NVIDIA Jetson Orin Nano 8GB |
| **L4T** | R36, REVISION 4.4 |
| **JetPack** | 6.x |
| **CUDA** | 12.6 (V12.6.68) |
| **Architecture** | aarch64 |
| **Kernel variant** | oot |
| **Python** | 3.10 |
| **llama.cpp build** | b8932 |

> **Note:** These instructions are verified on the exact versions above. Other JetPack or CUDA versions may require adjustments.

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [Verify Your System](#2-verify-your-system)
3. [Install Build Dependencies](#3-install-build-dependencies)
4. [Clone and Compile llama.cpp](#4-clone-and-compile-llamacpp)
5. [Install Hugging Face CLI](#5-install-hugging-face-cli)
6. [Download Models](#6-download-models)
7. [Run the Chat Server](#7-run-the-chat-server)
8. [Set Up Auto-Start Service](#8-set-up-auto-start-service)
9. [Switching Between Models](#9-switching-between-models)
10. [Recommended Models for 8GB](#10-recommended-models-for-8gb)
11. [Troubleshooting](#11-troubleshooting)

---

## 1. Prerequisites

- NVIDIA Jetson Orin Nano 8GB with JetPack 6.x flashed and running
- Internet connection
- SSH access or a terminal session on the device
- At least 15 GB of free disk space (for llama.cpp + models)

---

## 2. Verify Your System

Before starting, confirm your device and CUDA setup:

```bash
# Check L4T / JetPack version
cat /etc/nv_tegra_release
```

Expected output should include `R36 (release), REVISION: 4.4` or similar.

```bash
# Check CUDA version
/usr/local/cuda/bin/nvcc --version
```

Expected output should show `Cuda compilation tools, release 12.6` or similar.

```bash
# Check available GPU
nvidia-smi
```

> **If `nvcc` is not found**, make sure CUDA is in your PATH. Add this to your `~/.bashrc`:
> ```bash
> export PATH=/usr/local/cuda/bin:$PATH
> export LD_LIBRARY_PATH=/usr/local/cuda/lib64:$LD_LIBRARY_PATH
> ```
> Then run `source ~/.bashrc`.

---

## 3. Install Build Dependencies

llama.cpp needs `cmake` and `gcc` to compile. Install them:

```bash
sudo apt update
sudo apt install -y cmake gcc g++ git make
```

Verify they are installed:

```bash
cmake --version
gcc --version
```

---

## 4. Clone and Compile llama.cpp

Clone the repository and build with CUDA support:

```bash
cd ~
git clone https://github.com/ggml-org/llama.cpp.git
cd llama.cpp
cmake -B build -DGGML_CUDA=ON
cmake --build build --config Release -j$(nproc)
```

> **What the flags mean:**
> - `-DGGML_CUDA=ON` enables NVIDIA GPU acceleration
> - `-j$(nproc)` uses all available CPU cores for faster compilation

Compilation takes approximately 5-10 minutes on the Orin Nano. When finished, you should see lines like:

```
[100%] Built target llama-server
[100%] Built target llama-cli
```

Verify the binaries exist:

```bash
ls -la build/bin/llama-server
ls -la build/bin/llama-cli
```

---

## 5. Install Hugging Face CLI

The `hf` CLI tool is used to download GGUF model files from Hugging Face:

```bash
pip install huggingface_hub
```

> **Note:** If you get a permissions error, the install will default to `~/.local/bin/`. Make sure it's in your PATH:
> ```bash
> export PATH=$HOME/.local/bin:$PATH
> ```
> Add this line to your `~/.bashrc` to make it permanent.

Verify it works:

```bash
hf --help
```

> **Optional but recommended:** For faster downloads, install the transfer accelerator:
> ```bash
> pip install hf_transfer
> export HF_HUB_ENABLE_HF_TRANSFER=1
> ```

---

## 6. Download Models

Create a models directory and download your first model:

```bash
mkdir -p ~/llama.cpp/models
```

### Qwen3.5-4B (General purpose / Reasoning — 2.74 GB)

```bash
hf download unsloth/Qwen3.5-4B-GGUF Qwen3.5-4B-Q4_K_M.gguf --local-dir ~/llama.cpp/models/
```

### Gemma 4 E2B (Content creation / Translation / Multimodal — 3.11 GB)

```bash
hf download unsloth/gemma-4-E2B-it-GGUF gemma-4-E2B-it-Q4_K_M.gguf --local-dir ~/llama.cpp/models/
```

### Phi-4 Mini (Fast log analysis / Commands — ~2.8 GB)

```bash
hf download unsloth/Phi-4-mini-instruct-GGUF Phi-4-mini-instruct-Q4_K_M.gguf --local-dir ~/llama.cpp/models/
```

> **Tip:** Downloads can be resumed if interrupted. Just run the same command again.

Verify all models are downloaded:

```bash
ls -lh ~/llama.cpp/models/*.gguf
```

---

## 7. Run the Chat Server

Launch llama-server with GPU acceleration:

```bash
cd ~/llama.cpp
./build/bin/llama-server \
  -m models/Qwen3.5-4B-Q4_K_M.gguf \
  --gpu-layers 99 \
  --host 0.0.0.0 \
  --port 8080
```

> **What the flags mean:**
> - `-m` path to the GGUF model file
> - `--gpu-layers 99` offloads all layers to GPU for maximum speed
> - `--host 0.0.0.0` listens on all network interfaces (accessible from other devices)
> - `--port 8080` the port number for the web server

Wait for the warm-up to finish. You should see:

```
main: server is listening on 0.0.0.0:8080
```

### Access the Chat GUI

Open a browser on any device in your local network and go to:

```
http://<your-orin-ip>:8080
```

To find your Orin's IP address:

```bash
hostname -I
```

You should see a chat interface. Type a message and confirm the model responds.

### Expected Performance

| Model | Speed | Tokens |
|-------|-------|--------|
| Qwen3.5-4B Q4_K_M | ~9 t/s | Context auto-adjusted to ~52K |

---

## 8. Set Up Auto-Start Service

Create a systemd service so llama-server starts automatically on boot:

```bash
sudo nano /etc/systemd/system/llama-server.service
```

Paste the following content:

```ini
[Unit]
Description=llama.cpp server
After=network.target

[Service]
Type=simple
User=alex
ExecStart=/home/alex/llama.cpp/build/bin/llama-server -m /home/alex/llama.cpp/models/Qwen3.5-4B-Q4_K_M.gguf --gpu-layers 99 --host 0.0.0.0 --port 8080
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

> **Important:** Replace `alex` with your actual username if different.

Save the file (`Ctrl+O`, `Enter`, `Ctrl+X`), then enable and start the service:

```bash
sudo systemctl daemon-reload
sudo systemctl enable llama-server
sudo systemctl start llama-server
```

Verify it's running:

```bash
sudo systemctl status llama-server
```

You should see `Active: active (running)`. The server will now start automatically every time the Orin boots.

### Useful Service Commands

```bash
# Stop the server
sudo systemctl stop llama-server

# Restart the server
sudo systemctl restart llama-server

# View logs
sudo journalctl -u llama-server -f

# Disable auto-start
sudo systemctl disable llama-server
```

---

## 9. Switching Between Models

The Orin Nano can only run one model at a time due to the 8 GB memory constraint. To switch models:

**Option A: Restart the service with a different model**

Edit the service file:

```bash
sudo nano /etc/systemd/system/llama-server.service
```

Change the `-m` path to the desired model, then:

```bash
sudo systemctl daemon-reload
sudo systemctl restart llama-server
```

**Option B: Stop the service and run manually**

```bash
sudo systemctl stop llama-server

cd ~/llama.cpp
./build/bin/llama-server \
  -m models/gemma-4-E2B-it-Q4_K_M.gguf \
  --gpu-layers 99 \
  --host 0.0.0.0 \
  --port 8080
```

---

## 10. Recommended Models for 8GB

The Jetson Orin Nano 8GB can comfortably run models up to ~4B parameters in Q4 quantization. Here are the best options as of April 2026:

| Model | Size | Best For | Quantization |
|-------|------|----------|-------------|
| [Qwen3.5-4B](https://huggingface.co/unsloth/Qwen3.5-4B-GGUF) | 2.74 GB | General chat, reasoning, coding | Q4_K_M |
| [Gemma 4 E2B](https://huggingface.co/unsloth/gemma-4-E2B-it-GGUF) | 3.11 GB | Writing, translation, image+audio | Q4_K_M |
| [Phi-4 Mini](https://huggingface.co/unsloth/Phi-4-mini-instruct-GGUF) | ~2.8 GB | Fast analysis, logs, commands | Q4_K_M |
| [Qwen3.5-2B](https://huggingface.co/unsloth/Qwen3.5-2B-GGUF) | 1.28 GB | Ultra-light, 201 languages | Q4_K_M |

> **Models that do NOT fit comfortably:** Anything 7B+ in Q4 will exceed available memory or leave no room for context. Stick to 4B and below.

---

## 11. Troubleshooting

### `nvcc: command not found`

Add CUDA to your PATH:

```bash
echo 'export PATH=/usr/local/cuda/bin:$PATH' >> ~/.bashrc
echo 'export LD_LIBRARY_PATH=/usr/local/cuda/lib64:$LD_LIBRARY_PATH' >> ~/.bashrc
source ~/.bashrc
```

### Build fails with `CUDA not found`

Make sure JetPack is properly installed and CUDA toolkit is present:

```bash
ls /usr/local/cuda/
```

If the directory doesn't exist, reinstall JetPack following [NVIDIA's official guide](https://developer.nvidia.com/embedded/jetpack).

### `hf: command not found`

The Hugging Face CLI may have installed to `~/.local/bin/`. Add it to PATH:

```bash
echo 'export PATH=$HOME/.local/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
```

### `huggingface-cli` is deprecated

Use `hf` instead of `huggingface-cli`. The commands are the same:

```bash
# Old (deprecated)
huggingface-cli download ...

# New
hf download ...
```

### Download stuck at 0%

Check for stale lock files:

```bash
ls ~/.cache/huggingface/hub/.locks/
```

If you see `.lock` files, remove them:

```bash
rm -rf ~/.cache/huggingface/hub/.locks/*
```

Then retry the download.

### Server crashes with out-of-memory

The model is too large for your available RAM. Try a smaller quantization or a smaller model. llama-server will automatically reduce the context length to fit in memory, but if the model itself doesn't fit, it will crash.

### Server not accessible from other devices

Make sure you're using `--host 0.0.0.0` (not `127.0.0.1`). Also check that no firewall is blocking port 8080:

```bash
sudo ufw status
```

If UFW is active, allow the port:

```bash
sudo ufw allow 8080
```

---

## API Access

llama-server exposes an OpenAI-compatible API at:

```
http://<your-orin-ip>:8080/v1/chat/completions
```

Example with `curl`:

```bash
curl http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "any",
    "messages": [
      {"role": "system", "content": "You are a helpful assistant."},
      {"role": "user", "content": "Hello, what can you do?"}
    ]
  }'
```

Example with Python:

```python
from openai import OpenAI

client = OpenAI(
    base_url="http://<your-orin-ip>:8080/v1",
    api_key="not-needed"
)

response = client.chat.completions.create(
    model="any",
    messages=[
        {"role": "system", "content": "You are a helpful assistant."},
        {"role": "user", "content": "Hello!"}
    ]
)

print(response.choices[0].message.content)
```

> **Note:** The `model` field can be any string — llama-server ignores it and uses whichever model was loaded at startup.

---

## Resources

- [llama.cpp GitHub](https://github.com/ggml-org/llama.cpp)
- [Unsloth GGUF Models](https://huggingface.co/unsloth)
- [NVIDIA Jetson Developer Zone](https://developer.nvidia.com/embedded-computing)
- [Hugging Face Hub CLI](https://huggingface.co/docs/huggingface_hub/guides/cli)

---

## License

This guide is provided as-is for educational purposes. Models have their own licenses — check each model's Hugging Face page before commercial use.
