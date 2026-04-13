# How to Set Up Plex on Your Home Server: The Definitive Guide to Self-Hosted Media Streaming

Your home server is humming away quietly in the corner of your utility room. It's running Docker containers, managing backups, and perhaps serving up a few essential services like Pi-hole or Vaultwarden. But if you’re still relying on Netflix for your main movie night viewing, paying monthly subscription fees just to stream content across your local network—and occasionally out to the wilds of the internet—you're missing out on the true power of self-hosting. Plex isn't just another media server; it is the central nervous system of a modern home entertainment setup. It organises, transcribes, streams, and manages every piece of digital media you own, all under your complete control.

The problem with relying solely on commercial services is that you are renting access to your library. If Netflix decides to drop support for an obscure 1980s sci-fi flick you love, or if Disney+ changes its DRM handshake, suddenly *your* collection feels less permanent. Furthermore, paying £14.99 a month just to stream content from your own NAS is poor value when that same server could be running dozens of other services for free.

This guide will walk you through setting up Plex on a home server—specifically targeting the Beelink ME Mini running ZimaOS and Docker, as per my current setup—so you can enjoy flawless, high-quality media streaming from your living room TV to your tablet in the garden, all while keeping the data firmly within your own walls.

## Why Go Self-Hosted with Plex? (The Value Proposition)
Before diving into commands, it’s worth understanding *why* this is better than just using a cloud service.

**Control and Ownership:** Your media files live on your NAS or server storage. If you sell the house, the media goes with you. No more worrying about account suspensions or regional licensing changes affecting your favourite shows.
**Cost Efficiency:** Once the initial hardware investment is made (like my Beelink ME Mini), the operational cost of Plex is virtually zero—just electricity. This beats any monthly subscription fee over a few years.
**Flexibility:** You can easily add other services alongside it. Want to run Jellyfin too? Go for it. Need an external VPN connection via Gluetun so you can stream while on holiday? Plex integrates seamlessly with that.
**Transcoding Power:** This is where the hardware shines. If your server has a decent CPU (like the N150 in my ME Mini), Plex will automatically convert media formats *on the fly* to suit whatever device you are watching on—whether it’s an ancient Roku or a brand-new 4K Apple TV.

## Prerequisites: What You Need Before Starting
Before we touch Docker, ensure your server environment is ready. Based on my setup, these are the essentials:

1.  **A Server:** Ideally something low-power and reliable. The **Beelink ME Mini (ASIN: B0GKDB1RTY)** is perfect for this job; it’s quiet, efficient, and powerful enough to handle transcoding a few simultaneous streams without breaking a sweat.
2.  **Storage:** A dedicated location on your server/NAS where your media resides. Ensure the path is accessible by Docker containers (e.g., `/mnt/data/media`).
3.  **Docker & Docker Compose:** These are non-negotiable for modern, clean deployments. If you're running ZimaOS, these should already be installed, but it’s worth a quick check.

## Step 1: Preparing the Media Library Structure
Plex is fussy about organisation; treat it like a librarian. A consistent structure ensures Plex can scan and correctly identify everything.

**The Golden Rule:** Keep media in top-level folders named after their type (Movies, TV Shows, Music). Do not nest them too deeply initially.

For example, instead of `/mnt/data/media/TV/Show Name/Season 01/Episode 1`, aim for:
`/mnt/data/media/TV Shows/The Mandalorian/Season 01/` (with `The Mandalorian - S01E01 - Episode Title.mkv`)

**Action:** Navigate to your media directory and ensure you have at least one complete movie folder and one TV show season folder ready for testing.

## Step 2: The Docker Compose Blueprint
We will use `docker-compose` because it allows us to define all the necessary services (Plex itself, potentially a reverse proxy if needed) in one clean YAML file.

Create a new directory for your Plex setup and navigate into it:
```bash
mkdir ~/plex-server
cd ~/plex-server
```

Now, create the `docker-compose.yml` file using a text editor (like Nano):
```bash
nano docker-compose.yml
```

Paste the following configuration into the file. This is a robust starting point:

