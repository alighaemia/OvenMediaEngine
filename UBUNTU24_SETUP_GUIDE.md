# OvenMediaEngine Setup Guide for Ubuntu 24.04

This guide will help you clone, build, and run OvenMediaEngine on Ubuntu 24.04 server.

## Table of Contents
- [Prerequisites](#prerequisites)
- [Method 1: Using Docker (Recommended)](#method-1-using-docker-recommended)
- [Method 2: Building from Source](#method-2-building-from-source)
- [Method 3: Using Docker Compose](#method-3-using-docker-compose)
- [Testing Your Setup](#testing-your-setup)
- [Troubleshooting](#troubleshooting)

---

## Prerequisites

### For Docker-based setup:
```bash
# Update package list
sudo apt-get update

# Install Docker
sudo apt-get install -y docker.io docker-compose

# Add your user to docker group (to run docker without sudo)
sudo usermod -aG docker $USER

# Apply the group change (or logout and login again)
newgrp docker

# Verify Docker installation
docker --version
```

### For Building from Source:
```bash
# Update package list
sudo apt-get update

# Basic dependencies will be installed by prerequisites.sh script
```

---

## Method 1: Using Docker (Recommended)

This is the easiest and fastest way to get OvenMediaEngine running.

### Quick Start with Docker

```bash
# Replace 'Your.HOST.IP.Address' with your server's IP address
docker run --name ome -d -e OME_HOST_IP=Your.HOST.IP.Address \
  -p 1935:1935 \
  -p 9999:9999/udp \
  -p 9000:9000 \
  -p 3333:3333 \
  -p 3478:3478 \
  -p 10000-10009:10000-10009/udp \
  airensoft/ovenmediaengine:latest
```

### Docker with Custom Configuration

If you need to modify configuration files or add SSL certificates:

```bash
# 1. Create directories for configuration
export OME_DOCKER_HOME=/opt/ovenmediaengine
sudo mkdir -p $OME_DOCKER_HOME/conf
sudo mkdir -p $OME_DOCKER_HOME/logs

# Set permissions
sudo chgrp -R docker $OME_DOCKER_HOME 2>/dev/null || true
sudo chmod -R 775 $OME_DOCKER_HOME

# 2. Copy default configuration from container
docker run -d --name tmp-ome airensoft/ovenmediaengine:latest
docker cp tmp-ome:/opt/ovenmediaengine/bin/origin_conf/Server.xml $OME_DOCKER_HOME/conf
docker cp tmp-ome:/opt/ovenmediaengine/bin/origin_conf/Logger.xml $OME_DOCKER_HOME/conf
docker rm -f tmp-ome

# 3. Edit configuration if needed
nano $OME_DOCKER_HOME/conf/Server.xml

# 4. Run with custom configuration
docker run -d -it --name ome -e OME_HOST_IP=Your.HOST.IP.Address \
  -v $OME_DOCKER_HOME/conf:/opt/ovenmediaengine/bin/origin_conf \
  -v $OME_DOCKER_HOME/logs:/var/log/ovenmediaengine \
  -p 1935:1935 \
  -p 9999:9999/udp \
  -p 9000:9000 \
  -p 3333:3333 \
  -p 3478:3478 \
  -p 10000-10009:10000-10009/udp \
  airensoft/ovenmediaengine:latest

# 5. Check logs
tail -f $OME_DOCKER_HOME/logs/ovenmediaengine.log
```

### Docker Management Commands

```bash
# View logs
docker logs -f ome

# Stop the container
docker stop ome

# Start the container
docker start ome

# Restart the container
docker restart ome

# Remove the container
docker rm -f ome

# Check container status
docker ps -a | grep ome
```

---

## Method 2: Building from Source

Build OvenMediaEngine directly on your Ubuntu 24.04 server.

### Step 1: Clone the Repository

```bash
# Clone the repository
cd ~
git clone https://github.com/AirenSoft/OvenMediaEngine.git
cd OvenMediaEngine

# Or download and extract the master branch
curl -LOJ https://github.com/AirenSoft/OvenMediaEngine/archive/master.tar.gz
tar xvfz OvenMediaEngine-master.tar.gz
cd OvenMediaEngine-master
```

### Step 2: Install Dependencies

```bash
# Run the prerequisites script
# This will install all required dependencies
sudo ./misc/prerequisites.sh
```

**Note:** If the script fails, try running `sudo apt-get update` first and then rerun it.

### Step 3: Build OvenMediaEngine

```bash
# Navigate to source directory
cd src

# Build in release mode (optimized for production)
make release

# This will take several minutes depending on your CPU
# The compiled binary will be in: src/bin/RELEASE/OvenMediaEngine
```

### Step 4: Install as System Service

```bash
# Install OvenMediaEngine as a system service
sudo make install

# Start the service
sudo systemctl start ovenmediaengine

# Check status
sudo systemctl status ovenmediaengine

# Enable auto-start on boot
sudo systemctl enable ovenmediaengine.service

# View logs
sudo journalctl -u ovenmediaengine -f
```

### Alternative: Run Without Installing

```bash
# After building, you can run directly
cd src/bin/RELEASE
./OvenMediaEngine -c /path/to/your/config/directory
```

---

## Method 3: Using Docker Compose

For more complex setups with Origin-Edge clustering:

### Step 1: Clone Repository

```bash
git clone https://github.com/AirenSoft/OvenMediaEngine.git
cd OvenMediaEngine
```

### Step 2: Modify docker-compose.yml

```bash
# Edit the docker-compose.yml file
nano docker-compose.yml

# Update the OME_HOST_IP to your server's IP address
# Look for lines like: - OME_HOST_IP=192.168.0.160
```

### Step 3: Build and Run

```bash
# Build the Docker image locally
docker-compose build

# Start services (origin and edge)
docker-compose up -d

# View logs
docker-compose logs -f

# Stop services
docker-compose down
```

### Build from Local Source with Docker

If you want to build a Docker image from your local code changes:

```bash
# Use Dockerfile.local
docker build -f Dockerfile.local -t ovenmediaengine:local .

# Run your custom-built image
docker run --name ome -d -e OME_HOST_IP=Your.HOST.IP.Address \
  -p 1935:1935 -p 9999:9999/udp -p 9000:9000 -p 3333:3333 \
  -p 3478:3478 -p 10000-10009:10000-10009/udp \
  ovenmediaengine:local
```

---

## Testing Your Setup

### Test RTMP Ingest

You can use FFmpeg or OBS to push a stream:

```bash
# Using FFmpeg (if installed)
ffmpeg -re -i input.mp4 -c:v libx264 -preset veryfast \
  -b:v 1000k -maxrate 1000k -bufsize 2000k -c:a aac -b:a 128k \
  -f flv rtmp://Your.HOST.IP.Address/app/stream

# In OBS:
# - Server: rtmp://Your.HOST.IP.Address/app
# - Stream Key: stream
```

### Test WebRTC

Use the OvenPlayer demo page:
```
# Without TLS
http://demo.ovenplayer.com

# With TLS (requires certificate)
https://demo.ovenplayer.com
```

### Test LLHLS

Open in a browser:
```
http://Your.HOST.IP.Address:3333/app/stream/llhls.m3u8
```

### WebRTC Encoder (for testing ingest)
```
https://demo.ovenplayer.com/demo_input.html
```

---

## Firewall Configuration

If you're using Ubuntu's UFW firewall:

```bash
# Enable UFW if not already enabled
sudo ufw enable

# Allow SSH (important!)
sudo ufw allow 22/tcp

# Allow OvenMediaEngine ports
sudo ufw allow 1935/tcp    # RTMP
sudo ufw allow 9999/udp    # SRT
sudo ufw allow 4000/udp    # MPEG-2 TS
sudo ufw allow 9000/tcp    # Origin (OVT)
sudo ufw allow 3333/tcp    # LLHLS/WebRTC Signaling
sudo ufw allow 3334/tcp    # TLS LLHLS/WebRTC Signaling
sudo ufw allow 3478/tcp    # WebRTC TURN
sudo ufw allow 10000:10009/udp  # WebRTC ICE candidates

# Check firewall status
sudo ufw status
```

---

## Ports Reference

| Port            | Protocol | Purpose                                    |
|-----------------|----------|--------------------------------------------|
| 1935            | TCP      | RTMP Input                                 |
| 9999            | UDP      | SRT Input                                  |
| 4000            | UDP      | MPEG-2 TS Input                            |
| 9000            | TCP      | Origin Server (OVT)                        |
| 3333            | TCP      | LLHLS Streaming / WebRTC Signaling         |
| 3334            | TCP      | TLS LLHLS Streaming / WebRTC Signaling     |
| 3478            | TCP      | WebRTC TCP Relay (TURN Server)             |
| 10000-10009     | UDP      | WebRTC ICE Candidates                      |

---

## Troubleshooting

### Docker Issues

**Problem:** Permission denied when running docker commands
```bash
# Solution: Add user to docker group
sudo usermod -aG docker $USER
newgrp docker
# Or logout and login again
```

**Problem:** Port already in use
```bash
# Check what's using the port
sudo netstat -tulpn | grep :1935
# Or
sudo lsof -i :1935

# Stop the conflicting service or change OME port in configuration
```

**Problem:** Cannot access streams
```bash
# Check if container is running
docker ps -a | grep ome

# Check container logs
docker logs ome

# Verify firewall allows the ports
sudo ufw status
```

### Build from Source Issues

**Problem:** Prerequisites script fails
```bash
# Update package list first
sudo apt-get update

# Try running the script again
sudo ./misc/prerequisites.sh

# If still failing, check the error message and install missing packages manually
```

**Problem:** Make fails with errors
```bash
# Clean previous build
cd src
make clean

# Try building again
make release

# Check disk space
df -h
```

**Problem:** Service won't start
```bash
# Check service status and logs
sudo systemctl status ovenmediaengine
sudo journalctl -u ovenmediaengine -n 100 --no-pager

# Check configuration file
sudo cat /usr/share/ovenmediaengine/conf/Server.xml

# Test running manually
cd /usr/share/ovenmediaengine/bin
sudo ./OvenMediaEngine -c /usr/share/ovenmediaengine/conf
```

### WebRTC Not Working

**Problem:** WebRTC streams won't play in browser
- Modern browsers require HTTPS for WebRTC
- You need to install a valid SSL certificate
- See: [TLS Encryption Documentation](https://airensoft.gitbook.io/ovenmediaengine/configuration/tls-encryption)

### Getting Help

- **Documentation:** https://airensoft.gitbook.io/ovenmediaengine/
- **GitHub Issues:** https://github.com/AirenSoft/OvenMediaEngine/issues
- **Discord:** Join the OvenMediaEngine community
- **Docker Hub:** https://hub.docker.com/r/airensoft/ovenmediaengine

---

## Quick Reference Commands

### Docker Quick Start
```bash
# Run OvenMediaEngine (replace YOUR_IP)
docker run -d --name ome -e OME_HOST_IP=YOUR_IP \
  -p 1935:1935 -p 9999:9999/udp -p 9000:9000 -p 3333:3333 \
  -p 3478:3478 -p 10000-10009:10000-10009/udp \
  airensoft/ovenmediaengine:latest

# View logs
docker logs -f ome

# Stop/Start
docker stop ome
docker start ome
```

### Build from Source Quick Start
```bash
# Clone and build
git clone https://github.com/AirenSoft/OvenMediaEngine.git
cd OvenMediaEngine
sudo ./misc/prerequisites.sh
cd src && make release
sudo make install

# Start service
sudo systemctl start ovenmediaengine
sudo systemctl enable ovenmediaengine
```

---

## Additional Resources

- **Main Repository:** https://github.com/AirenSoft/OvenMediaEngine
- **Getting Started Guide:** https://airensoft.gitbook.io/ovenmediaengine/getting-started
- **Docker Guide:** https://airensoft.gitbook.io/ovenmediaengine/getting-started/getting-started-with-docker
- **Configuration Reference:** https://airensoft.gitbook.io/ovenmediaengine/configuration
- **REST API:** https://airensoft.gitbook.io/ovenmediaengine/rest-api
- **OvenPlayer (Client):** https://github.com/AirenSoft/OvenPlayer
- **OvenLiveKit (Encoder):** https://github.com/AirenSoft/OvenLiveKit-Web

---

## License

OvenMediaEngine is licensed under the AGPL-3.0-only license. For commercial licensing, contact contact@airensoft.com.
