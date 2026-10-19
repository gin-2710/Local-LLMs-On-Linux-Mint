# Hardware-Accelerated Local LLMs on Linux Mint (AMD Radeon GPU)

A complete guide to configuring hardware-accelerated local Large Language Models (LLMs) on **Linux Mint** using **Ollama**, **AMD ROCm / HIP**, and discrete Radeon graphics.

This walkthrough documents the exact setup and troubleshooting steps required when modern AMD cards encounter the `Agent creation failed: The GPU node has an unrecognized id` error during ROCm topology enumeration.

---

## System Specifications & Environment

The following hardware and software configuration was used for this setup:

| Component | Specification | Notes |
| :--- | :--- | :--- |
| **OS** | Linux Mint (Cinnamon) | Kernel with native `amdgpu` driver |
| **CPU** | AMD Ryzen 5 9600X | Integrated RDNA compute node (`gfx1036`) |
| **GPU** | AMD Radeon RX 9060 XT (16 GB) | Dedicated PCIe display adapter |
| **RAM** | 16 GB DDR5 | Dual-channel system RAM |
| **Storage** | NVMe M.2 SSD | Default models stored in `/usr/share/ollama/.ollama` |
| **Runtime** | Ollama + ROCm/HIP | Systemd service running on `localhost:11434` |

---

## 1. Install Ollama

Install the official Ollama Linux binary:

```bash

curl -fsSL https://ollama.com/install.sh | sh

```

Verify that the systemd service initialized:

```bash

sudo systemctl status ollama

```
---

## 2. Configure Hardware Access Permissions

By default, access to direct GPU rendering and compute nodes (`/dev/kfd`, `/dev/dri/renderD*`) requires membership in the `render` and `video` groups.

Add your current user to both groups:

```bash

sudo usermod -a -G render,video $USER

```

> **Important:** Log out of your desktop session and log back in (or reboot) for group permissions to take effect. Verify by running `groups` and ensuring both `render` and `video` appear in the output.

---

## 3. Install ROCm Userspace Diagnostics & Runtimes

Install the base ROCm management interface and HIP runtime packages:

```bash

sudo apt update

sudo apt install -y rocm-smi rocminfo libamdhip64-5 || sudo apt install -y rocm-smi rocminfo libamdhip64-6

```

### Verification

Run `rocm-smi` to inspect detected devices:

```bash

rocm-smi

```

You should see your discrete GPU listed as **GPU 0** (showing active clock speeds, power limits, and temperature) and the processor's integrated graphics as **GPU 1**.

---

## 4. Fix "Unrecognized GPU Node ID" via Systemd Override

On newer Radeon cards, standard ROCm userspace packages may fail to match the card's PCI device ID against hardcoded compiler targets, resulting in:

```text

Warning: Agent creation failed.

The GPU node has an unrecognized id.

```

To resolve this, configure an explicit RDNA compute target (`gfx1100`) and isolate GPU compute traffic to device `0`.

1. Open the systemd override editor:

   ```bash

   sudo systemctl edit ollama.service

   ```

2. Add the following configuration block:

   ```ini

   [Service]

   Environment="HSA_OVERRIDE_GFX_VERSION=11.0.0"

   Environment="HIP_VISIBLE_DEVICES=0"

   Environment="ROCR_VISIBLE_DEVICES=0"

   ```

3. Save and exit (`Ctrl + O`, `Enter`, then `Ctrl + X` in nano).

4. Reload systemd and restart the daemon:

   ```bash

   sudo systemctl daemon-reload

   sudo systemctl restart ollama

   ```

---

## 5. Verify Discovery & 100% GPU Offload

### Step A: Check Systemd Journal Logs

Inspect Ollama's hardware detection log:

```bash

journalctl -u ollama --no-pager -n 40 | grep -iE "gpu|rocm|vram"

```

**Expected Result:**

```text

level=INFO source=types.go:32 msg="inference compute" id=0 filter_id=0 library=ROCm compute=gfx1200 name=ROCm0 description="AMD Radeon Graphics" type=discrete total="15.9 GiB" available="15.8 GiB"

```

### Step B: Run a Test Inference

Pull and launch a test model:

```bash

ollama run qwen2.5:7b "Explain how a compiler works in three sentences."

```

### Step C: Confirm Full VRAM Allocation

While the model is running (or within its 5-minute keepalive window), open a second terminal:

```bash

ollama ps

```

**Expected Result:**

```text

NAME         ID            SIZE     PROCESSOR    CONTEXT    RUNNER      UNTIL

qwen2.5:7b   845dbda0ea48  4.8 GB   100% GPU     4096       llamacpp    4 minutes from now

```



---

## Recommended Models for 16 GB VRAM

With ~15.8 GiB of dedicated VRAM, these models run entirely on the GPU without spilling into system RAM:

| Task | Model | Footprint | Command |
| :--- | :--- | :--- | :--- |
| **General Assistant** | Qwen 2.5 7B | ~4.8 GB | `ollama run qwen2.5:7b` |
| **Code Refactoring & Syntax** | Qwen 2.5 Coder 7B | ~4.8 GB | `ollama run qwen2.5-coder:7b` |
| **Inline Tab-Autocomplete** | Qwen 2.5 Coder 1.5B | ~1.2 GB | `ollama run qwen2.5-coder:1.5b` |
| **Deep Reasoning & Math** | DeepSeek-R1 (8B distilled) | ~4.9 GB | `ollama run deepseek-r1:8b` |
| **Dense Intelligence** | Qwen 2.5 14B | ~9.0 GB | `ollama run qwen2.5:14b` |

---

## IDE Integration: VS Code (Continue Extension)

Ollama automatically exposes an OpenAI-compatible API at `http://localhost:11434`.

1. Install the **Continue** extension from the VS Code Extensions Marketplace (`Ctrl + Shift + X`).

2. Open `~/.continue/config.yaml` and configure local endpoints:

```yaml

models:

  - name: Qwen2.5 Coder 7B

    provider: ollama

    model: qwen2.5-coder:7b

    apiBase: http://localhost:11434



tabAutocompleteModel:

  title: Qwen2.5 Coder 1.5B

  provider: ollama

  model: qwen2.5-coder:1.5b

  apiBase: http://localhost:11434

```
---

## Useful Operational Commands

```bash

# Monitor live GPU thermals, power, and VRAM utilization

watch -n 1 rocm-smi


# Evict the active model from VRAM immediately

ollama stop qwen2.5:7b


# Check disk space used by pulled models

ollama list


# Delete a model to free disk storage

ollama rm <model-name>

```
