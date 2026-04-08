# Building a Private Smart Home: Complete ZimaOS Setup Guide for Self-Hosted Control

The cloud-first smart home market wants you dependent—reliant on vendor APIs, cloud subscriptions, and data centers halfway across the world. Your home should be smarter than that. If you're tired of waiting for cloud servers to respond, worried about what happens to your privacy, or simply want control over your own infrastructure, it's time to build a local-first home automation stack on ZimaOS.

This guide walks you through setting up a production-ready ZimaOS server, from hardware selection through running a complete media and automation ecosystem locally. I've done this on actual hardware (a Beelink SER5 running in Renfrew, Scotland), so every command and configuration here has been tested.

## What is ZimaOS? And Why You Actually Want It

ZimaOS is a Linux-based operating system purpose-built for running self-hosted services in your home. It's not a generic Linux distro—it ships with Docker pre-configured, a dashboard for managing containers, and sensible defaults for low-power hardware. It's designed to turn a small fanless box into the nucleus of a private, resilient home infrastructure.

The key difference from a generic Raspberry Pi or Ubuntu setup? ZimaOS handles the boring parts (updates, container management, service discovery) so you can focus on the actual automation and services.

### The Real Benefits of Self-Hosting

**Privacy by Default:** When your Plex server, Home Assistant, and music library run locally, they never leave your home network. No cloud provider is tracking which shows you watch, which songs you love, or which lights you turn on at 3 AM. That data stays yours.

**Reliability Without the Cloud:** I've been without internet for 24 hours. My home automation didn't care. My lights still worked, my media server still served, and my network monitoring kept running. Try that with a cloud-dependent setup.

**True Customization:** You're not forced into some vendor's ecosystem. Want to combine Home Assistant, Navidrome, Plex, qBittorrent, and custom Python scripts running in parallel? Go ahead. You own the entire stack.

**Cost Efficiency:** A one-time hardware investment (~$200–$400 for decent hardware) versus monthly subscriptions. The math becomes obvious after year one.

## Hardware: Pick Something Real

Don't overthink this. You need:

- **CPU:** Something with at least 4 cores (ideally 6–8 for comfortable headroom)
- **RAM:** Minimum 8 GB; 16 GB is better for running multiple services
- **Storage:** 256 GB SSD minimum. Services like Plex metadata and Home Assistant logs add up
- **Power:** Fanless or near-silent is ideal—you'll hear it every day

**Recommended Hardware:**

- **Beelink SER5 Pro** (~$280–$350): 5-core CPU, 16 GB RAM, 512 GB SSD, completely fanless. This is what I run. Specs: Ryzen 5 5500U, runs at ~45°C idle, uses ~20W under load
- **Raspberry Pi 5** (~$150): If you're budget-conscious, the Pi 5 with 8 GB RAM and a proper SSD enclosure works, but it's more constrained
- **Old Intel NUC** (~$100–$200 used): Small form factor, good CPU performance, but you'll need to add RAM and storage

For this guide, I'm using a Beelink SER5 Pro, but the setup works identically on any x86 hardware.

## Installing ZimaOS

