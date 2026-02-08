# 🧭 Dippy Studio Bittensor Miner [SN-11]

A specialized Bittensor subnet miner (subnet 231 on testnet) for high-performance FLUX.1-dev inference with TensorRT acceleration and LoRA refitting, plus deterministic FLUX.1-Kontext-dev image editing.

---

## 📚 Table of Contents

- [About](#-about)
- [Features](#-features)
- [Tech Stack](#-tech-stack)
- [Installation](#️-installation)
- [Usage](#-usage)
- [Configuration](#-configuration)
- [Screenshots](#-screenshots)
- [API Documentation](#-api-documentation)
- [Contact](#-contact)
- [Acknowledgements](#-acknowledgements)

---

## 🧩 About

This project provides a Bittensor miner that serves FLUX.1-dev image generation and FLUX.1-Kontext-dev image editing with TensorRT acceleration. It solves the need for a high-performance, LoRA-capable inference endpoint on Bittensor subnet 231 (testnet), with optional deterministic editing via Kontext.

**Key goals:**

- TensorRT-accelerated FLUX.1-dev inference with LoRA refitting
- Optional FLUX.1-Kontext-dev deterministic image editing
- Reverse proxy for Bittensor auth and routing
- Docker-based deployment with Make targets for setup and testing

---

## ✨ Features

- **Inference mode** – TensorRT-accelerated image generation with optional LoRA and async callbacks
- **Kontext editing mode** – Deterministic image editing with FLUX.1-Kontext-dev and callback support
- **Reverse proxy** – Bittensor authentication and request routing to internal services
- **Make-based workflow** – `setup-inference`, `setup-kontext`, `trt-build`, `test-kontext-determinism`, and more
- **Docker deployment** – Isolated builds for inference, Kontext, and TRT engine

---

## 🧠 Tech Stack

| Category   | Technologies |
| ---------- | ------------ |
| **Languages** | Python |
| **Frameworks** | FastAPI, Uvicorn, Bittensor |
| **ML/Inference** | FLUX.1-dev, FLUX.1-Kontext-dev, TensorRT, Diffusers, LoRA (LyCORIS/PEFT) |
| **Database** | N/A (file-based output) |
| **Tools** | Docker, Docker Compose, Make, uv, Hugging Face Hub |

---

## ⚙️ Installation

### Prerequisites

- **GPU:** H100 PCIe with driver/CUDA as below (other GPUs with compute capability 7.0+ may work; 24GB+ VRAM recommended).
- **Driver/CUDA (reference):**
  ```
  NVIDIA-SMI 570.172.08   Driver Version: 570.172.08   CUDA Version: 12.8
  ```
- **Python:** [uv](https://docs.astral.sh/uv/getting-started/installation/) is recommended.

Before running the miner:

1. **HuggingFace** – Create an account and token at [HuggingFace Settings](https://huggingface.co/settings/tokens) (write permissions for model uploads).
2. **FLUX.1-dev license** – Accept the [FLUX.1-dev model license](https://huggingface.co/black-forest-labs/FLUX.1-dev).

```bash
# Clone the repository
git clone https://github.com/dippy-ai/dippy-studio-bittensor-miner.git
cd dippy-studio-bittensor-miner

# Copy environment template and add your credentials
cp .env.example .env
# Edit .env and set HF_TOKEN=your_huggingface_token_here

# Install dependencies (if running locally without Docker)
# pip install -r requirements.txt   # or: uv pip install -r requirements.txt
```

---

## 🚀 Usage

### Choose deployment mode

- **Inference only** – Pre-trained FLUX.1-dev with TensorRT
- **Kontext editing** – FLUX.1-Kontext-dev deterministic editing

```bash
# Inference-only (builds TRT engine if needed)
make setup-inference

# With Kontext editing
make setup-kontext
# Or: echo "ENABLE_KONTEXT_EDIT=true" >> .env && make restart

# View logs
make logs
```

Then open or call:

👉 **Miner API:** [http://localhost:8091](http://localhost:8091)

### Reverse proxy (Bittensor entrypoint)

1. Register on testnet: `btcli s register --netuid 231 --subtensor.network test`
2. Transfer 0.01 testnet TAO to the mining permit address (see repo/docs).
3. Configure `reverse_proxy/.env` (e.g. `MINER_HOTKEY`, `INFERENCE_SERVER_URL`).
4. Run:
   ```bash
   cd reverse_proxy
   uv pip install -e .[dev]
   python server.py
   ```

### Make commands (summary)

| Command | Description |
|--------|-------------|
| `make setup-inference` | Deploy inference-only server |
| `make setup-kontext`   | Deploy with Kontext editing |
| `make build`           | Build Docker images |
| `make trt-build`       | Build TensorRT engine (~20–30 min) |
| `make trt-rebuild`     | Force TRT rebuild |
| `make up` / `make down`| Start / stop services |
| `make logs` / `make restart` | Logs / restart |
| `make test-kontext-determinism` | Kontext E2E tests |
| `make test-kontext-unit`       | Kontext unit tests |
| `make clean-cache`     | Remove cached TRT engines |
| `make help`            | List all commands |

---

## 🧾 Configuration

Create a `.env` file in the project root:

```bash
# Required
HF_TOKEN=your_huggingface_token_here

# Mode
ENABLE_INFERENCE=true
ENABLE_KONTEXT_EDIT=false   # set true for Kontext editing
MODEL_PATH=black-forest-labs/FLUX.1-dev
OUTPUT_DIR=/app/output
MINER_SERVER_PORT=8091
MINER_SERVER_HOST=0.0.0.0
SERVICE_URL=http://localhost:8091
```

For the **reverse proxy**, create `reverse_proxy/.env`:

```bash
MINER_HOTKEY=your_miner_hotkey_here
INFERENCE_SERVER_URL=http://localhost:8091
```


---

## 📜 API Documentation

Base URL: `http://localhost:8091`

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/inference` | Generate image (optional LoRA, callback) |
| `POST` | `/edit` | Edit image with FLUX.1-Kontext-dev (callback supported) |
| `GET`  | `/inference/status/{job_id}` | Inference job status |
| `GET`  | `/edit/status/{job_id}` | Edit job status |
| `GET`  | `/inference/result/{job_id}` | Download generated image |
| `GET`  | `/edit/result/{job_id}` | Download edited image |
| `GET`  | `/health` | Health check |

**Example – generate image:**

```bash
curl -X POST http://localhost:8091/inference \
  -H "Content-Type: application/json" \
  -d '{
    "prompt": "A beautiful sunset over mountains",
    "width": 1024,
    "height": 1024,
    "num_inference_steps": 28,
    "guidance_scale": 7.5,
    "seed": 42
  }'
```

**Example – edit image:**

```bash
curl -X POST http://localhost:8091/edit \
  -H "Content-Type: application/json" \
  -d '{
    "prompt": "Add a red hat to the person",
    "image_b64": "base64_encoded_image_data",
    "seed": 42,
    "guidance_scale": 2.5,
    "num_inference_steps": 28
  }'
```

Full Kontext API and async callbacks: [docs/kontext-editing.md](docs/kontext-editing.md).  
Async inference E2E: [docs/async_inference_e2e.md](docs/async_inference_e2e.md).

---

## 📬 Contact 
- Author: PJS
- Email: cpalvarez95999@gmail.com
- GitHub: @algodev999

---

## 🌟 Acknowledgements

- [Bittensor](https://docs.bittensor.com) – Subnet and miner framework
- [FLUX.1-dev](https://huggingface.co/black-forest-labs/FLUX.1-dev) – Black Forest Labs
- [FLUX.1-Kontext-dev](https://huggingface.co/black-forest-labs/FLUX.1-Kontext-dev) – Kontext editing model
- [Diffusers](https://github.com/huggingface/diffusers) – Hugging Face
- [TensorRT](https://developer.nvidia.com/tensorrt) – NVIDIA
- Contributors and testnet validators on subnet 231
