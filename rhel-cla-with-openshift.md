## Configuring the CLA to Use a Model Hosted in OpenShift AI

**The official Red Hat documentation does not currently describe a specific procedure for connecting the RHEL command-line assistant to a model hosted in OpenShift AI.** However, based on the architecture and documented configuration options, here is what is relevant:

1. **The CLA client connects to an endpoint** defined in `/etc/xdg/command-line-assistant/config.toml` via the `endpoint` setting. By default for online mode this is `https://cert.console.redhat.com/api/lightspeed/v1`, and for offline mode it points to the local container stack (e.g., `http://127.0.0.1:8501/v1`).

2. **The offline container stack** uses the `LLM` variable in `~/.config/rhel-cla/.env` to determine which model the `ramalama` container serves locally. The default is Microsoft Phi4-mini.

3. **The CLA is not documented to directly connect to an OpenShift AI model serving endpoint** (e.g., a vLLM InferenceService on OpenShift AI). The CLA expects a specific API contract from its backend (the `rlsapi`/`lightspeed-stack` container), which orchestrates the RAG database lookup and model inference together -- it is not a simple pass-through to a raw LLM endpoint.

4. The official docs include this warning: *"Changing the RHEL Offline Container model to unsupported models can enable the execution of arbitrary code or compromise the integrity of your RHEL systems."*

If your goal is to use OpenShift AI as the model backend, the recommended approach based on the product architecture would be to explore **Red Hat Lightspeed** (the broader platform) rather than the standalone CLA offline container, or to use the **goose-redhat** package which is explicitly designed to support "Bring Your Own Model" scenarios including connecting to external model providers.

### Sources

- [Interacting with the command-line assistant - RHEL 10 (includes disconnected install procedure)](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/10/html-single/interacting_with_the_command-line_assistant/index)
- [Interacting with the command-line assistant powered by RHEL Lightspeed - RHEL 9](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9/html-single/interacting_with_the_command-line_assistant_powered_by_rhel_lightspeed/index)
- [Optimize the RHEL command-line assistant tasks by using goose-redhat](https://access.redhat.com/articles/7142302)
