# rhel-lightspeed-local
RHEL LightSpeed Local Agent Deployment

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
podman run -u : --rm -v $HOME/.config:/config:Z -v $HOME/.local/bin:/config/.local/bin:Z quay.io/redhat-user-workloads/rhel-lightspeed-tenant/installer:latest install
```

```console
⚠️  Red Hat Enterprise Linux (RHEL) command line assistant is a tool that uses AI technology. Do not include any personal information or other sensitive information in your input. Results should not be relied on without human review. More information available: https://red.ht/3JqbWuu.

ℹ️  🚀 Installing rhel-lightspeed command...
✅ 🎉 RHEL Lightspeed installed!

ℹ️  📁 Files installed:
   • Command: ~/.local/bin/rhel-lightspeed
   • Config: ~/.config/rhel-lightspeed/.env
   • Runner: ~/.config/rhel-lightspeed/rhel-lightspeed-runner.sh

⚠️  ⚠️  NVIDIA GPU users (Linux): Edit ~/.config/rhel-lightspeed/.env and uncomment:
   LLAMACPP_IMAGE="quay.io/ramalama/cuda:latest"

⚠️  ⚠️  If running MacOS or Windows, make the command executable:
   chmod +x ~/.local/bin/rhel-lightspeed

⚠️  ⚠️  Make sure ~/.local/bin is in your PATH
   # For bash: echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc && source ~/.bashrc
   # For zsh:  echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zshrc && source ~/.zshrc
   # For fish: fish_add_path ~/.local/bin

ℹ️  🚀 Quick start:
   rhel-lightspeed start
   rhel-lightspeed status
   rhel-lightspeed stop

ℹ️  🗑️   Uninstall RHEL Lightspeed:
   rhel-lightspeed uninstall
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

endpoint = "http://10.0.35.196:8000/" # This is the RHEL GPU server running the lightspeed pod
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

- I have not tested this at all. Template files are under the openshift directory of this repository. Might work

# Potential Bugs?

- default env file shows this: HOST_DEVICE="/dev/dri" but uses image LLAMACPP_IMAGE="quay.io/ramalama/ramalama:latest" # These don't seem to go together

- rhel-lightspeed start will return but rhel-lightspeed status will show that the API is not available for 10-15 seconds. Maybe the start script should wait until the API is available?

- rhel-lightspeed stop command doesn't cleanup the containers or pods, especially if there is a failed start command
