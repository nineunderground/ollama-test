# Ollama Test Container

Simple Ollama setup for testing with GPU support. Configured for network access from other hosts.

## Prerequisites

### GPU Support (WSL2 on Windows)

1. **Install NVIDIA GPU drivers on Windows** (not inside WSL)
   - Download from: https://www.nvidia.com/Download/index.aspx
   - The Windows driver includes WSL2 GPU support

2. **Verify GPU is visible in WSL:**
   ```bash
   nvidia-smi
   ```
   You should see your GPU(s) listed.

3. **Install NVIDIA Container Toolkit in WSL:**
   ```bash
   # Add NVIDIA package repository
   distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
   curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
   curl -s -L https://nvidia.github.io/libnvidia-container/$distribution/libnvidia-container.list | \
     sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
     sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

   # Install toolkit
   sudo apt-get update
   sudo apt-get install -y nvidia-container-toolkit

   # Configure Docker
   sudo nvidia-ctk runtime configure --runtime=docker
   sudo systemctl restart docker
   ```

4. **Verify Docker can see GPUs:**
   ```bash
   docker run --rm --gpus all nvidia/cuda:12.0-base nvidia-smi
   ```

## Quick Start

```bash
# Start container with GPU support
docker-compose up -d

# Verify GPU is available inside container
docker exec ollama-test nvidia-smi

# Pull a model
docker exec ollama-test ollama pull tinyllama

# Or a larger model (benefits more from GPU)
docker exec ollama-test ollama pull llama3.1:8b

# Chat with it
docker exec -it ollama-test ollama run tinyllama
```

## GPU Configuration

### Use All GPUs (default)
The default configuration uses all available GPUs:
```yaml
deploy:
  resources:
    reservations:
      devices:
        - driver: nvidia
          count: all
          capabilities: [gpu]
```

### Use a Specific GPU
If you have multiple GPUs and want to use only one:

**Option 1: In docker-compose.yml**
```yaml
deploy:
  resources:
    reservations:
      devices:
        - driver: nvidia
          device_ids: ['0']  # Use GPU 0 only
          capabilities: [gpu]
```

**Option 2: Via environment variable**
```yaml
environment:
  - CUDA_VISIBLE_DEVICES=1  # Use GPU 1 only
```

### Check Which GPU Ollama is Using
```bash
# Watch GPU usage while running a model
watch -n 1 nvidia-smi

# In another terminal, generate something
docker exec ollama-test ollama run llama3.1:8b "Tell me a story"
```

## API Usage (Local)

```bash
# Generate
curl http://localhost:11434/api/generate -d '{
  "model": "tinyllama",
  "prompt": "Hello!",
  "stream": false
}'

# Chat
curl http://localhost:11434/api/chat -d '{
  "model": "tinyllama",
  "messages": [{"role": "user", "content": "Hello!"}],
  "stream": false
}'

# List models
curl http://localhost:11434/api/tags
```

## Network Access (From Other Hosts)

This container is configured to accept connections from any IP (`0.0.0.0:11434`).

### 1. Get your host IP

```bash
# Linux/WSL
ip addr show eth0 | grep "inet "

# Or
hostname -I | awk '{print $1}'
```

### 2. WSL2 Additional Setup (Windows)

WSL2 uses a NAT network, so you need port forwarding from Windows to WSL:

```powershell
# PowerShell as Administrator

# Get WSL IP
$wslIp = (wsl hostname -I).Trim().Split(" ")[0]
echo "WSL IP: $wslIp"

# Add port forwarding
netsh interface portproxy add v4tov4 listenport=11434 listenaddress=0.0.0.0 connectport=11434 connectaddress=$wslIp

# Add firewall rule
netsh advfirewall firewall add rule name="Ollama API" dir=in action=allow protocol=TCP localport=11434

# Verify
netsh interface portproxy show all
```

To remove later:
```powershell
netsh interface portproxy delete v4tov4 listenport=11434 listenaddress=0.0.0.0
netsh advfirewall firewall delete rule name="Ollama API"
```

### 3. Test from another host

```bash
# Replace <HOST_IP> with your Windows/Linux host IP on the local network
curl http://<HOST_IP>:11434/api/tags

# Generate a response
curl http://<HOST_IP>:11434/api/generate -d '{
  "model": "tinyllama",
  "prompt": "Hello from remote!",
  "stream": false
}'
```

### 4. Use with OpenClaw

In your OpenClaw config:
```json
{
  "providers": {
    "ollama": {
      "baseUrl": "http://<HOST_IP>:11434"
    }
  }
}
```

## Available Models

| Model | Size | Best for |
|-------|------|----------|
| `tinyllama` | ~1.5GB | Quick tests, low resources |
| `phi3:mini` | ~2GB | Better quality, still small |
| `llama3.1:8b` | ~4.7GB | Good balance |
| `mistral:7b` | ~4GB | Great for coding |
| `qwen2.5:14b` | ~9GB | Strong reasoning |
| `llama3.1:70b` | ~40GB | Best quality (needs lots of VRAM) |

```bash
# Pull any model
docker exec ollama-test ollama pull <model_name>
```

## Stop

```bash
docker-compose down
```

## Troubleshooting

**GPU not detected in container:**
- Ensure NVIDIA drivers are installed on Windows (not WSL)
- Run `nvidia-smi` in WSL to verify GPU visibility
- Install nvidia-container-toolkit and restart Docker
- Check with: `docker run --rm --gpus all nvidia/cuda:12.0-base nvidia-smi`

**"could not select device driver" error:**
- NVIDIA Container Toolkit not installed or not configured
- Run: `sudo nvidia-ctk runtime configure --runtime=docker`
- Restart Docker: `sudo systemctl restart docker`

**Wrong GPU being used:**
- Set `CUDA_VISIBLE_DEVICES=0` or `device_ids: ['1']` in docker-compose.yml
- Check GPU memory usage with `nvidia-smi` while running inference

**Connection refused from other hosts:**
- Check firewall rules
- Verify port forwarding (WSL2)
- Ensure container is running: `docker ps`

**WSL IP changed after reboot:**
- Re-run the port forwarding commands
- Or create a startup script

**Model too slow even with GPU:**
- Verify GPU is actually being used (`nvidia-smi` shows memory usage)
- Try a quantized model variant (e.g., `llama3.1:8b-q4_0`)
- Check if model fits in VRAM (otherwise it uses CPU)