```yaml
version: "3.8"

services:
  plex:
    image: lscr.io/linuxserver/plex:latest
    container_name: plex
    network_mode: host # Using host mode simplifies networking immensely on ZimaOS
    environment:
      - PUID=1000         # Change this to your user ID (usually 1000)
      - PGID=1000         # Change this to your group ID (usually 1000)
      - TZ=Europe/London  # Set this to your timezone!
    volumes:
      - /path/to/your/config:/config # <-- CRITICAL: Path where Plex stores its database/metadata
      - /mnt/data/media/TV Shows:/tv # <-- CRITICAL: Path to your TV Show library
      - /mnt/data/media/Movies:/movies # <-- CRITICAL: Path to your Movie library
      # - /mnt/data/music:/music # Uncomment if you have music ready
    ports:
      # Plex runs on port 32400 by default. Host mode handles this, but listing it is good practice.
      - "32400:32400/tcp"
      - "3005:3005/tcp" # For discovery/web UI access
    restart: unless-stopped
```

**Crucial Edits Before Proceeding:**
1.  Change `PUID` and `PGID` to match your server's user IDs (run `id -u` and `id -g` on the host).
2.  Change `/path/to/your/config` to a specific directory, e.g., `/home/node/plex-server/config`.
3.  Verify `/mnt/data/media/TV Shows` and `/mnt/data/media/Movies` point *exactly* to where your media lives on the host machine.

## Step 3: Deployment and Verification
With the file saved, deploy the stack using Docker Compose:

```bash
docker compose up -d
```

This command pulls the latest Plex image, creates the container named `plex`, maps all the necessary volumes (config, TV shows, movies), and runs it in detached mode (`-d`).

**Verification:** Check that the container is running correctly.
```bash
docker ps | grep plex
```
You should see a line showing the `plex` container with status "Up".

## Step 4: Initial Library Scan via Web UI
Now, you need to tell Plex where your media *is*. Open a web browser and navigate to `http://[YourServerIP]:3005/web`. You will be prompted to sign in or create an account. Sign in with your Plex credentials (or create one).

Once logged in, the setup wizard should guide you:
1.  **Claim Your Server:** If it's new, claim it.
2.  **Add Libraries:** Click "Add Library." Select **"Movies"**, and then browse to the folder that maps to `/mnt/data/media/Movies` on your server. Repeat this process for TV Shows (selecting the correct metadata type).

Plex will immediately begin scanning those directories, downloading posters, synopses, cast information, and artwork from The Movie Database (TMDb) or TheTVDB. This initial scan can take a while depending on how much media you have!

## Step 5: Fine-Tuning and Advanced Tips
Once the initial scan is complete, you are ready to stream. But here are a few tips to make it truly shine:

**Transcoding Test:** Start playing a high-bitrate movie (e.g., a 4K HDR file) on your phone while connected to the same network as the server. Go into the Plex Dashboard and check the "Activity" tab. If you see "Transcoding," it means Plex is actively converting the stream for your mobile device—success!

**Remote Access:** To watch outside your home, ensure port 32400 is forwarded on your router to the IP address of your Beelink ME Mini. Alternatively, if using a dynamic DNS service (like DuckDNS), you can point `plex.yourdomain.com` directly to that port.

**Music Integration:** If you added the music library in Step 2, go into Plex settings and ensure the Music Library is set to scan the correct path (`/mnt/data/music`). You might also want to enable "Automatic Genre Detection" for better organisation.

## Affiliate Links: Gear That Makes This Possible
To build this setup, I relied on some excellent hardware. Here are the links so you can grab what you need and support the blog:

*   **Beelink ME Mini (The Server):** [Amazon UK Link](https://www.amazon.co.uk/dp/B0GKDB1RTY/?tag=baronvonhag0c-21) - *This is the workhorse.*
*   **Raspberry Pi 5 (For a secondary NAS/AdBlocker):** [Amazon UK Link](https://www.amazon.co.uk/dp/B0CK2FCG1K/?tag=baronvonhag0c-21) - *Great for running Pi-hole alongside Plex.*
*   **Cat6A Ethernet Cable (For reliable connection):** [Amazon UK Link](https://www.amazon.co.uk/dp/B07KVGPVXG/?tag=baronvonhag0c-21) - *Don't skimp on the cable quality.*

## Conclusion
Setting up Plex might seem like a chore, but once it’s running smoothly—serving up crisp 4K streams of your favourite content while you sit back with a pint—you realise how much better life is when you own the infrastructure. It moves media consumption from being a subscription *expense* to being a self-hosted *asset*. Get this done; your viewing experience will thank you for it.
