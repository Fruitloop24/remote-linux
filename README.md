# Linux Zero Trust Remote Access Setup

## Overview

Complete zero-trust remote access solution for Linux systems providing browser-based GUI, terminal, and file access with no open ports. Built using Cloudflare Tunnels, Caddy reverse proxy, and modern web technologies.

## Architecture

```
Internet → Cloudflare Edge → Cloudflare Tunnel → Caddy → Local Services
```

**Services Stack:**
- **x11vnc**: VNC server for desktop sharing (port 5900)
- **noVNC**: HTML5 VNC client (port 8080)
- **ttyd**: Web-based terminal emulator (port 7681)
- **filebrowser**: Web file manager (port 8081)
- **Caddy**: Reverse proxy and web server (port 80)
- **cloudflared**: Cloudflare tunnel client

## Prerequisites

- Ubuntu 24.04 LTS (or similar)
- Cloudflare account with domain
- Cloudflare Zero Trust account
- sudo access

## Installation Guide

### 1. Install Core Dependencies

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install VNC and X11 components
sudo apt install -y x11vnc xvfb

# Install Python for noVNC
sudo apt install -y python3 python3-pip

# Install ttyd
sudo apt install -y ttyd

# Install filebrowser
curl -fsSL https://raw.githubusercontent.com/filebrowser/get/master/get.sh | bash
sudo mv filebrowser /usr/local/bin/
```

### 2. Install Caddy

```bash
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
sudo apt update
sudo apt install caddy
```

### 3. Install Cloudflared

```bash
# Download and install cloudflared
wget https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb
sudo dpkg -i cloudflared-linux-amd64.deb
```

### 4. Download and Setup noVNC

```bash
cd ~/
git clone https://github.com/novnc/noVNC.git
cd noVNC
```

## Configuration

### 1. Configure x11vnc Service

Create VNC password:
```bash
mkdir -p ~/.vnc
x11vnc -storepasswd ~/.vnc/passwd
```

Create systemd service:
```bash
sudo nano /etc/systemd/system/x11vnc.service
```

```ini
[Unit]
Description=X11 VNC Server
After=multi-user.target

[Service]
Type=simple
ExecStart=/usr/bin/x11vnc -display :0 -rfbauth /home/mini/.vnc/passwd -rfbport 5900 -shared -forever -bg
User=mini
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

### 2. Configure noVNC Service

Create noVNC service:
```bash
sudo nano /etc/systemd/system/novnc.service
```

```ini
[Unit]
Description=noVNC Web Server
After=network.target

[Service]
Type=simple
ExecStart=/usr/bin/python3 -m http.server 8080
WorkingDirectory=/home/mini/noVNC
User=mini
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

### 3. Configure Filebrowser

Create files directory:
```bash
mkdir -p ~/files
```

Create filebrowser service:
```bash
sudo nano /etc/systemd/system/filebrowser.service
```

```ini
[Unit]
Description=File Browser
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/filebrowser -a 0.0.0.0 -p 8081 -r /home/mini/files -d /home/mini/filebrowser.db
User=mini
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

### 4. Configure Caddy

Create Caddyfile:
```bash
sudo nano /etc/caddy/Caddyfile
```

```caddy
{
    auto_https off
}

:80 {
    @filebrowser host linux-files.panacea-tech.net
    handle @filebrowser {
        reverse_proxy localhost:8081
    }
    reverse_proxy localhost:8080
}
```

### 5. Configure Cloudflare Tunnel

Authenticate with Cloudflare:
```bash
cloudflared tunnel login
```

Create tunnel:
```bash
cloudflared tunnel create linux
```

Note the tunnel ID from output. Create config file:
```bash
nano ~/.cloudflared/config.yml
```

```yaml
tunnel: [YOUR-TUNNEL-ID]
credentials-file: /home/mini/.cloudflared/[YOUR-TUNNEL-ID].json
ingress:
  - hostname: linux-remote.panacea-tech.net
    service: http://127.0.0.1:80
    originRequest:
      noTLSVerify: true
      httpHostHeader: "linux-remote.panacea-tech.net"
  - hostname: linux-term.panacea-tech.net
    service: http://127.0.0.1:7681
  - hostname: linux-files.panacea-tech.net
    service: http://127.0.0.1:80
  - service: http_status:404
```

