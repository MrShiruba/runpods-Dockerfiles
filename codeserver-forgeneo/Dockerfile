# ---- Base image ----
FROM nvidia/cuda:13.0.0-runtime-ubuntu22.04

ENV DEBIAN_FRONTEND=noninteractive

# ---- Install basic packages + Python 3.11 ----
RUN apt-get update && apt-get install -y \
    software-properties-common curl wget git vim unzip htop tree ripgrep ffmpeg build-essential \
    libgl1 libglib2.0-0 libcairo2-dev pkg-config \
    && add-apt-repository ppa:deadsnakes/ppa \
    && apt-get update && apt-get install -y \
    python3.11 python3.11-venv python3.11-distutils python3.11-dev python3-pip \
    && update-alternatives --install /usr/bin/python3 python3 /usr/bin/python3.11 2 \
    && rm -rf /var/lib/apt/lists/*

# ---- Install code-server ----
RUN curl -fsSL https://code-server.dev/install.sh | sh

# ---- Configure code-server (disable password authentication + dark theme) ----
RUN mkdir -p /root/.config/code-server && \
cat << 'EOF' > /root/.config/code-server/config.yaml
bind-addr: 0.0.0.0:7777
auth: password
password: runpod-super-secret-password
cert: false
user-data-dir: /root/.config/code-server/user-data
EOF

# ---- Ensure dark theme settings ----
RUN mkdir -p /root/.config/code-server/user-data/User && \
cat << 'EOF' > /root/.config/code-server/user-data/User/settings.json
{
    "workbench.colorTheme": "Default Dark Modern",
    "chat.editor.view.enabled": false,
    "chat.inlineChat.enabled": false,
    "chat.experimental.composeIntoChat": false,
    "workbench.experimental.action.showAIChat": false,
    "chat.sideBySideLayout": "hidden",
    "workbench.secondarySideBar.defaultVisibility": "hidden",
    "workbench.panel.opensMaximized": "never",
    "workbench.panel.defaultLocation": "right"
}
EOF

# ---- Set working directory ----
WORKDIR /workspace

# ---- Install RunPod File Uploader ----
RUN curl -sSL https://github.com/kodxana/RunPod-FilleUploader/raw/main/scripts/installer.sh -o installer.sh \
    && chmod +x installer.sh \
    && ./installer.sh \
    && rm installer.sh

# ---- Startup script embedded in Dockerfile ----
RUN cat << 'EOF' > /start-all.sh
#!/bin/bash
set -e

# Ensure workspace and logs exist
mkdir -p /workspace/logs
cd /workspace

# Clone Forge Neo if missing
if [ ! -d "sd-webui-forge-neo" ]; then
    git clone https://github.com/Haoming02/sd-webui-forge-classic sd-webui-forge-neo --branch neo
fi

cd sd-webui-forge-neo

# Setup virtual environment if missing
if [ ! -d "venv" ]; then
    python3.11 -m venv venv
    source venv/bin/activate
    pip install --upgrade pip
    pip install torch torchvision --index-url https://download.pytorch.org/whl/cu130
    pip install -r requirements.txt
else
    source venv/bin/activate
fi

# Start Forge Neo and uploader in background - with logs redirected
venv/bin/python3 launch.py --listen --port 3000 --xformers --cuda-stream --cuda-malloc --pin-shared-memory \
    >> /workspace/logs/forge-neo.log 2>&1 &

runpod-uploader --host 0.0.0.0 --port 2999 \
    >> /workspace/logs/runpod-uploader.log 2>&1 &

# Start code-server with log redirection as the main process
exec code-server --host 0.0.0.0 --port 7777 /workspace \
    >> /workspace/logs/code-server.log 2>&1
EOF

RUN chmod +x /start-all.sh

EXPOSE 2999 3000 7777

# ---- Run startup script at container start ----
CMD ["/start-all.sh"]
