# ---- Base image ----
FROM nvidia/cuda:13.0.0-runtime-ubuntu22.04

ENV DEBIAN_FRONTEND=noninteractive

# ---- Install basic packages + Python 3.13 (Forge/AI-Toolkit) + Python 3.11 (WanGP) + Node.js (for ai-toolkit UI) ----
RUN apt-get update && apt-get install -y \
    software-properties-common curl wget git vim unzip htop lsof tree ripgrep ffmpeg build-essential \
    libgl1 libglib2.0-0 libcairo2-dev pkg-config jq \
    && curl -fsSL https://deb.nodesource.com/setup_22.x | bash - \
    && add-apt-repository ppa:deadsnakes/ppa \
    && apt-get update && apt-get install -y \
    python3.13 python3.13-venv python3.13-dev python3-tk python-is-python3 nodejs \
    python3.11 python3.11-venv python3.11-dev \
    && rm -rf /var/lib/apt/lists/*

# ---- Install code-server ----
RUN curl -fsSL https://code-server.dev/install.sh | sh

# ---- Configure code-server (disable password authentication + dark theme) ----
RUN mkdir -p /root/.config/code-server && \
cat << 'EOF' > /root/.config/code-server/config.yaml
bind-addr: 0.0.0.0:7777
auth: password
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

# ---- Install VS Code extensions ----
RUN code-server --user-data-dir /root/.config/code-server/user-data --install-extension lyuwenhan.zipper

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

# ----------------------------------------------------
# 1. SETUP FORGE NEO
# ----------------------------------------------------
if [ ! -d "sd-webui-forge-neo" ]; then
    git clone https://github.com/Haoming02/sd-webui-forge-classic sd-webui-forge-neo --branch neo
fi

cd /workspace/sd-webui-forge-neo

cat << 'REQEOF' > requirements.txt
GitPython==3.1.45
Pillow==10.4.0
accelerate==1.10.1
clean-fid==0.1.35
diffusers==0.35.2
diskcache==5.6.3
einops==0.8.1
facexlib==0.3.0
fastapi==0.112.4
httpcore==0.15.0
httpx==0.24.1
huggingface-hub==0.34.6
inflection==0.5.1
joblib==1.5.1
jsonmerge==1.8.0
kornia==0.6.7
lark==1.2.2
lightning==2.5.6
loadimg==0.1.2
numpy==1.26.4
omegaconf==2.2.3
open-clip-torch==2.32.0
opencv-python==4.8.1.78
peft==0.17.1
piexif==1.1.3
pillow-heif==0.22.0
pillow-jxl-plugin==1.3.6
protobuf==4.25.8
psutil==5.9.8
pydantic-core==2.23.4
pydantic==2.9.2
resize-right==0.0.2
safetensors==0.7.0
scikit-image==0.25.2
setuptools==69.5.1
spandrel-extra-arches==0.2.0
spandrel==0.4.1
tomesd==0.1.3
torchdiffeq==0.2.5
torchsde==0.2.6
tqdm==4.67.1
transformers==4.56.2
REQEOF

if [ ! -d "venv" ]; then
    echo "Creating virtual environment for Forge Neo..."
    python3.13 -m venv venv
    source venv/bin/activate
    pip install --upgrade pip
    pip install torch torchvision --index-url https://download.pytorch.org/whl/cu130
    pip install comfy_kitchen audioop-lts av
    pip install -r requirements.txt
    deactivate
fi

# ----------------------------------------------------
# 2. SETUP AI-TOOLKIT (Chacun son venv !)
# ----------------------------------------------------
cd /workspace
if [ ! -d "ai-toolkit" ]; then
    echo "Cloning AI-Toolkit..."
    git clone https://github.com/ostris/ai-toolkit.git
    cd ai-toolkit
    git submodule update --init --recursive
else
    cd ai-toolkit
fi

if [ ! -d "venv" ]; then
    echo "Creating virtual environment for AI-Toolkit..."
    python3.13 -m venv venv
    source venv/bin/activate
    pip install --upgrade pip
    pip install torch torchvision --index-url https://download.pytorch.org/whl/cu130
    pip install python-dotenv
    pip install -r requirements.txt
    deactivate
fi

if [ -d "ui" ] && [ ! -d "ui/node_modules" ]; then
    echo "Setting up AI-Toolkit UI..."
    cd ui
    npm install
    cd ..
fi

# ----------------------------------------------------
# 3. SETUP WANGP (MiniMax H3) - Python 3.11 + torch 2.10 cu130 (versions recommandees par WanGP)
# ----------------------------------------------------
cd /workspace
if [ ! -d "Wan2GP" ]; then
    echo "Cloning WanGP..."
    git clone https://github.com/deepbeepmeep/Wan2GP.git
fi

cd /workspace/Wan2GP

if [ ! -d "venv" ]; then
    echo "Creating virtual environment for WanGP..."
    python3.11 -m venv venv
    source venv/bin/activate
    pip install --upgrade pip
    pip install torch==2.10.0 torchvision==0.25.0 torchaudio==2.10.0 --index-url https://download.pytorch.org/whl/cu130
    pip install -r requirements.txt
    deactivate
fi

# ----------------------------------------------------
# 4. LAUNCH ALL SERVICES
# ----------------------------------------------------
export PYTORCH_VERSION="2.12.1+cu130"

# Start Forge Neo (Utilise son propre venv)
echo "Starting Forge Neo on port 3000..."
cd /workspace/sd-webui-forge-neo
nohup ./venv/bin/python3 -u launch.py \
    --port 3000 \
    --listen \
    --api \
    --xformers \
    --cuda-stream \
    --cuda-malloc \
    --pin-shared-memory \
    --enable-insecure-extension-access \
    > /workspace/logs/forge-neo.log 2>&1 &

# Start AI-Toolkit UI (On force le PATH vers le venv d'AI-Toolkit pour ce processus et ses enfants)
echo "Starting AI-Toolkit UI on port 8675..."
cd /workspace/ai-toolkit/ui
nohup env PATH="/workspace/ai-toolkit/venv/bin:$PATH" npm run build_and_start > /workspace/logs/ai-toolkit-ui.log 2>&1 &

# Start WanGP (Port 7860) - mot de passe obligatoire car l'URL RunPod est publique
echo "Starting WanGP on port 7860..."
cd /workspace/Wan2GP
nohup env WANGP_AUTH_PASSWORD="${WANGP_PASSWORD:-change-moi-wangp}" \
    ./venv/bin/python -u wgp.py \
    --listen \
    --server-port 7860 \
    --public-url "https://${RUNPOD_POD_ID}-7860.proxy.runpod.net" \
    --auth \
    > /workspace/logs/wangp.log 2>&1 &

# Start RunPod Uploader (Port 2999)
runpod-uploader --host 0.0.0.0 --port 2999 --max-size 107374182400 \
    > /workspace/logs/runpod-uploader.log 2>&1 &

# Start Code-Server (Port 7777) - mot de passe lu depuis la variable d'env CODE_SERVER_PASSWORD
export PASSWORD="${CODE_SERVER_PASSWORD:-change-moi-codeserver}"
exec code-server --host 0.0.0.0 --port 7777 /workspace \
    > /workspace/logs/code-server.log 2>&1
EOF

RUN sed -i 's/\r$//' /start-all.sh && chmod +x /start-all.sh

# Expose ports for HTTP access in RunPod
EXPOSE 2999 3000 7777 7860 8675

CMD ["/start-all.sh"]
