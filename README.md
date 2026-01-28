# Ollama Test Container

Simple Ollama setup for testing with a small model.

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

## API Usage

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
```

## Stop

```bash
docker-compose down
```
