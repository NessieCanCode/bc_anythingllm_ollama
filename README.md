# Open OnDemand: AnythingLLM + Ollama

A production-ready Open OnDemand (OOD) interactive application that provisions a completely private, secure, and isolated Large Language Model (LLM) workspace on HPC compute nodes.

This package orchestrates **AnythingLLM** (for the web UI, vector database, and Retrieval-Augmented Generation/RAG) and **Ollama** (for local model serving) within a single Slurm batch job using Singularity/Apptainer. 

It dynamically scales to fit the requested hardware, allowing researchers to run fast 8B models on CPU nodes or massive 120B reasoning models on multi-GPU nodes (A100/H100).

## Key Features
* **100% Data Privacy:** No APIs, no data leaves the compute node. Perfect for researchers analyzing sensitive IP, PHI, or restricted datasets.
* **Built-in RAG:** Users can upload PDFs, codebases, and datasets. AnythingLLM embeds them into a local LanceDB vector database for chat context.
* **Shared Weights, Private Data:** Heavy LLM weights are stored in a centralized shared directory to save disk space, while user chat histories, settings, and vector databases are written securely to their `~/.anythingllm_storage` directory.

---

## Architecture
Because Open OnDemand serves interactive apps via dynamic subpaths `/rnode/<hostname>/<port>/`, apps expecting the domain root `/`, like AnythingLLM, will fail to route properly. We solve this using `resolvr`, a lightweight SPA subpath router container that bridges the traffic to the AnythingLLM frontend. Meanwhile, AnythingLLM communicates securely with the Ollama model server over isolated, internal localhost ports.

1. `resolvr` (Listens on OOD assigned port, proxies to AnythingLLM)
2. `AnythingLLM` (Web frontend, RAG pipeline, Vector DB)
3. `Ollama` (Backend model runner, dynamically allocates to GPUs)

---

## Prerequisites

### Host Node Requirements
* **Open OnDemand** (v3.0+)
* **SingularityCE** or **Apptainer** installed on compute nodes.
* Compute nodes with GPUs **MUST** have the NVIDIA Container Toolkit installed:
  ```bash
  # RHEL/Rocky/Alma Example
  sudo yum install nvidia-container-toolkit -y
  ```

### Image Setup
You will need to pull the required Docker images into Singularity `.sif` format and store them in a globally accessible location on your cluster (e.g., `/opt/unmanaged/images/llm/` or a shared NFS/GPFS mount).

```bash
# Pull Ollama Backend
singularity pull ollama.sif docker://ollama/ollama:latest

# Pull AnythingLLM Frontend
singularity pull anythingllm.sif docker://mintplexlabs/anythingllm:latest

# Pull Reverse Proxy
singularity pull resolvr.sif docker://ghcr.io/sqoia-dev/resolvr:latest
```

### Model Directory Setup
To prevent users from downloading massive 40GB+ models into their home directories every time they launch a session, create a shared directory and pre-pull your standard models.

```bash
mkdir -p /opt/llm/models/models

# Open a shell into the container with the bind mount
singularity shell --bind /opt/llm/models/models:/root/.ollama/models /opt/llm/ollama.sif

# Start the daemon in the background
Singularity> ollama serve &

# Pull your cluster's supported models
Singularity> ollama pull command-r:latest
Singularity> ollama pull qwen2.5-coder:32b
Singularity> ollama pull llava:34b
Singularity> exit
```

---

## Installation & Configuration

1. **Clone the repository** into your OOD interactive apps directory:
   ```bash
   cd /var/www/ood/apps/sys
   git clone git@github.com:NessieCanCode/ood_anythingllm-ollama.git anythingllm-ollama
   ```

2. **Update Image Paths:**
   Edit `template/before.sh.erb` and update the paths to point to where you stored the `.sif` images on your cluster:
   ```bash
   export OLLAMA_SIF="/your/shared/path/ollama.sif"
   export ALLM_SIF="/your/shared/path/anythingllm.sif"
   export RESOLVR_SIF="/your/shared/path/resolvr.sif"
   export SHARED_MODELS="/your/shared/path/models/models"
   ```

3. **Configure Partitions & Hardware (`form.yml.erb`):**
   You **must** edit `form.yml.erb` to match your cluster's Slurm queues, accounts, and hardware specifications. 
   * Update the `public_partitions` block.
   * Update the `gpu_type` values to match the exact names defined in your cluster's `gres.conf` (e.g., if Slurm expects `--gres=gpu:a100:2`, your form value must be `a100`).

4. **Review Slurm Submission (`submit.yml.erb`):**
   If your cluster uses unique naming conventions for GPUs, utilize the translation block inside `submit.yml.erb` to map the web form values to your strict `slurm.conf` requirements before submission.

---

## The HPC Agent (MCP Integration) - Not Shared, but if interested, please reach out.
Model Context Protocol (MCP). This allows AnythingLLM to act as a cluster-aware agent.

This grants the AI (running as the unprivileged user) native access to common commands and context.

Users can chat with their workspace using `@agent`:
* *"How busy is the appliedmath queue right now?"*
* *"Show my running jobs."*
* *"What is my storage quota?"*

*(Note: Ensure `SLURM_CONF` in `before.sh.erb` points to your cluster's actual slurm.conf path).*

---

## Troubleshooting

* **Job fails immediately (Requested node configuration is not available):**
  This is a Slurm rejection, not an app failure. Ensure the `gpu_type` value requested in the form exactly matches the GRES naming convention in your `slurm.conf` / `gres.conf`.
* **Container crashes instantly:**
  Check the isolated logs generated in the user's OOD session directory (`logs/anythingllm.log` or `logs/ollama.log`). This is usually caused by a bad bind-mount path in `script.sh.erb` where the requested host directory does not exist.
* **Permission Denied on `.anythingllm_storage`:**
  Singularity runs as the host user. Do not attempt to force `UID` or `GID` environment variables to `1000` via Docker-style PUID flags, as this will crash the Singularity execution due to lack of `usermod` privileges. Let Singularity handle permissions natively.
