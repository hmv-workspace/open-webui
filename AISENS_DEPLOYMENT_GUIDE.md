# AiSens Deployment Guide

## Overview

This guide documents the process of rebranding Open WebUI to AiSens and deploying it to a GCP VM with Ollama running separately. The deployment uses Docker for containerization and supports both Docker Compose and direct Docker run methods.

## Prerequisites

- GCP VM with Docker installed
- Ollama deployed separately (accessible via HTTP/HTTPS)
- Git repository access to the forked Open WebUI codebase
- Basic knowledge of Docker and command-line operations

## Step 1: Environment Setup

### Install Docker on GCP VM
```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker $USER

# Restart session or run: newgrp docker
```

### Clone Repository
```bash
git clone https://github.com/your-username/open-webui.git
cd open-webui
```

## Step 2: Rebranding to AiSens

### Core Configuration Changes

**File: `src/lib/constants.ts`**
```typescript
export const APP_NAME = 'AiSens';
```

**File: `package.json`**
```json
{
  "name": "aisens",
  "version": "0.6.26",
  ...
}
```

**File: `static/site.webmanifest`**
```json
{
  "name": "AiSens",
  "short_name": "AiSens",
  ...
}
```

### Documentation Updates

**File: `README.md`**
- Change title: `# AiSens 👋`
- Update all references from "Open WebUI" to "AiSens"
- Update URLs and links as needed

### Translation Files
Update translation files in `src/lib/i18n/locales/` to replace "Open WebUI" references with "AiSens" where appropriate.

### Additional Files to Check
- HTML templates in `.svelte-kit/output/`
- Any hardcoded strings in components
- License and copyright notices (ensure compliance with Open WebUI license)

## Step 3: Build Custom Docker Image

### Standard Build (CPU Only)
```bash
# Build the image
docker build -t aisens .

# Verify build
docker images | grep aisens
```

### GPU-Enabled Build (CUDA Support)
If you need GPU acceleration for WebUI's built-in inference features:

```bash
# Build with CUDA support
docker build \
  --build-arg USE_CUDA=true \
  --build-arg USE_CUDA_VER=cu128 \
  -t aisens:gpu .

# For the GPU build, run with:
docker run -d \
  --name aisens \
  --gpus all \
  -p 3000:8080 \
  -v aisens:/app/backend/data \
  -e OLLAMA_BASE_URL=https://your-ollama-server-url \
  -e WEBUI_SECRET_KEY=your-generated-secret-key \
  --restart unless-stopped \
  aisens:gpu
```

**Troubleshooting GPU Build Issues:**

**CRITICAL: Ensure sufficient disk space before building GPU images**
- GPU builds require significant disk space (4-6GB+) due to CUDA libraries and dependencies
- **Primary space consumers are Docker build cache and intermediate images**
- Ensure at least 20GB free space in Docker root directory
- Check Docker disk usage: `docker system df`
- Clear build cache: `docker builder prune -f`
- Clear unused images/containers: `docker system prune -a`
- Remove unused images: `docker image prune -a`
- Monitor space during build: `docker system df` (watch for build cache growth)

**Check for unused Docker images:**
```bash
# List all images with sizes
docker images

# List dangling images (untagged)
docker images -f dangling=true

# Remove unused images
docker image prune -a

# Remove specific images
docker rmi <image_id>
```

If the GPU build fails:

1. **Use CPU build as immediate solution:**
   ```bash
   docker build -t aisens:cpu .
   docker run -d --name aisens -p 3000:8080 \
     -e OLLAMA_BASE_URL=https://your-ollama-server-url \
     -e WEBUI_SECRET_KEY=your-generated-secret-key \
     aisens:cpu
   ```

