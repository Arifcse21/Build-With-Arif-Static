If you've ever wanted to run a local AI assistant or chatbot with a beautiful web interface on your own machine, you're not alone. Tools like **Ollama** (for running open-source LLMs) and **Open WebUI** (a beautiful chat frontend) make this surprisingly accessible. 

In this ariticle, I will walk you through a clean Terraform configuration that deploys both services in Docker containers with support for **CPU-only** systems *and* optional **NVIDIA GPU acceleration**.

---

### The Architecture at a Glance

- **Ollama**: Runs the language model (e.g., `llama3`, `qwen`, `mistral`) and exposes an API on port `11434`.
- **Open WebUI**: A user-friendly web app that connects to Ollama’s API, served on port `3000`.
- Both containers live on a shared Docker network (`ollama-network`) so they can talk to each other using container names (`http://ollama:11434`).
- Persistent data is stored in Docker volumes:
  - `ollama_data`: Caches downloaded models.
  - `open_webui_data`: Saves conversations, settings, and user data.


---

### The Terraform Setup (CPU-First)

Here’s the core of our `main.tf`:

```hcl
terraform {
  required_providers {
    docker = {
      source  = "kreuzwerker/docker"
      version = "~> 3.6"
    }
  }
}

provider "docker" {}

resource "docker_network" "ollama_net" {
  name   = "ollama-network"
  driver = "bridge"
}

resource "docker_volume" "ollama_data" {
  name = "ollama_data"
  lifecycle {
    prevent_destroy = true  # Don’t accidentally wipe your models!
  }
}

resource "docker_volume" "open_webui_data" {
  name = "open_webui_data"
}

resource "docker_container" "ollama" {
  name  = "ollama"
  image = "ollama/ollama:latest"

  ports {
    external = 11434
    internal = 11434
  }

  env = [
    "OLLAMA_HOST=0.0.0.0",
    "OLLAMA_ORIGINS=*",
    "OLLAMA_MAX_LOADED_MODELS=1",  # One model at a time (saves RAM)
    "OLLAMA_KEEP_ALIVE=15m",
    "OLLAMA_NOPRUNE=true"
  ]

  networks_advanced {
    name = docker_network.ollama_net.name
  }

  volumes {
    volume_name    = docker_volume.ollama_data.name
    container_path = "/root/.ollama"
  }

  memory = 8589934592  # ~8 GB RAM
  restart = "unless-stopped"
}

resource "docker_container" "open_webui" {
  name  = "open-webui"
  image = "ghcr.io/open-webui/open-webui:main"

  networks_advanced {
    name = docker_network.ollama_net.name
  }

  volumes {
    volume_name    = docker_volume.open_webui_data.name
    container_path = "/app/backend/data"
  }

  env = [
    "OLLAMA_BASE_URL=http://ollama:11434",
    "WEBUI_HOST=0.0.0.0",
    "WEBUI_PORT=8080"
  ]

  ports {
    internal = 8080
    external = 3000
  }

  memory = 2147483648  # ~2 GB RAM
  restart = "unless-stopped"
  depends_on = [docker_container.ollama]
}
```

Apply it with:
```bash
terraform init
terraform plan 
terraform apply
```

Then visit `http://localhost:3000` 

---

### Why You Need Volumes?

Containers are ephemeral by nature. Means if you stop or remove a container, everything inside it goes away.
Here, your downloaded models (which can be 6-8GB each, sometimes more) and chat history would disappear. By mounting Docker volumes:

- `/root/.ollama`: `ollama_data`: Persists models across runs.
- `/app/backend/data`: `open_webui_data`: Keeps your UI settings and conversations.

The `prevent_destroy = true` on `ollama_data` is a safety measure: it stops Terraform from deleting the volume even if you destroy the stack (unless you explicitly override it). Smart for expensive-to-download assets!
you can manually delete it with `docker volume rm ollama_data`

---

### Adding NVIDIA GPU Support (Optional)

By default, this setup uses your CPU. But if you have an NVIDIA GPU and want faster inference, you can enable GPU access with the **NVIDIA Container Toolkit**.

#### Step 1: Install the Conatainer Toolkit
Follow the official guide:  
https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html

On Ubuntu, it is usually:
```bash
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
  sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
  sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
sudo apt update && sudo apt install -y nvidia-container-toolkit
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

#### Step 2: Adjust Your Terraform

Add the `gpu` block in the Ollama container:

```hcl
resource "docker_container" "ollama" {
  # ... existing config ...

  # Enable all GPUs
  gpu {
    mode = "all"
  }

  # Optional: fine-tune CPU affinity
  # cpu_set = "0-3"  # Use cores 0 through 3
}
```

Now you can run `terraform apply` again.

> Note: The `gpu` block requires Docker provider v3.6+ and a Docker daemon configured with NVIDIA runtime.

Now Ollama will leverage your GPU for inference, often **2–5x faster** than CPU, depending on your model and hardware.

---

### Security & Tuning Tips

- **Don’t expose Ollama publicly**: It has no auth by default. Keep port `11434` bound to `localhost` unless behind a reverse proxy with auth.
- **Limit models**: `OLLAMA_MAX_LOADED_MODELS=1` prevents memory overload on smaller systems.
- **Adjust RAM**: If you have less than 8 GB free, reduce `memory` (but expect swapping or `*CRASHES*` with larger models).

---

### Try It Yourself

This setup works on:
- Linux (with Docker + optional NVIDIA toolkit)
- macOS (Docker Desktop CPU only, no GPU)
- Windows (WSL2 + Docker Desktop CPU only)
---

### Final Thoughts

Local LLMs are no longer just for researchers or cloud budgets. With Ollama, Open WebUI, and a bit of Terraform magic, you can have a responsive, private, and fully controllable AI assistant running on your own hardware, whether it’s a Raspberry Pi (okay, maybe not… yet) or a gaming rig with an RTX 4090.

And because everything is defined as code, sharing or recreating your setup is as easy as `terraform apply`.

Happy prompting! 🦙✨