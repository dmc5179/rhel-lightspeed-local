# rhel-lightspeed-local
RHEL LightSpeed Local Agent Deployment

## Mirroring


- Download your OpenShift pull-secret from console.redhat.com
- Setup authentication with container registry
```console
mkdir ~/.docker
cp pull-secret ~/.docker/config.json
```

- Create config dir that doesn't always exist for new users
```console
mkdir $HOME/.config
```

- May need to configure SELinux for device passthrough
```console
sudo setsebool -P container_use_devices true
```

- Command to install
```console
podman run -u : --rm -v $HOME/.config:/config:Z -v $HOME/.local/bin:/config/.local/bin:Z registry.redhat.io/rhel-cla/installer-rhel10:latest install-systemd
```

```console
⚠️  Red Hat Enterprise Linux (RHEL) command line assistant is a tool that uses AI technology. Do not include any personal information or other sensitive information in your input. Results should not be relied on without human review. More information available: https://red.ht/3JqbWuu.

ℹ️  Installing RHEL CLA systemd service...
✅ 🎉 RHEL RHEL CLA installed!

ℹ️  📁 Files installed:
   • Command: ~/.local/bin/rhel-cla
   • Config: ~/.config/rhel-cla/.env
   • Runner: ~/.config/rhel-cla/rhel-cla-runner.sh
   • Systemd service: ~/.config/systemd/user/rhel-cla.service

⚠️  ⚠️  Make sure ~/.local/bin is in your PATH:
   • For bash users:
 echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc && source ~/.bashrc'
   • For zsh users (macOS default):
 echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zshrc && source ~/.zshrc'
   • For fish users:
 fish_add_path ~/.local/bin

ℹ️  🚀 Quick start:
   rhel-cla start    # Start RHEL CLA
   rhel-cla status   # Check status
   rhel-cla stop     # Stop RHEL CLA

ℹ️  📊 View logs:
   journalctl --user -u rhel-cla -f     # Follow logs

ℹ️  🗑️   Uninstall RHEL CLA:
   rhel-cla uninstall
```






# Configuring GPU

- Install Nvidia kernel driver for RHEL
- Install CUDA 12.4 or higher
- sudo dnf install nvidia-ctk
- sudo mkdir /etc/cdi
- sudo nvidia-ctk cdi generate --output=/etc/cdi/nvidia.yaml # Must run each time the GPUs change like stopping and starting an EC2 GPU based instance

# Configuring another RHEL server to connect to RHEL lightspeed on your GPU box
# On the separate client machine I edited the file to set

- Install clad
```console
sudo dnf -y install command-line-assistant
```

- Edit clad config
```console
sudo vim /etc/xdg/command-line-assistant/config.toml
```

- If using an OpenShift Route
enpoint = "http://ocp-route:80"  # Router exposes 80 which forwards to service 8000

- Direct RHEL endpoint
endpoint = "http://10.0.35.196:8000/" # This is the RHEL GPU server running the lightspeed pod

- Disable SSL if needed
verify_ssl = false # Since we are not using SSL in this case. Maybe could add in a cert on the rhel-lightspeed side? Unsure

- Restart clad
```console
sudo systemctl restart clad
```

- Ask clad a question. The first time can take much longer to respond
```console
c "what is an immutable file?"
```

# OpenShift Deployment


# Potential Bugs?

- default env file shows this: HOST_DEVICE="/dev/dri" but uses image LLAMACPP_IMAGE="quay.io/ramalama/ramalama:latest" # These don't seem to go together

- rhel-lightspeed start will return but rhel-lightspeed status will show that the API is not available for 10-15 seconds. Maybe the start script should wait until the API is available?

- rhel-lightspeed stop command doesn't cleanup the containers or pods, especially if there is a failed start command




podman run -u : --rm -v $HOME/.config:/config:Z -v $HOME/.local/bin:/config/.local/bin:Z registry.redhat.io/rhel-cla/installer-rhel10:latest install-systemd
Error: lsetxattr(label=system_u:object_r:container_file_t:s0:c448,c649) /home/danclark/.config/google-chrome/NativeMessagingHosts/com.citrix.workspace.native.json: operation not permitted





# RHEL Lightspeed offline deployment

https://www.redhat.com/en/blog/use-rhel-command-line-assistant-offline-new-developer-preview

https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/interacting_with_the_command-line_assistant_powered_by_rhel_lightspeed/containerized-command-line-assistant-for-disconnected-environments

## RPM packages

command-line-assistant command-line-assistant-selinux python3-dasbus python3-greenlet python3-sqlalchemy python3-tomli

## Container Images

```console
podman pull registry.redhat.io/rhel-cla/installer-rhel10:latest
podman pull registry.redhat.io/rhel-cla/rag-database-rhel10:latest
podman pull quay.io/ramalama/ramalama:0.16.0
podman pull registry.redhat.io/rhel-cla/rlsapi-rhel10:latest
podman pull quay.io/redhat-user-workloads/rhel-lightspeed-tenant/okp-mcp
```



## MCP mode

https://github.com/rhel-lightspeed/okp-mcp/blob/main/README.md