2. **Fix GPU build disk space issues:**
   ```bash
   # Check Docker disk usage (most accurate for Docker builds)
   docker system df

   # Clear build cache (primary space consumer during builds)
   docker builder prune -f

   # Clear unused images/containers/volumes
   docker system prune -a

   # Remove unused images
   docker image prune -a

   # List and manually remove old/unused images if needed
   docker images
   docker rmi <unused_image_id>

   # Monitor filesystem space
   df -h

   # Rebuild GPU image
   docker build --no-cache --build-arg USE_CUDA=true --build-arg USE_CUDA_VER=cu128 -t aisens:gpu .
   ```

3. **Alternative CUDA versions (if cu128 fails):**
   ```bash
   # Try cu121 (smaller packages)
   docker build --build-arg USE_CUDA=true --build-arg USE_CUDA_VER=cu121 -t aisens:gpu .
   ```

**Note**: GPU support in WebUI is for its own model operations (like embeddings, reranking), not for Ollama. Ollama handles its own GPU acceleration when running separately.

### Ollama GPU Configuration
Since Ollama is deployed separately, ensure your Ollama instance has GPU access:

```bash
# If running Ollama directly on host
# Make sure NVIDIA drivers and CUDA are installed
# Ollama will automatically detect and use GPU

# If running Ollama in Docker
docker run -d --gpus=all -v ollama:/root/.ollama -p 11434:11434 --name ollama ollama/ollama

# Verify GPU usage
docker logs ollama | grep -i gpu
```

## Step 4: Deployment Options

### Option A: Docker Compose (Recommended)

Create `docker-compose.yaml`:

```yaml
services:
  aisens:
    # Use aisens:latest for CPU or aisens:gpu for GPU acceleration
    image: aisens:latest
    container_name: aisens
    volumes:
      - aisens:/app/backend/data
    ports:
      - "3000:8080"
    environment:
      - OLLAMA_BASE_URL=https://your-ollama-server-url
      - WEBUI_SECRET_KEY=your-generated-secret-key
    # Uncomment for GPU support (requires nvidia-docker runtime)
    # runtime: nvidia
    # environment:
    #   - NVIDIA_VISIBLE_DEVICES=all
    extra_hosts:
      - host.docker.internal:host-gateway
    restart: unless-stopped

volumes:
  aisens: {}
```

**For GPU deployment with Docker Compose:**
```yaml
services:
  aisens:
    image: aisens:gpu
    container_name: aisens_gpu
    volumes:
      - aisens:/app/backend/data
    ports:
      - "3000:8080"
    environment:
      - OLLAMA_BASE_URL=https://your-ollama-server-url
      - WEBUI_SECRET_KEY=your-generated-secret-key
    runtime: nvidia
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
    extra_hosts:
      - host.docker.internal:host-gateway
    restart: unless-stopped

volumes:
  aisens: {}
```

Deploy with:
```bash
# For CPU deployment
docker-compose up -d

# For GPU deployment (ensure GPU image is built first)
# Edit docker-compose.yaml to use aisens:gpu and uncomment GPU settings
docker-compose up -d
```

### Option B: Direct Docker Run

```bash
docker run -d \
  --name aisens \
  -p 3000:8080 \
  -v aisens:/app/backend/data \
  -e OLLAMA_BASE_URL=https://your-ollama-server-url \
  -e WEBUI_SECRET_KEY=your-generated-secret-key \
  --restart unless-stopped \
  aisens:latest
```

## Step 5: Configuration

### Environment Variables

| Variable | Description | Example |
|----------|-------------|---------|
| `OLLAMA_BASE_URL` | URL of your separate Ollama instance | `https://ollama.your-domain.com` |
| `WEBUI_SECRET_KEY` | Secret key for session management | `openssl rand -hex 32` |
| `PORT` | Internal port (default: 8080) | `8080` |

### Generate Secret Key
```bash
openssl rand -hex 32
# Copy the output for WEBUI_SECRET_KEY
```

## Step 6: Verification

### Check Container Status
```bash
docker ps | grep aisens
docker logs aisens
```

### Access Application
- Open browser to `http://your-gcp-vm-ip:3000`
- Verify AiSens branding appears
- Test connection to Ollama

