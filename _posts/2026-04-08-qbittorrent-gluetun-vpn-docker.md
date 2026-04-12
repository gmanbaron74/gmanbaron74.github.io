# How to Set Up qBittorrent with Gluetun VPN on Docker: Complete Guide

If you're running a torrent client at home, you want three things: speed, privacy, and simplicity. qBittorrent is solid. A VPN keeps your ISP from seeing what you're downloading. And Docker makes it all reproducible without cluttering your system. The catch? Combining them properly takes a bit of know-how.

This guide walks you through setting up **qBittorrent with Gluetun VPN on Docker**—the right way. You'll get a working config, understand what's actually happening, verify your VPN is working, and know how to troubleshoot when things go sideways.

## Why This Setup?

qBittorrent is free, open-source, and light on resources. Gluetun is a VPN container that routes traffic through your chosen VPN provider without needing a separate VPN app. Docker isolates everything, so if something breaks, you restart a container instead of debugging your whole system.

The real win? **Kill switch built in.** If the VPN drops, qBittorrent stops connecting. Your ISP never sees unencrypted torrent traffic.

## What You'll Need

- **Hardware:** Any machine running Docker. A Raspberry Pi 5, a Beelink mini PC (£150–£250), or even an old laptop works fine. I'm running this on a Beelink ME Mini with 12GB RAM—specs listed below.
- **Software:** Docker and Docker Compose (pre-installed on most NAS platforms like TrueNAS, Unraid, ZimaOS).
- **VPN provider:** Mullvad (free, no account needed), ProtonVPN, Windscribe, or similar. Must support port forwarding if you want incoming connections.
- **Time investment:** About 30 minutes to get it running, including verification.

## Step 1: Choose Your VPN Provider and Get Credentials

