## Deploying the RHEL Command-Line Assistant Offline (Disconnected Environment)

> **Important:** The offline RHEL command-line assistant is **Developer Preview** software only. Red Hat does not support Developer Preview software and it is not production-ready.

**Source:** [Interacting with the command-line assistant - Red Hat Enterprise Linux 10](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/10/html-single/interacting_with_the_command-line_assistant/index) (the disconnected procedure also applies to RHEL 9.6+)

---

### Architecture

The offline CLA is delivered as a set of container images run with Podman:

| Container | Purpose |
|---|---|
| **installer** | Pulls other required containers, installs the `rhel-cla` command, optionally creates a systemd service |
| **rlsapi** (lightspeed-stack) | Provides the API endpoint the CLA client communicates with |
| **rag-database** (postgresql + knowledge-bridge) | RAG database supplementing the LLM with RHEL documentation |
| **ramalama** | Provides local LLM inference |
| **rhokp** | Red Hat Offline Knowledge Portal content |

---

### Prerequisites

- RHEL system registered to a **Red Hat Satellite** subscription (or via RHSM/RHC). Registration is required even for offline use ([KB: Does the offline RHEL command-line-assistant require the system to be registered?](https://access.redhat.com/solutions/7136542))
- **Podman** installed
- **container-tools** meta-package installed
- `$HOME/.config` and `$HOME/.local/bin` directories exist
- An **API key for the RHOKP container** (request via the Red Hat Offline Knowledge Portal or your Red Hat account representative)

**Hardware requirements:**

| System Type | Requirements |
|---|---|
| CPU-only (RHEL 9.6+/10+/Fedora 42/Windows 11) | 8 GB RAM, 2 CPU cores |
| GPU-capable (RHEL 9.6+/10+/Fedora 42/Windows 11) | 4 GB RAM, GPU with 4+ GB VRAM |
| macOS 15.x | M2 chip or newer |
| All systems | **35-40 GB** available disk space |

---

### Part 1: On the Internet-Connected Machine

**Step 1 - Authenticate to the Red Hat container registry:**

```bash
$ podman login registry.redhat.io
```

**Step 2 - Pull all required container images:**

```bash
$ podman pull registry.redhat.io/rhel-cla/installer-rhel10:latest
$ podman pull registry.access.redhat.com/hi/postgresql:18
$ podman pull registry.redhat.io/lightspeed-core/lightspeed-stack-rhel9:v0.6-latest
$ podman pull quay.io/ramalama/ramalama:latest
$ podman pull registry.redhat.io/rhel-cla/rhel-knowledge-bridge-rhel10
$ podman pull registry.redhat.io/offline-knowledge-portal/rhokp-rhel9:latest
```

**Step 3 - Save all images into a single tar archive:**

```bash
$ podman save -o rhel-cla-installer-images.tar \
    registry.redhat.io/rhel-cla/installer-rhel10:latest \
    registry.access.redhat.com/hi/postgresql:18 \
    registry.redhat.io/lightspeed-core/lightspeed-stack-rhel9:v0.6-latest \
    quay.io/ramalama/ramalama:latest \
    registry.redhat.io/rhel-cla/rhel-knowledge-bridge-rhel10 \
    registry.redhat.io/offline-knowledge-portal/rhokp-rhel9:latest
```

**Step 4 - Transfer the archive** to the disconnected system using a USB drive, network file copy, or other sneakernet method.

---

### Part 2: On the Disconnected RHEL 9 System

**Step 1 - Load the container images from the tar archive:**

```bash
$ podman load -i rhel-cla-installer-images.tar
```

**Step 2 - Ensure prerequisite directories exist:**

```bash
$ mkdir -p $HOME/.config $HOME/.local/bin
```

**Step 3 - Run the installer container:**

```bash
$ podman run -u $(id -u):$(id -g) --rm \
    -v $HOME/.config:/config:Z \
    -v $HOME/.local/bin:/config/.local/bin:Z \
    rhel-cla/installer-rhel10:latest
```

**Step 4 (Optional) - Enable automatic startup via systemd:**

```bash
$ rhel-cla enable-service
```

**Step 5 - Configure the GPU** (if applicable):

Edit `~/.config/rhel-cla/.env` and set:
- `LLAMACPP_IMAGE` - Set to the appropriate RamaLama container for your GPU hardware
  - For **NVIDIA GPU**: `quay.io/ramalama/cuda:0.14`
  - For **CPU-only**: the default image or `quay.io/ramalama/ramalama:latest`
- `HOST_DEVICE` - Set to your GPU device path
- NVIDIA-specific variables as needed (documented in comments within the `.env` file)
- `LLM` - Can be changed to use a different model (default is Microsoft Phi4-mini)

**Step 6 - Start the offline command-line assistant:**

```bash
$ rhel-cla start
```

You should see output including:

```
RHEL CLA pod is running! Services available at:
  API: http://localhost:8501
  Model Server: http://localhost:8888
  Database: localhost:5432
```

**Step 7 - Install the command-line-assistant client package:**

```bash
$ sudo dnf install command-line-assistant
```

> Note: The `command-line-assistant` RPM must be available via local repos, Satellite, or pre-installed, since this is a disconnected system.

**Step 8 - Configure the CLA client to use the local offline endpoint:**

Edit `/etc/xdg/command-line-assistant/config.toml`:

```toml
endpoint = "http://127.0.0.1:8501/v1"
```

If the offline CLA containers are hosted on a **different** system, point to that system's IP/hostname and open port 8501 in the firewall.

**Step 9 - Restart the CLA daemon:**

```bash
$ sudo systemctl restart clad
```

**Step 10 - Verify:**

```bash
$ c "what is an immutable file?"
```

---

### Configuring the CLA to Use a Model Hosted in OpenShift AI

**The official Red Hat documentation does not currently describe a specific procedure for connecting the RHEL command-line assistant to a model hosted in OpenShift AI.** However, based on the architecture and documented configuration options, here is what is relevant:

1. **The CLA client connects to an endpoint** defined in `/etc/xdg/command-line-assistant/config.toml` via the `endpoint` setting. By default for online mode this is `https://cert.console.redhat.com/api/lightspeed/v1`, and for offline mode it points to the local container stack (e.g., `http://127.0.0.1:8501/v1`).

2. **The offline container stack** uses the `LLM` variable in `~/.config/rhel-cla/.env` to determine which model the `ramalama` container serves locally. The default is Microsoft Phi4-mini.

3. **The CLA is not documented to directly connect to an OpenShift AI model serving endpoint** (e.g., a vLLM InferenceService on OpenShift AI). The CLA expects a specific API contract from its backend (the `rlsapi`/`lightspeed-stack` container), which orchestrates the RAG database lookup and model inference together -- it is not a simple pass-through to a raw LLM endpoint.

4. The official docs include this warning: *"Changing the RHEL Offline Container model to unsupported models can enable the execution of arbitrary code or compromise the integrity of your RHEL systems."*

If your goal is to use OpenShift AI as the model backend, the recommended approach based on the product architecture would be to explore **Red Hat Lightspeed** (the broader platform) rather than the standalone CLA offline container, or to use the **goose-redhat** package which is explicitly designed to support "Bring Your Own Model" scenarios including connecting to external model providers.

---

### Key Caveats

- **First query delay:** The first query in a session loads the model into memory, which may be slow.
- **CPU-only timeout:** The CLA client in RHEL 9.6 times out after 30 seconds (not configurable until RHEL 9.7+). CPU inference may exceed this timeout.
- **Single-system use:** The offline CLA is intended for individual system use, not multi-client deployments.
- **System registration required:** Even in offline mode, the RHEL system must be registered (via Satellite recommended, or RHSM/RHC).

### Sources

- [Interacting with the command-line assistant - RHEL 10 (includes disconnected install procedure)](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/10/html-single/interacting_with_the_command-line_assistant/index)
- [Interacting with the command-line assistant powered by RHEL Lightspeed - RHEL 9](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9/html-single/interacting_with_the_command-line_assistant_powered_by_rhel_lightspeed/index)
- [Does the offline RHEL command-line-assistant require the system to be registered?](https://access.redhat.com/solutions/7136542)
- [rhel-cla fails to start with ramalama error - workaround for .env configuration](https://access.redhat.com/solutions/7134825)
- Red Hat Blog: "Use the RHEL command-line assistant offline with this new developer preview" (Sept 2025)
