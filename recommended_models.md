# Recommended Ollama Models for Odysseus

Based on your system specifications (**NVIDIA GeForce RTX 5090** with **24 GB VRAM**, **96 GB system RAM**) and the agentic, multimodal, and coding capabilities of Odysseus, here is the curated list of models to download and use:

---

### 1. Primary Chat & General Agent Work (Your Installed Model)
* **`qwen3:32b`** (Already Installed)
  * **Memory Footprint**: ~20 GB (Fits entirely in VRAM).
  * **Why**: Keep this as your primary daily driver for text chat, planning, and Deep Research. It is extremely fast on the RTX 5090.

---

### 2. Multimodal & Browser Agents (Reading Uploads & Screenshots)
Odysseus has a browser tool (taking screenshots) and file upload options, which require a Multimodal (Vision) model. It also requires the model to support function/tool calling reliably.
* **🏆 `gemma4:26b`** (Highly Recommended)
  * **Memory Footprint**: ~18 GB (Fits entirely in VRAM).
  * **Why**: The standout multimodal model of 2026. It features native vision processing (perfect for browser screenshots) alongside outstanding structured JSON output and function calling.
  * **Command**: `ollama pull gemma4:26b`
* **`phi4`** (3.8B - For rapid, text-only agent loops)
  * **Memory Footprint**: ~3 GB (Lightning fast).
  * **Why**: Extremely lightweight and fast. If you are running repetitive agent tasks that do not require vision, this model will execute steps in milliseconds.
  * **Command**: `ollama pull phi4`

---

### 3. Agentic Software Engineering & Coding
For the Odysseus Document Editor and running scripts in the terminal:
* **🏆 `qwen3-coder:30b`** (Highly Recommended)
  * **Memory Footprint**: ~18 GB (Fits entirely in VRAM).
  * **Why**: The state-of-the-art coder model. It uses a Mixture of Experts (MoE) architecture with 3.3B activated parameters. It supports a **256K context window**, allowing the agent to read entire code directories or log files at once in Odysseus.
  * **Command**: `ollama pull qwen3-coder:30b`

---

### 4. Reasoning & Logic (Deep Research Planning)
For complex research planning where the AI must "think" before acting:
* **`deepseek-r1:32b`**
  * **Memory Footprint**: ~20 GB (Fits entirely in VRAM).
  * **Why**: Distilled from DeepSeek-R1. It outputs a hidden reasoning chain before writing its final response, yielding far better logical accuracy.
  * **Command**: `ollama pull deepseek-r1:32b`

---

### 5. High-Capacity Synthesis (CPU/GPU Hybrid Offloading)
* **`llama3.3:70b`**
  * **Memory Footprint**: ~42 GB (GPU handles ~20GB, your **96GB RAM** handles the rest).
  * **Why**: The benchmark for high-parameter general-purpose instruction following. It will run slower since it exceeds your 24GB VRAM, but your system RAM is more than large enough to run it smoothly.
  * **Command**: `ollama pull llama3.3:70b`

---

### 🎨 Local Image Generation Setup (RTX 5090)
Ollama serves text/vision models, but does not support local text-to-image models out of the box. Since you have 24GB VRAM, you can run top-tier image models locally:

#### Recommended Local Image Models
1. **FLUX.2 [dev]**: State-of-the-art detail, prompt adherence, and text-in-image rendering. (Fits in 24GB VRAM).
2. **FLUX.2 [schnell]**: Fast 4-step generation (1–2 seconds per image on RTX 5090).
3. **Stable Diffusion 3.5 (Large/Medium)**: Extremely versatile for artistic styling.

#### How to Connect Local Image Gen to Odysseus
1. Run a local image engine like **ComfyUI** or **AUTOMATIC1111 (Stable Diffusion WebUI)**.
2. Run an OpenAI-compatible API wrapper, such as **[openedai-images](https://github.com/matatonic/openedai-images)** or **[LocalAI](https://github.com/mudler/LocalAI)**. This exposes an endpoint at `http://localhost:5005/v1`.
3. In Odysseus **Settings** -> **Add Models** -> **API**, add:
   * **Base URL**: `http://localhost:5005/v1`
   * **API Key**: `skip`
4. Under **Settings** -> **System**, set your default image generation model to your local model.

---

### 🎙️ Local Voice Input (Speech-to-Text)
You can speak to Odysseus using your microphone rather than typing.
* **The Model**: **Whisper (e.g. `large-v3-turbo`)**
* **How to enable**:
  1. Install the voice dependencies in the Odysseus virtual environment:
     ```bash
     .\venv\Scripts\pip install faster-whisper
     ```
  2. Restart Odysseus and go to **Settings** -> **System** -> **Speech to Text** and select **Local** as your provider.

---

### 🧠 Local Semantic Memory & RAG Embeddings
Odysseus uses vectors to store long-term memory and index uploaded documents.

* **Recommended Embedding Model**: **`nomic-embed-text`**
  * **Why**: High-performance local embeddings with an 8k token window.
  * **Command**: `ollama pull nomic-embed-text`
* **Local Vector Database (ChromaDB)**:
  * Odysseus uses ChromaDB to store vector indices. To run a local instance:
    1. Install it in the venv:
       ```bash
       .\venv\Scripts\pip install chromadb
       ```
    2. Start the ChromaDB server:
       ```bash
       .\venv\Scripts\chroma run --host localhost --port 8100
       ```
    3. Restart Odysseus, and it will automatically connect.
