# How to Set Up Immich: Self-Hosted Google Photos Alternative

Your home photo library deserves better than a cloud subscription that constantly creeps up on you. You're tired of paying £10 a month just to back up family snaps, and the idea of your precious memories being locked behind Google’s ever-changing privacy policies feels... well, frankly, irritating. If you've been eyeing self-hosting solutions but felt overwhelmed by the sheer complexity of setting up something like Nextcloud or PhotoPrism, Immich is the answer. It provides a near-perfect, modern replacement for Google Photos—automatic backups, beautiful UI, object recognition, and crucially, it runs beautifully in Docker on almost any home server hardware you own. This guide will walk you through getting Immich running smoothly, using my preferred stack: ZimaOS (running on a Beelink ME Mini) with Docker Compose.

## The Problem: Cloud Lock-in and Feature Creep
The core issue isn't just storage; it’s the *experience*. Google Photos is brilliant, but its ecosystem feels proprietary. You can't easily pull your library into another service without using their export tools, which are clunky. Furthermore, while they offer 'Magic Eraser,' that feature often requires a paid subscription tier. When you own your data, you control the features. Immich gives you that ownership back, allowing you to run it alongside Plex for video and Navidrome for music—a true self-hosted media ecosystem.

## Why Immich Over Others?
While there are many contenders, Immich stands out due to several key differentiators:
*   **Modern UI:** It looks like Google Photos right out of the box. No clunky interfaces that feel like they were designed in 2012.
*   **Automatic Backup:** The mobile app handles everything—uploading photos and videos automatically as you take them, even when offline.
*   **AI Features:** Object recognition is top-tier, and its search capabilities are far superior to many competitors.
*   **Docker Native:** It's built for containers, meaning deployment via Docker Compose is straightforward, reliable, and repeatable.

## Prerequisites: What You Need Before Starting
Before you dive into the YAML files, ensure your home server has these things sorted. Based on my own setup, here’s what I recommend:

**Hardware Recommendation:** A **Beelink ME Mini** (ASIN: B0GKDB1RTY) is fantastic for this job. It's small enough to tuck away but powerful enough to handle the initial indexing load of a large photo library while keeping CPU usage low. If you need more muscle, an older Intel NUC will do the trick.

**Software Stack:**
*   A running OS (ZimaOS is my preference for its simplicity).
*   Docker installed and running.
*   Docker Compose installed.
*   Sufficient storage space (SSD highly recommended for fast indexing).

## Step 1: Preparing Your Storage Volume
Immich needs a dedicated place to store all your photos, videos, and the database files it generates. You must create this directory structure on your server first. If you are using ZimaOS, navigate to `/home/node/.openclaw/workspace` or wherever you keep your main project folders.

Let's assume we want a dedicated folder called `immich-data`. We will place the actual media inside a subfolder named `upload`.

```bash
# Navigate to your desired location (e.g., in the workspace)
cd /home/node/.openclaw/workspace/

# Create the main data directory if it doesn't exist
mkdir -p immich-data

# Create the actual media upload folder inside it
mkdir -p immich-data/upload

echo "Storage structure ready at: /home/node/.openclaw/workspace/immich-data"
```

## Step 2: The Docker Compose File (The Blueprint)
This is where the magic happens. We need to define Immich itself, its PostgreSQL database, and Redis cache. I've prepared a standard `docker-compose.yml` file for you. Create this file in your `/home/node/.openclaw/workspace/` directory.