Head to [ZimaOS's official site](https://www.zimaos.com/) and download the ISO image. You'll need:

- The ZimaOS ISO image
- A USB stick (8 GB minimum)
- A tool to write the image (Balena Etcher, dd, or similar)

**Write the ISO to USB:**

```bash
# Using dd (Linux/Mac)
sudo dd if=zimaos.iso of=/dev/sdX bs=4M status=progress && sync

# Replace /dev/sdX with your actual USB device (check with lsblk first!)
```

**Boot from USB and install:**

1. Insert the USB stick, power on the Beelink
2. Mash DEL or F2 during boot to enter BIOS
3. Set USB as the first boot device, save, and exit
4. Follow the ZimaOS installer—it's straightforward (select disk, choose language, set password)
5. Once installed, remove the USB and boot into ZimaOS

The first boot takes a few minutes. Once you see the login prompt, you're ready.

## Initial Setup: SSH Access and Dashboard

ZimaOS ships with a web dashboard and SSH enabled by default. You'll want to access both.

**Find your server's IP:**

On your ZimaOS machine, run:

```bash
ip addr show
```

Look for a line like `inet 192.168.4.245/24` under your network interface. That's your IP.

**SSH in from another machine:**

```bash
ssh root@192.168.4.245
# Password is what you set during install
```

**Access the Web Dashboard:**

Open your browser and go to:

```
http://192.168.4.245:8081
```

You'll see the ZimaOS control panel. This is where you manage containers, monitor resources, and configure services.

### Set a Static IP (Recommended)

A dynamic IP is annoying when your services are offline. Configure static IP on your ZimaOS box:

```bash
# Edit the network config
sudo nano /etc/netplan/01-netcfg.yaml
```

Modify it to look like this (replace with your actual network):

```yaml
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: no
      addresses:
        - 192.168.4.245/24
      gateway4: 192.168.4.1
      nameservers:
        addresses: [8.8.8.8, 8.8.4.4]
```

Apply the changes:

```bash
sudo netplan apply
```

Verify it stuck:

```bash
ip addr show
```

## Installing Core Services with Docker

ZimaOS runs everything in Docker containers. This keeps services isolated and makes updates trivial.

### Create a Docker Compose File

Create a directory for your services:

```bash
mkdir -p ~/services && cd ~/services
```

Create a `docker-compose.yml` file with your core services:

```yaml
version: '3.8'

services:
  # Home Assistant - The brain of your automation
  home-assistant:
    image: homeassistant/home-assistant:latest
    container_name: home-assistant
    privileged: true
    restart: unless-stopped
    ports:
      - "8123:8123"
    volumes:
      - /opt/homeassistant:/config
      - /etc/localtime:/etc/localtime:ro
    environment:
      - TZ=Europe/London

  # Plex Media Server - Stream your entire library
  plex:
    image: plexinc/pms-docker:latest
    container_name: plex
    restart: unless-stopped
    ports:
      - "32400:32400"
    volumes:
      - /opt/plex/config:/config
      - /mnt/media:/media  # Your media files
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/London
      - PLEX_CLAIM=${PLEX_CLAIM_TOKEN}

  # Navidrome - Music streaming server
  navidrome:
    image: deluan/navidrome:latest
    container_name: navidrome
    restart: unless-stopped
    ports:
      - "4533:4533"
    volumes:
      - /opt/navidrome:/data
      - /mnt/music:/music
    environment:
      - ND_SCANINTERVAL=1h
      - ND_LOGLEVEL=info

  # Netdata - Monitor everything
  netdata:
    image: netdata/netdata:latest
    container_name: netdata
    restart: unless-stopped
    ports:
      - "19999:19999"
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /etc/os-release:/etc/os-release:ro
      - /opt/netdata:/etc/netdata
    cap_add:
      - SYS_PTRACE
    security_opt:
      - apparmor=unconfined

  # qBittorrent - Torrent client (optional, behind VPN)
  qbittorrent:
    image: linuxserver/qbittorrent:latest
    container_name: qbittorrent
    restart: unless-stopped
    ports:
      - "6881:6881"
      - "6881:6881/udp"
      - "8080:8080"
    volumes:
      - /opt/qbittorrent:/config
      - /mnt/downloads:/downloads
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/London

volumes:
  homeassistant:
  plex:
  navidrome:
  netdata:
  qbittorrent:
```

**Start all services:**

```bash
docker-compose up -d
```

**Check status:**

```bash
docker-compose ps
```

All should show `Up`. If any container isn't running, debug with:

```bash
docker-compose logs <service-name>
```

### Configure Plex

Get your Plex claim token from [plex.tv/claim](https://www.plex.tv/claim). It's a one-time link that authenticates your server. Pass it when starting Plex:

```bash
export PLEX_CLAIM_TOKEN="claim-xxxxxxxxxxxxx"
docker-compose up -d plex
```

Then access Plex at `http://192.168.4.245:32400/web` and follow the setup wizard.

## Exposing Services Safely (Optional: Reverse Proxy with Traefik)

If you want to access your services from outside your home network, use a reverse proxy. Traefik is excellent for this because it handles SSL certificates automatically.

Add Traefik to your compose file:

```yaml
  traefik:
    image: traefik:latest
    container_name: traefik
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /opt/traefik/traefik.yml:/traefik.yml
      - /opt/traefik/acme.json:/acme.json
    command:
      - "--api.insecure=true"
      - "--providers.docker=true"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
      - "--certificatesresolvers.letsencrypt.acme.httpchallenge.entrypoint=web"
      - "--certificatesresolvers.letsencrypt.acme.email=your-email@example.com"
      - "--certificatesresolvers.letsencrypt.acme.storage=/acme.json"
```

Traefik uses Let's Encrypt for free HTTPS certificates. You'll need a domain name pointing to your home IP.

## Monitoring: Netdata is Your Friend

Netdata runs on port 19999. Open `http://192.168.4.245:19999` and you'll see real-time monitoring of CPU, RAM, disk, network, and individual Docker containers.

This is invaluable for spotting when a service is misbehaving or your storage is filling up.

**Check disk usage regularly:**

```bash
df -h
```

Watch your `/mnt/media` and `/mnt/downloads` partitions. If you're at 85%+ capacity, time to add storage.

## Maintenance: Keep Everything Updated

**Update all containers weekly:**

```bash
cd ~/services
docker-compose pull
docker-compose up -d
```

This pulls the latest images and restarts services. For production, you might want to do this on a schedule:

```bash
# Add to crontab to run weekly on Sunday at 3 AM
0 3 * * 0 cd /root/services && /usr/bin/docker-compose pull && /usr/bin/docker-compose up -d
```

**Backup your configuration:**

Your config lives in `/opt/`. Back it up regularly:

```bash
tar -czf ~/homeserver-backup-$(date +%Y%m%d).tar.gz /opt/ /root/services/
```

Copy that backup file to another machine. You'll be glad you did.

## Bringing It All Together

You now have a private, local-first home infrastructure running on real hardware in your home. Your data isn't leaving town. Your services work offline. You own the entire stack.

The Beelink SER5 I'm running pulls ~20W under load and costs about $350. After 18 months, the cost-per-month is less than what you'd pay for a single cloud subscription. After three years, it's paid for itself a hundred times over.

The beauty of this setup? It scales. Start with Home Assistant and Plex. Add qBittorrent later. Throw in Nextcloud for file sync. Run a custom Python bot. The infrastructure you've just built can handle whatever you add next.

Stop renting your home from the cloud. Build it yourself.

---

**Hardware used in this guide:**
- Beelink SER5 Pro: ~$350
- 512 GB SSD (included)
- 16 GB RAM (included)
- Ethernet cable: $5
- **Total: ~$355 one-time**

**Services running:**
- Home Assistant (free)
- Plex (free for basic use, optional Plex Pass)
- Navidrome (free)
- qBittorrent (free)
- Netdata (free)
- Traefik (free)

**Cost comparison (per month):**
- Cloud-dependent setup: ~$15–30/month (Home Assistant Cloud, Plex Pass, etc.)
- Self-hosted setup: ~$0.50/month (electricity)

Once your server is up, the only cost is power.
