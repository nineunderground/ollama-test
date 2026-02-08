# Ollama Test Container

Simple Ollama setup for testing with a small model. Configured for network access from other hosts.

## Quick Start

```bash
# Start container
docker-compose up -d

# Pull a small model (~1.5GB)
docker exec ollama-test ollama pull tinyllama

# Or phi3:mini (~2GB, smarter)
docker exec ollama-test ollama pull phi3:mini

# Chat with it
docker exec -it ollama-test ollama run tinyllama
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
| `llama3.1:70b` | ~40GB | Best quality (needs lots of RAM) |

```bash
# Pull any model
docker exec ollama-test ollama pull <model_name>
```

## Stop

```bash
docker-compose down
```

## Troubleshooting

**Connection refused from other hosts:**
- Check firewall rules
- Verify port forwarding (WSL2)
- Ensure container is running: `docker ps`

**WSL IP changed after reboot:**
- Re-run the port forwarding commands
- Or create a startup script

**Model too slow:**
- Use a smaller model (tinyllama, phi3:mini)
- Allocate more RAM to Docker/WSL