Create DNS records:
```bash
cloudflared tunnel route dns linux linux-remote.panacea-tech.net
cloudflared tunnel route dns linux linux-term.panacea-tech.net
cloudflared tunnel route dns linux linux-files.panacea-tech.net
```

Create cloudflared service:
```bash
sudo nano /etc/systemd/system/cloudflared.service
```

```ini
[Unit]
Description=Cloudflare Tunnel - Linux
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/cloudflared tunnel --config /home/mini/.cloudflared/config.yml run
Restart=on-failure
RestartSec=10
User=mini

[Install]
WantedBy=multi-user.target
```

## Service Management

Enable and start all services:
```bash
# Enable services
sudo systemctl enable x11vnc novnc filebrowser caddy cloudflared

# Start services
sudo systemctl start x11vnc novnc filebrowser caddy cloudflared

# Check status
sudo systemctl status x11vnc novnc filebrowser caddy cloudflared
```

## Cloudflare Zero Trust Setup

1. Go to Cloudflare Zero Trust dashboard
2. Navigate to Access → Applications
3. Create applications for each hostname:

**Application 1: Linux Desktop**
- Subdomain: `linux-remote`
- Domain: `panacea-tech.net`
- Policy: Allow specific email addresses

**Application 2: Linux Terminal**
- Subdomain: `linux-term`
- Domain: `panacea-tech.net`
- Policy: Allow specific email addresses

**Application 3: Linux Files**
- Subdomain: `linux-files`
- Domain: `panacea-tech.net`
- Policy: Allow specific email addresses

## Access URLs

After setup, access via:
- **Desktop GUI**: https://linux-remote.panacea-tech.net
- **Terminal**: https://linux-term.panacea-tech.net
- **File Manager**: https://linux-files.panacea-tech.net

## Security Features

- **Zero open ports**: All traffic through Cloudflare tunnel
- **Email-based authentication**: Cloudflare Access policies
- **Encrypted connections**: HTTPS/WSS end-to-end
- **User-level isolation**: Services run as non-root user
- **Audit logging**: Cloudflare Access logs all connections

## Troubleshooting

### Check service status:
```bash
sudo systemctl status [service-name]
```

### View logs:
```bash
sudo journalctl -u [service-name] -f
```

### Test local services:
```bash
curl -I localhost:8080  # noVNC
curl -I localhost:7681  # ttyd
curl -I localhost:8081  # filebrowser
curl -I localhost:80    # Caddy
```

### Verify tunnel:
```bash
cloudflared tunnel list
cloudflared tunnel info linux
```

## File Structure

```
/home/mini/
├── .cloudflared/
│   ├── config.yml
│   ├── cert.pem
│   └── [tunnel-id].json
├── .vnc/
│   └── passwd
├── files/              # File browser root
├── filebrowser.db      # File browser database
└── noVNC/              # noVNC web client
```

## Business Model Integration

This setup supports:
- **Multi-tenant**: Each customer gets own tunnel/domain
- **Pay-per-endpoint**: Scale by adding more Linux boxes
- **Managed service**: Automated deployment via scripts
- **Enterprise security**: Cloudflare Zero Trust integration

## Automation Scripts

### Install Script
```bash
#!/bin/bash
# install-linux-remote.sh
# Add full installation automation here
```

### Service Management
```bash
#!/bin/bash
# manage-services.sh
# Add service start/stop/restart automation here
```

## Performance Considerations

- **noVNC**: Optimized for low latency desktop access
- **ttyd**: Minimal overhead terminal sessions
- **Caddy**: High-performance reverse proxy
- **Cloudflare**: Global edge network reduces latency

## Maintenance

### Regular tasks:
- Update cloudflared: `sudo apt update && sudo apt upgrade cloudflared`
- Rotate VNC passwords: `x11vnc -storepasswd ~/.vnc/passwd`
- Monitor logs: `sudo journalctl -u cloudflared --since "1 hour ago"`
- Check tunnel health: `cloudflared tunnel info linux`

### Backup:
- Config files: `~/.cloudflared/config.yml`
- VNC password: `~/.vnc/passwd`
- Caddy config: `/etc/caddy/Caddyfile`
- Service files: `/etc/systemd/system/`

## Notes

- Replace `mini` with actual username throughout setup
- Replace `panacea-tech.net` with your domain
- Update tunnel ID in config files
- Ensure firewall allows outbound HTTPS (443) for tunnel