### Health Check
```bash
curl http://localhost:3000/health
# Should return: {"status":true}
```

## Step 7: Maintenance

### Update Deployment
```bash
# Pull latest changes
git pull origin main

# Reapply rebranding changes
# (edit files as needed)

# Rebuild image
docker build -t aisens .

# Restart container
docker-compose down
docker-compose up -d
```

### Backup Data
```bash
# Backup volume data
docker run --rm -v aisens:/data -v $(pwd):/backup alpine tar czf /backup/aisens-backup.tar.gz -C /data .
```

### Logs and Monitoring
```bash
# View logs
docker-compose logs -f aisens

# Monitor resource usage
docker stats aisens
```

## Troubleshooting

### Common Issues

**Connection to Ollama fails:**
- Verify `OLLAMA_BASE_URL` is correct and accessible
- Check firewall rules on GCP VM
- Ensure Ollama is running and accepting connections

**Container won't start:**
```bash
docker logs aisens
# Check for error messages
```

**Port conflicts:**
```bash
sudo netstat -tulpn | grep :3000
# Change port mapping if needed
```

**Data persistence issues:**
- Ensure volume is properly mounted
- Check disk space: `df -h`

### Performance Optimization

**For GCP VM:**
- Use appropriate machine type based on expected load
- Configure persistent disks for data volumes
- Set up monitoring and alerts

**Docker optimizations:**
```yaml
services:
  aisens:
    # ... other config
    deploy:
      resources:
        limits:
          memory: 2G
        reservations:
          memory: 1G
```

## Security Considerations

- Use HTTPS in production (set up reverse proxy like Nginx)
- Regularly update Docker images
- Implement proper firewall rules
- Use strong `WEBUI_SECRET_KEY`
- Monitor access logs

## NGINX Reverse Proxy Configuration (HTTPS)

When deploying AiSens behind NGINX with HTTPS, **you must configure NGINX to support SSE streaming** for chat functionality to work properly.

### Common Issue: Chat Not Working Behind NGINX

If you see errors like:
```
Unexpected token 'd', "data: {"id"... is not valid JSON
```

This means NGINX is buffering the streaming responses. See [`NGINX_SSE_STREAMING_CONFIG.md`](NGINX_SSE_STREAMING_CONFIG.md) for the complete solution.

### Quick Fix - Critical NGINX Settings

Add these settings to your NGINX location block:

```nginx
location / {
    proxy_pass http://aisens:8080;
    
    # CRITICAL: Disable buffering for SSE streaming
    proxy_buffering off;
    proxy_cache off;
    
    # HTTP/1.1 for streaming
    proxy_http_version 1.1;
    proxy_set_header Connection '';
    proxy_set_header Accept-Encoding '';
    
    # Extended timeouts for AI responses
    proxy_read_timeout 600s;
    proxy_send_timeout 600s;
    
    # Standard proxy headers
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}

# WebSocket support for Socket.IO
location /socket.io/ {
    proxy_pass http://aisens:8080;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_buffering off;
}
```

For complete NGINX configuration including SSL setup, Docker Compose integration, and troubleshooting, refer to [`NGINX_SSE_STREAMING_CONFIG.md`](NGINX_SSE_STREAMING_CONFIG.md).

## License Compliance

Remember that Open WebUI uses a custom license. When rebranding to AiSens:
- For non-commercial use: No restrictions
- For commercial deployments: May need to maintain some Open WebUI branding (consult license terms)

## Support

For issues specific to:
- **AiSens customizations**: Check your forked repository
- **Open WebUI core functionality**: Refer to official documentation
- **Docker/GCP issues**: Check respective documentation

## Version History

- **v1.2**: Updated disk space troubleshooting - identified Docker build cache as primary space consumer, added `docker system df` and `docker builder prune` commands
- **v1.1**: Added critical disk space requirements for GPU builds (minimum 20GB free space)
- **v1.0**: Initial AiSens deployment guide
- Based on Open WebUI v0.6.26