**File: docker-compose.yml**
```yaml
version: '3.8'

services:
  immich-postgres:
    image: postgres:15-alpine
    container_name: immich_postgres
    restart: always
    environment:
      POSTGRES_USER: immich_user
      POSTGRES_PASSWORD: your_strong_db_password # <-- CHANGE THIS!
      POSTGRES_DB: immich_database
    volumes:
      - ./immich-data/postgres:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  immich-redis:
    image: redis:7-alpine
    container_name: immich_redis
    restart: always
    volumes:
      - ./immich-data/redis:/data
    ports:
      - "6379:6379"

  immich-server:
    image: ghcr.io/immich-app/immich-server:latest
    container_name: immich_server
    restart: always
    depends_on:
      - immich-postgres
      - immich-redis
    environment:
      # Database Configuration
      DB_HOSTNAME: immich-postgres
      DB_PORT: 5432
      DB_USERNAME: immich_user
      DB_PASSWORD: your_strong_db_password # <-- MUST MATCH DB Password!
      DB_DATABASE: immich_database
      # Redis Configuration
      REDIS_HOSTNAME: immich-redis
      REDIS_PORT: 6379
      # Optional: Set a timezone if you aren't using the host default
      TZ: Europe/London
    volumes:
      - ./immich-data/server:/usr/src/app/upload # This is where your photos land!
    ports:
      - "2283:3001" # The main web UI port
    command: [ "--server", "--database-url", "postgres://immich_user:your_strong_db_password@immich-postgres:5432/immich_database", "--redis-host", "immich-redis", "--redis-port", "6379" ]

  immich-microservices:
    image: ghcr.io/immich-app/immich-microservices:latest
    container_name: immich_microservices
    restart: always
    depends_on:
      - immich-server
    environment:
      # Database Configuration (must match server)
      DB_HOSTNAME: immich-postgres
      DB_PORT: 5432
      DB_USERNAME: immich_user
      DB_PASSWORD: your_strong_db_password # <-- MUST MATCH DB Password!
      DB_DATABASE: immich_database
      # Redis Configuration (must match server)
      REDIS_HOSTNAME: immich-redis
      REDIS_PORT: 6379
      TZ: Europe/London
    volumes:
      - ./immich-data/microservices:/usr/src/app/upload # Shares the same upload volume!
    ports:
      # Microservices often don't need a public port, but it doesn't hurt.
      - "3002:3002"

```

**Crucial Step:** Remember to replace `your_strong_db_password` in all three places with a unique password!

## Step 3: Deployment and Initial Run
With the file saved, open your terminal in `/home/node/.openclaw/workspace/` and run:

```bash
docker compose up -d --build
```

This command tells Docker Compose to build any necessary images (though Immich usually pulls them) and start all services (`-d` runs them detached in the background). This initial pull and setup can take several minutes depending on your internet speed.

## Step 4: Verification and First Upload
Once the command finishes, you should check that everything is running smoothly:

```bash
docker compose ps
```
You want to see `immich_postgres`, `immich_redis`, `immich_server`, and `immich_microservices` all showing a status of **Up**.

Now, open your web browser (or use the `browser` tool) and navigate to: `http://localhost:2283` (or whatever IP/hostname your server is on). You should see the Immich welcome screen.

To test it out, simply copy a few photos from your desktop into the `/home/node/.openclaw/workspace/immich-data/upload` folder and refresh the web UI. They should appear almost instantly!

## Step 5: Mobile App Integration (The Final Polish)
While the web interface is great for browsing on the server itself, the real power comes from the mobile app. Download the Immich app from the Google Play Store or Apple App Store. When prompted to add a new instance, use these details:

*   **Server URL:** `http://[Your_Server_IP]:2283`
*   **API Key:** You can generate this in the web UI under Settings -> General. (If you skip generating it now, the app will prompt you to create one upon first connection).

## Final Thoughts: Beyond Photos
Immich isn't just a photo backup; it’s a central hub. Once it's running, you can easily integrate other services by adding more containers to your `docker-compose.yml` file—perhaps linking it to Plex so that when Immich detects a new video file in the same `/upload` directory, it automatically scans and indexes metadata for you.

This setup is robust, scalable, and gives Gerry complete control over his digital memories. It’s far superior to just letting Google handle everything passively. Give this a go; once you see your photos populating beautifully on the mobile app, you won't look back at those monthly subscription bills again.

***

### 📸 Immich Hardware & Link Summary
*   **Recommended Server:** Beelink ME Mini (Intel N150) - [Amazon UK Link](https://www.amazon.co.uk/dp/B0GKDB1RTY/?tag=baronvonhag0c-21)
*   **Storage Drive Recommendation:** A reliable 4TB NAS drive is a good starting point for media storage (e.g., WD Red Plus). Check out this one: [WD Red Plus 4TB](https://www.amazon.co.uk/dp/B09W3X2V7L/?tag=baronvonhag0c-21)
*   **Networking:** Ensure your server is connected via a good Cat6A cable (like this one): [Cat6A Ethernet Cable 10m](https://www.amazon.co.uk/dp/B07KVGPVXG/?tag=baronvonhag0c-21)

### 🔗 Affiliate Links
*   Beelink ME Mini: https://www.amazon.co.uk/dp/B0GKDB1RTY/?tag=baronvonhag0c-21
*   WD Red Plus 4TB: https://www.amazon.co.uk/dp/B09W3X2V7L/?tag=baronvonhag0c-21
*   Cat6A Ethernet Cable (10m): https://www.amazon.co.uk/dp/B07KVGPVXG/?tag=baronvonhag0c-21
