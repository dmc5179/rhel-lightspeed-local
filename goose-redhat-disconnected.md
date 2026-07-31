## Installing and Configuring goose-redhat on a Disconnected RHEL System

This guide covers installing the `goose-redhat` package on a RHEL system without commercial internet access and configuring it to use either a locally-run LLM or a remotely-hosted LLM (such as one served by OpenShift AI).

**Sources:**
- [Optimize the RHEL command-line assistant tasks by using goose-redhat](https://access.redhat.com/articles/7142302)
- [Red Hat AI Inference Server - Getting Started](https://access.redhat.com/documentation/en-us/red_hat_ai_inference_server/3.4/html-single/getting_started/index)
- [Red Hat Extensions Repository Support Policy](https://access.redhat.com/support/policy/updates/rhel-extensions)
- [Upstream goose documentation - Supported LLM Providers](https://goose-docs.ai/docs/getting-started/providers)

---

### What is goose-redhat?

`goose-redhat` is a configuration package that pre-configures the open-source [goose](https://github.com/block/goose) AI agent framework with Red Hat-specific extensions and contexts optimized for RHEL environments.

Unlike the RHEL command-line assistant (CLA), which is read-only and advisory, goose is an **autonomous agent** that can execute commands, modify files, and orchestrate multi-step workflows on your system. It supports "Bring Your Own Model" (BYOM) -- you can connect it to any compatible LLM provider including local inference servers.

---

### RPM Packages Required

The following RPMs are installed from the **RHEL Extensions** repository:

| Package | Description |
|---|---|
| `goose` | Core agent framework and CLI tool (e.g., `goose-1.23.2-1.el9_8.x86_64`) |
| `goose-redhat` | Red Hat-specific configuration, extensions, and pre-configured contexts |
| `goose-proxy-selinux` | SELinux policy module for the goose proxy (e.g., `goose-proxy-selinux-1.0.0-1.el9_8.noarch`) |

> **Warning:** Do not remove the `goose-redhat` package independently. Removing it breaks the installation and leaves the system with only the `goose-proxy-selinux` and `goose` packages in an incomplete state.

**Availability:**
- RHEL 9.8+ (repo: `rhel-9-for-$(arch)-extensions-rpms`)
- RHEL 10.2+ (repo: `rhel-10-for-$(arch)-extensions-rpms`)

---

### Part 1: Preparing RPMs on the Connected System

Since the disconnected system cannot reach Red Hat repositories directly, you need to download the RPMs on a connected system and transfer them.

**Step 1 - Register the system and enable the RHEL Extensions repository:**

For RHEL 9:

```bash
$ sudo subscription-manager register
$ sudo subscription-manager repos --enable=rhel-9-for-$(arch)-extensions-rpms
```

For RHEL 10:

```bash
$ sudo subscription-manager register
$ sudo subscription-manager repos --enable=rhel-10-for-$(arch)-extensions-rpms
```

**Step 2 - Download the RPMs without installing (including dependencies):**

```bash
$ sudo dnf download --resolve --destdir=/tmp/goose-rpms goose-redhat
```

This downloads `goose`, `goose-redhat`, `goose-proxy-selinux`, and any dependencies into `/tmp/goose-rpms`.

**Step 3 - Transfer the RPMs** to the disconnected system via USB drive, SCP over a local network, or other sneakernet method.

---

### Part 2: Installing RPMs on the Disconnected System

**Step 1 - Copy the RPMs to the disconnected system** (e.g., to `/tmp/goose-rpms/`).

**Step 2 - Install from local files:**

```bash
$ sudo dnf install /tmp/goose-rpms/*.rpm
```

**Step 3 - Open a new shell session for changes to take effect:**

```bash
$ bash
```

**Step 4 - Verify the installation:**

```bash
$ goose --version
```

---

### Part 3: Providing an LLM for the Disconnected System

In a disconnected environment, you cannot use cloud-based LLM providers. You have two options:

- **Option A:** Run a local LLM on the same RHEL system
- **Option B:** Point goose to a model served elsewhere on your network (e.g., OpenShift AI)

---

### Option A: Running a Local LLM on the Same System

There are multiple ways to serve a local LLM. The two most relevant for RHEL are **Ollama** and **Red Hat AI Inference Server (vLLM)**.

#### Option A1: Using Ollama (Simplest for CPU or GPU)

Ollama provides a simple way to run models locally and exposes an API endpoint that goose natively supports.

**On the connected system -- download Ollama and a model:**

1. Download the Ollama binary/installer from `https://ollama.com/download` and transfer it to the disconnected system.

2. Pull a model that supports tool-calling (required by goose):

```bash
$ ollama pull qwen2.5
```

The model files are stored under `~/.ollama/models/`. Transfer this entire directory to the disconnected system.

**On the disconnected system:**

1. Install Ollama and place the model files in `~/.ollama/models/`.

2. Start the Ollama server:

```bash
$ ollama serve
```

Ollama listens on `http://localhost:11434` by default.

3. Verify the model is available:

```bash
$ ollama list
```

4. Configure goose to use Ollama:

```bash
$ goose configure
```

When prompted:
- Select **Ollama** as the provider
- Enter host: `http://localhost:11434` (the default)
- Select or enter the model name (e.g., `qwen2.5`)

Alternatively, set environment variables:

```bash
$ export OLLAMA_HOST="http://localhost:11434"
```

5. Verify:

```bash
$ goose info
$ goose run -t "What is the current OS version and kernel release?"
```

#### Option A2: Using Red Hat AI Inference Server (vLLM)

For GPU-equipped systems, you can use the Red Hat AI Inference Server container which provides enterprise-grade, OpenAI-compatible model serving via vLLM.

**On the connected system -- pull the container image and model:**

1. Log in to the Red Hat registry and pull the vLLM container:

```bash
$ podman login registry.redhat.io
$ podman pull registry.redhat.io/rhaiis/vllm-cuda-rhel9:latest
```

2. Save the container image:

```bash
$ podman save -o rhaiis-vllm.tar registry.redhat.io/rhaiis/vllm-cuda-rhel9:latest
```

3. Download the model. For example, for `RedHatAI/Llama-3.2-1B-Instruct-FP8`:

```bash
$ pip install huggingface-hub
$ huggingface-cli download RedHatAI/Llama-3.2-1B-Instruct-FP8 \
    --local-dir ./rhaiis-cache/huggingface
```

4. Transfer `rhaiis-vllm.tar` and `rhaiis-cache/` to the disconnected system.

**On the disconnected system:**

1. Load the container image:

```bash
$ podman load -i rhaiis-vllm.tar
```

2. Start the vLLM inference server with the locally-stored model:

```bash
$ podman run --rm -it \
    --device nvidia.com/gpu=all \
    --security-opt=label=disable \
    --shm-size=4GB -p 8000:8000 \
    --userns=keep-id:uid=1001 \
    --env "HF_HUB_OFFLINE=1" \
    --env=VLLM_NO_USAGE_STATS=1 \
    -v ./rhaiis-cache:/opt/app-root/src/.cache:Z \
    registry.redhat.io/rhaiis/vllm-cuda-rhel9:latest \
    --model /opt/app-root/src/.cache/huggingface \
    --served-model-name my-local-model
```

The server listens on `http://localhost:8000` and provides an OpenAI-compatible API.

3. Configure goose to use the vLLM endpoint via the OpenAI provider:

```bash
$ export OPENAI_HOST="http://localhost:8000"
$ export OPENAI_API_KEY="not-needed"
```

Then run:

```bash
$ goose configure
```

When prompted:
- Select **OpenAI** as the provider
- Set `OPENAI_HOST` to `http://localhost:8000`
- Set `OPENAI_API_KEY` to any non-empty string (vLLM does not require a real key by default)
- Enter the model name that matches `--served-model-name` (e.g., `my-local-model`)

Alternatively, create a custom provider JSON file at `~/.config/goose/custom_providers/local_vllm.json`:

```json
{
  "name": "local_vllm",
  "engine": "openai",
  "display_name": "Local vLLM",
  "description": "Local Red Hat AI Inference Server",
  "api_key_env": "OPENAI_API_KEY",
  "base_url": "http://localhost:8000/v1/chat/completions",
  "models": [
    {
      "name": "my-local-model",
      "context_limit": 16384
    }
  ],
  "supports_streaming": true,
  "requires_auth": false
}
```

4. Verify:

```bash
$ goose info
$ goose run -t "What is the current OS version?"
```

#### Option A3: Using Ramalama

Ramalama is API-compatible with Ollama and is available via the `ramalama` container image:

```bash
$ ramalama serve --runtime-args="--jinja" ollama://qwen2.5
```

Configure goose to use the Ollama provider with:

```bash
$ export OLLAMA_HOST="http://0.0.0.0:8080"
```

---

### Option B: Using a Remote LLM (OpenShift AI or Other Network-Accessible Endpoint)

If your disconnected environment has an OpenShift AI cluster (or any other system) serving a model via an OpenAI-compatible endpoint (e.g., vLLM via KServe InferenceService), you can point goose directly to it.

#### Prerequisites

- The remote model serving endpoint must be network-reachable from the RHEL system running goose
- The endpoint must expose an **OpenAI-compatible API** (e.g., `/v1/chat/completions`). Red Hat AI Inference Server (vLLM) and OpenShift AI's KServe-based model serving both provide this.
- You need the endpoint URL and any authentication token/API key

#### Determining the OpenShift AI Endpoint

If the model is served via OpenShift AI with KServe, the inference endpoint URL typically follows this pattern:

```
https://<inference-service-name>-<namespace>.apps.<cluster-domain>/v1
```

You can find it in the OpenShift AI dashboard under the model's deployment details, or via:

```bash
$ oc get inferenceservice <name> -n <namespace> -o jsonpath='{.status.url}'
```

You will also need the authentication token:

```bash
$ oc whoami -t
```

#### Configuring goose for the Remote Endpoint

**Method 1: Environment variables (simplest)**

```bash
$ export OPENAI_HOST="https://<your-inference-endpoint>"
$ export OPENAI_API_KEY="<your-auth-token>"
```

Then configure goose:

```bash
$ goose configure
```

Select **OpenAI** as the provider and enter the model name that matches the `--served-model-name` configured on the remote server.

**Method 2: Custom provider JSON file**

Create `~/.config/goose/custom_providers/openshift_ai.json`:

```json
{
  "name": "openshift_ai",
  "engine": "openai",
  "display_name": "OpenShift AI Model",
  "description": "Model served by OpenShift AI",
  "api_key_env": "OPENSHIFT_AI_API_KEY",
  "base_url": "https://<inference-service-name>-<namespace>.apps.<cluster-domain>/v1/chat/completions",
  "models": [
    {
      "name": "<served-model-name>",
      "context_limit": 16384
    }
  ],
  "supports_streaming": true,
  "requires_auth": true
}
```

Then set the API key and start goose:

```bash
$ export OPENSHIFT_AI_API_KEY="<your-auth-token>"
$ goose session start --provider openshift_ai
```

**Method 3: Using LiteLLM provider** (if you have a LiteLLM proxy fronting multiple models)

```bash
$ export LITELLM_HOST="https://<your-litellm-proxy>"
$ export LITELLM_BASE_PATH="v1/chat/completions"
$ export LITELLM_API_KEY="<your-key>"
```

#### TLS Considerations in Disconnected Environments

If your OpenShift AI cluster uses a self-signed or internal CA certificate, you may need to:

1. Add the CA certificate to the system trust store:

```bash
$ sudo cp custom-ca.pem /etc/pki/ca-trust/source/anchors/
$ sudo update-ca-trust
```

2. Or set the `REQUESTS_CA_BUNDLE` or `SSL_CERT_FILE` environment variable to point to your CA bundle:

```bash
$ export REQUESTS_CA_BUNDLE="/etc/pki/ca-trust/extracted/pem/tls-ca-bundle.pem"
```

---

### Verification and Usage

Once goose is configured with any provider, verify the setup:

```bash
# Check goose version and provider info
$ goose --version
$ goose info

# Run a simple test
$ goose run -t "What is the current OS version and kernel release?"

# Start an interactive session
$ goose session start
```

goose can execute system commands, inspect logs, modify files, and chain complex workflows -- all with your approval at each step.

---

### Summary: What to Transfer to the Disconnected System

| Component | Source | Purpose |
|---|---|---|
| `goose-redhat` RPMs | RHEL Extensions repo (`dnf download --resolve`) | goose agent + Red Hat config |
| Ollama binary + model files | ollama.com / `ollama pull` | Option A1: Local LLM via Ollama |
| `rhaiis/vllm-cuda-rhel9` container + model | registry.redhat.io / HuggingFace | Option A2: Local LLM via RHAIIS |
| `ramalama` container | quay.io/ramalama | Option A3: Local LLM via Ramalama |
| *(None for remote)* | *(Network access to endpoint)* | Option B: Remote LLM on OpenShift AI |

### Key Differences: Local vs. Remote LLM

| Aspect | Local LLM (same system) | Remote LLM (OpenShift AI) |
|---|---|---|
| **Hardware** | Requires GPU (recommended) or sufficient CPU/RAM on the RHEL system | GPU/hardware on the remote cluster |
| **Model quality** | Limited by local hardware (smaller models) | Can serve larger, higher-quality models |
| **Network** | No network needed | Requires network path to the serving endpoint |
| **Provider in goose** | Ollama or OpenAI (pointing to localhost) | OpenAI (pointing to remote endpoint) |
| **Latency** | Low (local) | Depends on network |

### Sources

- [Optimize the RHEL command-line assistant tasks by using goose-redhat](https://access.redhat.com/articles/7142302)
- [Red Hat AI Inference Server - Getting Started](https://access.redhat.com/documentation/en-us/red_hat_ai_inference_server/3.4/html-single/getting_started/index)
- [Red Hat AI Inference Server - Serving local downloaded LLM model](https://access.redhat.com/solutions/7134852)
- [Upstream goose documentation - Supported LLM Providers](https://goose-docs.ai/docs/getting-started/providers)
- [Red Hat Extensions Repository Support Policy](https://access.redhat.com/support/policy/updates/rhel-extensions)
