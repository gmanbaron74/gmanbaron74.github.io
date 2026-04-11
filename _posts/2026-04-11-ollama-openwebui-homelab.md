# Running Your Own LLM Playground: How to Set Up Ollama and Open WebUI on Any Home Server

The hype cycle around AI has never been faster, and with that speed comes the realization that reliance on closed, proprietary cloud services is not sustainable for serious hobbyists or technical enthusiasts. If you’ve spent time tinkering with Docker containers for Plex, qBittorrent, or Grafana, you know that running things locally grants a level of control and privacy that the cloud simply cannot match. The same principle now applies to Large Language Models (LLMs).

This guide will walk you through setting up your own private, self-hosted AI playground. We’re pairing **Ollama**—an incredibly simple tool for running local models—with **Open WebUI**—a polished, ChatGPT-like interface that makes interacting with those raw models feel seamless and professional. Whether you are on an old NUC box in the corner or a dedicated mini PC like the Beelink machines we've discussed before, this stack can bring powerful AI capabilities directly into your home network.

## Why Self-Host Your LLM Experience? (The Privacy & Cost Angle)

Before diving into the Docker Compose syntax, let’s cover *why* you should go through this effort. The biggest draw is control. When you use a public chatbot service, your inputs are data points flowing across someone else's servers—data that can be used for training, rate-limited, or subject to sudden policy changes.

By running Ollama and Open WebUI at home, you achieve several goals:
1.  **Absolute Privacy:** Your prompts, the context of your work, and any sensitive code snippets never leave your premises. This is crucial if you are dealing with proprietary internal documentation or personal journals.
2.  **Cost Predictability:** After the initial hardware investment, running a local LLM costs only electricity. There are no surprise usage bills popping up in a cloud provider dashboard.
3.  **Customization & Depth:** You can integrate RAG (Retrieval-Augmented Generation) against *your own* documents—your old project notes, your manuals, even the files in this workspace—making the AI an expert on *your* life and projects, not just its training data.

## Prerequisite Hardware Checklist

While this guide is designed to be hardware-agnostic, performance matters immensely when running LLMs. To avoid a frustrating experience where every response takes three minutes, pay attention to these specs. Remember, the goal here isn't necessarily the fastest GPU money can buy, but one that efficiently handles inference for quantized models.

*   **The Core:** Any modern mini PC (Intel NUC, Beelink, etc.) running Linux is ideal.
*   **RAM:** 16GB minimum, 32GB recommended if you plan on running multiple services simultaneously (e.g., Home Assistant *and* AI).
*   **GPU (Highly Recommended):** If your mini PC has an integrated GPU with enough VRAM, or if you add a low-power discrete card, performance will jump dramatically. For example, dedicated ML cards are overkill; even older NVIDIA cards can accelerate the process immensely by offloading computations from the CPU.

***Amazon UK Affiliate Link Suggestion:*** *If upgrading your compute power is necessary, look at modern mini PCs on Amazon. Keep an eye out for models with high core counts and decent integrated graphics.* [https://www.amazon.co.uk/dp/LINK_TO_MINI_PC_HERE/?tag=baronvonhag0c-21]

## Step 1: Setting Up the Foundation (Docker Compose)

The cleanest, most repeatable way to run this stack is via Docker Compose. This isolates the services, meaning if one component fails or needs an update, it won't take down your entire home server ecosystem.

We will create a `docker-compose.yml` file in a dedicated directory—let’s call it `ai_playground`.

**Actionable Steps:**
1.  Navigate to your desired working directory:
    ```bash
    cd /home/node/.openclaw/workspace/ai_playground
    ```
2.  Create the necessary volume for persistent data storage. This ensures that even if you rebuild the containers, your downloaded models and chat history remain intact.
    ```bash
    mkdir -p ./data
    docker volume create ollama-data
    ```
3.  Now, write the core configuration file.

Here is the required content for `docker-compose.yml`. **Ensure you save this exact text into that file.**

````yaml
version: '3.8'

services:
  ollama:
    image: ollama/ollama
    container_name: ollama
    volumes:
      - ./data/ollama:/root/.ollama # Persistent storage for downloaded models
    ports:
      - "11434:11434" # Standard Ollama port
    restart: unless-stopped

  open-webui:
    image: ghcr.io/open-webui/open-webui:main
    container_name: open-webui
    depends_on:
      - ollama
    environment:
      # This tells the Open WebUI container where to find the Ollama API
      - OLLAMA_BASE_URL=http://ollama:11434
    ports:
      - "3000:8080" # Exposes WebUI on port 3000 of your host machine
    restart: unless-stopped
````

## Step 2: Deployment and First Run

With the file saved, deployment is a single command. This pulls the required images (Ollama core service and the Open WebUI frontend) and starts them in an orchestrated manner.

```bash
docker compose up -d
```

Wait about two minutes after running this. Docker will download everything needed. A successful startup means both containers are running, and you should see no major errors in the logs.

## Step 3: Initial Model Download & Interaction

Your services are up, but they aren't smart yet. You need to tell Ollama which brain models it can use. We'll start with Llama 3, a fantastic general-purpose model that performs exceptionally well for its size.

Since the web UI is what you interact with, we need to run a command *against* the `ollama` container itself first.

```bash
docker exec -it ollama ollama pull llama3
```

Once that finishes (it downloads several gigabytes of weights), your local AI backend is ready for advanced prompting.

Finally, open your web browser and navigate to `http://localhost:3000`. You will see the Open WebUI login screen. For simplicity in this initial setup, you can often bypass registration by using the "Sign up" feature with a simple email/password combo that only *you* know, or if it supports it, skip initial authentication for local testing.

## Advanced Context: Integrating Your Knowledge Base (RAG)

The true power comes when you move beyond general chat and teach the AI about *your* specific domain knowledge—the stuff stored in your home directory. This requires setting up a Vector Database and an embedding model, but conceptually, it works like this:

1.  **Ingest:** You feed documents (PDFs of old manuals, Markdown files from `/home/node/.openclaw/workspace`) into a dedicated ingestion service.
2.  **Embed:** The system runs those documents through an embedding model (often hosted locally via Ollama) which converts text chunks into numerical vectors.
3.  **Store & Retrieve:** These vectors are stored in a database (like ChromaDB or Qdrant, which you'd run in another container). When you ask a question, the system searches this vector store for *semantic similarity* to your query and retrieves the top N relevant document chunks.
4.  **Generate:** Finally, it passes these retrieved text snippets *alongside* your prompt into Llama 3: "Using the following context: [CONTEXT SNIPPETS], answer this question: [USER QUESTION]."

This RAG pipeline is what transforms a chatbot playground into a genuine personal expert system for your digital life.

## Troubleshooting Common Issues

*   **Connection Refused on Port 11434:** Check that no other service (like an old local LLM instance) is using port 11434. Use `sudo lsof -i :11434` to check.
*   **Open WebUI Can't Talk to Ollama:** Ensure the `OLLAMA_BASE_URL` environment variable in your `docker-compose.yml` correctly points to `http://ollama:11434`. Container networking relies on service names (like `ollama`), not host IPs, when running together.
*   **Performance is Slow:** Your bottleneck is almost certainly VRAM or system RAM capacity relative to the model size. Consider using highly quantized versions of models (`Q4_K_M`) as they offer a significant speed boost for minimal quality loss.

By following these steps—from container setup to model pulling and finally, thinking about RAG integration—you have built a robust, private AI backbone on your own infrastructure. This self-hosted capability is not just a tech trend; it’s a fundamental shift in how personal computing power should be managed moving forward.