Gluetun supports [dozens of VPN providers](https://github.com/qdm12/gluetun/wiki). Popular choices:

- **Mullvad:** Free, no login needed. Just set `VPN_SERVICE_PROVIDER=mullvad`.
- **ProtonVPN:** Paid, but reliable. You'll need your OpenVPN credentials (username/password, not your account password).
- **Windscribe:** Affordable, good speeds. Requires API key.

For this guide, I'll use **Mullvad** since it needs zero setup. If you choose another provider, grab your credentials now—you'll paste them into the docker-compose file later.

## Step 2: Create the docker-compose.yml File

Here's the complete working config. This sets up two containers: Gluetun (the VPN tunnel) and qBittorrent (the torrent app), linked so all qBittorrent traffic goes through Gluetun.

```yaml
version: '3.8'

services:
  gluetun:
    image: qmcgaw/gluetun:latest
    container_name: gluetun
    cap_add:
      - NET_ADMIN
    environment:
      # VPN Configuration
      VPN_SERVICE_PROVIDER: mullvad
      VPN_TYPE: openvpn
      OPENVPN_USER: ""
      OPENVPN_PASSWORD: ""
      # Server selection (optional; leave blank for random)
      SERVER_REGIONS: se
      # Kill switch: block all traffic if VPN drops
      FIREWALL: on
      FIREWALL_OUTBOUND_SUBNETS: 192.168.0.0/16
      # DNS for privacy
      DNS_ADDRESS: 1.1.1.1
      # Logging
      LOG_LEVEL: info
    ports:
      - "6881:6881/tcp"
      - "6881:6881/udp"
      - "6889:6889/tcp"
      - "6889:6889/udp"
      - "8080:8080/tcp"
    volumes:
      - /etc/localtime:/etc/localtime:ro
    networks:
      - torrent_network
    restart: unless-stopped

  qbittorrent:
    image: linuxserver/qbittorrent:latest
    container_name: qbittorrent
    depends_on:
      - gluetun
    network_mode: service:gluetun
    environment:
      PUID: 1000
      PGID: 1000
      TZ: Europe/London
      WEBUI_PORT: 8080
      TORRENTING_PORT: 6881
    volumes:
      - /mnt/data/qbittorrent/config:/config
      - /mnt/data/downloads:/downloads
      - /etc/localtime:/etc/localtime:ro
    restart: unless-stopped

networks:
  torrent_network:
    driver: bridge
```

**What's happening here:**

- **gluetun:** The VPN container. `NET_ADMIN` capability lets it manage networking. `FIREWALL: on` blocks traffic if VPN drops. `SERVER_REGIONS: se` picks Swedish servers (privacy-friendly); remove this line for random selection.
- **qbittorrent:** Runs inside the Gluetun network via `network_mode: service:gluetun`. All its traffic exits through the VPN.
- **Volumes:** `/mnt/data/qbittorrent/config` stores settings, `/mnt/data/downloads` is where torrents land. Adjust paths to match your system.
- **PUID/PGID:** Set to your user ID. Run `id` to find it; default 1000 works on most systems.

## Step 3: Adjust for Your VPN Provider (If Not Using Mullvad)

If you're using ProtonVPN or another provider:

```yaml
environment:
  VPN_SERVICE_PROVIDER: protonvpn  # or windscribe, expressvpn, etc.
  VPN_TYPE: openvpn
  OPENVPN_USER: your_username_here
  OPENVPN_PASSWORD: your_password_here
```

Check the [Gluetun documentation](https://github.com/qdm12/gluetun/wiki) for your provider's exact setup.

## Step 4: Deploy the Stack

Save the config as `docker-compose.yml` in a directory like `/docker/qbittorrent/`, then:

```bash
cd /docker/qbittorrent
docker compose up -d
```

Watch the logs to spot issues:

```bash
docker compose logs -f gluetun
```

You should see something like:

```
gluetun    | 2026-04-08 03:10:15 INFO: Connecting to VPN server...
gluetun    | 2026-04-08 03:10:22 INFO: VPN is running
```

Once Gluetun is up, qBittorrent starts automatically (it depends on Gluetun).

## Step 5: Access qBittorrent and Configure It

Open your browser and go to:

```
http://your-server-ip:8080
```

Default login: `admin` / `adminPassword`

**Change this immediately.** In the web UI:

1. Go to **Settings** → **Web UI**
2. Change the password
3. Note the port (default 8080)

Configure download paths:

1. **Settings** → **Downloads**
2. Set **Save files to location:** `/downloads/` (or your mounted path)
3. Leave everything else default

## Step 6: Verify the VPN Is Actually Working

This is critical. You want to be 100% sure your torrent traffic is encrypted.

### Method 1: Check Your External IP Inside the Container

Run this command:

```bash
docker exec gluetun curl -s https://api.ipify.org?format=json
```

You should see an IP that's NOT your real ISP IP. Run it a few times; Gluetun sometimes rotates servers, so the IP might change.

Compare it to your real IP (what your ISP sees):

```bash
curl -s https://api.ipify.org?format=json
```

If they're different, you're encrypted. ✓

### Method 2: Check DNS Leaks

Visit [dnsleaktest.com](https://dnsleaktest.com) from qBittorrent's perspective using a test torrent:

1. Download a torrent file that's actively seeded (e.g., Ubuntu ISO)
2. Start the torrent, then immediately run this:

```bash
docker exec qbittorrent curl -s https://api.ipify.org
```

The IP should match what dnsleaktest reports. If DNS is leaking to your ISP, you'll see different IPs—fix it by changing `DNS_ADDRESS` in the docker-compose file.

### Method 3: Real-World Test with qBittorrent

Use an open-port checker. qBittorrent exposes port 6881 (or whatever you set). Gluetun routes it through the VPN. Test it:

```bash
docker exec gluetun curl -s http://localhost:6881/test
```

This won't return a response (port 6881 is for torrent peers, not HTTP), but if it times out or fails, the port is blocked. If you picked a VPN server that supports port forwarding, you can verify it works by checking with a tool like [canyouseeme.org](https://canyouseeme.org) from outside your network.

## Understanding Kill Switch (Why It Matters)

Here's the critical bit: if your VPN connection drops for even a second, what happens?

**Without kill switch:** qBittorrent connects directly to peers over your ISP. Your ISP sees torrent traffic. Bad.

**With kill switch:** All outbound traffic stops. qBittorrent can't connect. You notice. You restart. Clean.

In this config, `FIREWALL: on` and `FIREWALL_OUTBOUND_SUBNETS: 192.168.0.0/16` create the kill switch:

- The firewall blocks any outbound traffic that didn't go through the VPN tunnel
- `192.168.0.0/16` is a safe subnet (your local network) so Docker healthchecks and local admin access still work

If you see qBittorrent stall (no downloads, no uploads), first check if the VPN is still connected:

```bash
docker compose logs gluetun | tail -20
```

Look for "VPN is running" or connection errors.

## Troubleshooting Common Issues

### Issue 1: qBittorrent Container Won't Start

**Symptom:** `docker compose logs qbittorrent` shows connection errors.

**Fix:** qBittorrent depends on Gluetun. If Gluetun takes more than 30 seconds to connect, qBittorrent gives up.

```bash
docker compose logs gluetun
```

Check if Gluetun is running. If it's stuck, it might be picking a dead server. Force a restart:

```bash
docker compose restart gluetun
```

### Issue 2: No Internet, or Downloads Stall

**Symptom:** Peers connecting but speeds are 0 KB/s, or no peers at all.

**Causes:**
1. Firewall blocking ports. Check:
   ```bash
   docker exec gluetun netstat -tulpn | grep 6881
   ```
2. VPN server doesn't support incoming connections. Try a different server by removing `SERVER_REGIONS` or picking a different region.
3. Kill switch triggered. Check logs:
   ```bash
   docker compose logs gluetun | grep -i firewall
   ```

**Fix:** Restart Gluetun and wait 10 seconds before checking again:

```bash
docker compose restart gluetun && sleep 10 && docker compose logs gluetun
```

### Issue 3: VPN Drops Randomly

**Symptom:** VPN connected, then "disconnected" messages appear in logs.

**Causes:**
1. Unstable network connection (WiFi dropout, ISP blip).
2. VPN server overloaded or crashing.
3. OpenVPN timeout settings too aggressive.

**Fix:** Add reconnection parameters to the Gluetun environment:

```yaml
environment:
  OPENVPN_CUSTOM_CONFIG: |
    connect-retry 5 300
    connect-retry-max 10
    ping 10
    ping-restart 60
```

This tells OpenVPN to retry connection more aggressively.

### Issue 4: Web UI Won't Load

**Symptom:** Browser times out at `http://your-ip:8080`.

**Fix:** Check if the port is exposed:

```bash
docker compose ps
```

Confirm port `8080` is mapped. If you changed the port in `docker-compose.yml`, use the new port. Also check firewall rules on the host—some systems block incoming ports by default.

### Issue 5: Can't Access Container from Local Network

**Symptom:** Other devices on the network can't reach qBittorrent.

**Root cause:** qBittorrent is in the Gluetun network, which is isolated.

**Workaround (quick):** Access it from the host machine only (`localhost:8080`). 

**Proper fix:** You'd need to add a secondary network to qBittorrent for local access—more complex. For most home setups, accessing from the server itself is fine.

## Hardware: What You Actually Need

I'm running this on a **Beelink ME Mini** (Intel N150, 12GB LPDDR5, 6x M.2 NVMe slots). Costs around £369. For qBittorrent + Gluetun, specs are modest:

- **CPU:** Any modern multi-core processor. Even a Raspberry Pi 4 works, though speeds max out around 30–50 MB/s.
- **RAM:** 2GB minimum; 4GB is comfortable.
- **Storage:** 20GB for the OS and Docker, then however much you want for downloads.
- **Network:** Gigabit Ethernet is ideal. WiFi works but adds latency and instability.

Budget options:
- **Raspberry Pi 5 (8GB):** ~£70, slow but silent
- **Used mini PC (eBay):** ~£100–£200, good balance
- **Beelink ME Mini:** ~£250, solid all-rounder

## Maintenance and Monitoring

Once this is running, check in occasionally:

```bash
docker compose logs --tail=50 gluetun
docker compose logs --tail=50 qbittorrent
```

Look for recurring errors. Update containers monthly:

```bash
docker compose pull
docker compose up -d
```

If you want to monitor resources (CPU, RAM, network), add a metrics tool like Netdata:

```bash
docker run --name=netdata --detach \
  --publish=19999:19999 \
  --volume=/etc/localtime:/etc/localtime:ro \
  netdata/netdata:stable
```

Then visit `http://your-ip:19999` for real-time stats.

## Amazon UK Hardware Links

If you're shopping for a mini PC or network gear:

- [Beelink ME Mini (Intel N150, 12GB, 6x M.2)](https://www.amazon.co.uk/dp/B0GKDB1RTY/?tag=baronvonhag0c-21)
- [Raspberry Pi 5 (8GB)](https://www.amazon.co.uk/dp/B0CK2FCG1K/?tag=baronvonhag0c-21)
- [Cat6A Ethernet Cable (10m)](https://www.amazon.co.uk/dp/B07KVGPVXG/?tag=baronvonhag0c-21)
- [USB Gigabit Ethernet Adapter](https://www.amazon.co.uk/dp/B00MYT481C/?tag=baronvonhag0c-21)

## Final Thoughts

qBittorrent + Gluetun + Docker is a rock-solid combination. Once you've got it running, it just works—silent, reliable, private. The kill switch gives you peace of mind. The logs tell you when something breaks.

Happy downloading.

---


