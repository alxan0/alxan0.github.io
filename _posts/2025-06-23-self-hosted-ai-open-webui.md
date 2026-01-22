---
title: "Self-Hosting AI: Deploying Open-WebUI and Llama.cpp with Docker"
date: 2025-06-23 10:00:00 +0000
categories: [AI, Home Lab]
tags: [open-webui, llamacpp, docker, self-hosted, amd-gpu]
description: A guide to setting up a distributed AI environment using Open-WebUI on a server and Llama.cpp on a laptop for hardware acceleration.
---


Are you tired of rising API costs, privacy concerns with cloud-based AI, or internet dependency? What if you could run a powerful AI model, similar to ChatGPT, entirely on your own hardware? 

With the rise of open-source LLMs (Large Language Models), self-hosting is no longer just for researchers. Today, we are building a private, distributed AI platform that is extensible and user-friendly.
![OpenWebUI](/assets/openwebui.png)


## What are we building?
- **Open-WebUI** - an [extensible](https://docs.openwebui.com/features/plugin/), feature-rich, and user-friendly self-hosted AI platform designed to operate entirely offline
- **Llama.cpp** - a powerful "engine" that runs the AI models efficiently

## 1. The big picture

In this post i'll show you how to setup both services with docker. My setup is as fallow: The server runs Open-WebUI (running behind Traefik v3) which then connects to my Llama.cpp container running on my laptop which can do the heavy lifting. I do this because i want to have access to my chats from any device and I'll also want to use ChatGPT with my Open-WebUI instance, from any device.

## 2. Open-WebUI docker compose file

```yaml
services:
  openwebui:
    image: ghcr.io/open-webui/open-webui:main
    container_name: openwebui
    security_opt:
      - no-new-privileges:true
    environment:
      - WEBUI_URL=https://<subdomain.your-domain>
      - OPENAI_API_BASE_URLS=http://<laptop-ip>:8000/v1
    #ports:
    #  - 3000:8080
    volumes:
      - ${APPDATA}/openwebui:/app/backend/data
    networks:
      traefik: null
networks:
  traefik:
    name: traefik_openwebui
    external: true
```

## 3. Llama.cpp docker compose file

```yaml
services:
  llamacpp:
    image: ghcr.io/ggml-org/llama.cpp:server-vulkan
    container_name: llama_vulkan
    security_opt:
        - no-new-privileges:true
    volumes:
        - <path-to-models>:/models:ro,Z
    ports:
        - 8000:8000
    devices:
        - "/dev/dri:/dev/dri"
    command: >
        -m /models/unsloth__Qwen3-30B-A3B-Instruct-2507-GGUF/Qwen3-30B-A3B-Instruct-2507-Q4_K_M.gguf
        -c 6192
        --host 0.0.0.0
        --port 8000
        -ngl 99

```

## Useful links
- [Llama.cpp docker docs](https://github.com/ggml-org/llama.cpp/blob/master/docs/docker.md)
- [Llama.cpp server docs](https://github.com/ggml-org/llama.cpp/tree/master/tools/server)
- [Open-WebUI docs](https://docs.openwebui.com/)
- [Hugging Face models](https://huggingface.co/models)
- [Amdgpu_top for GPU utilization](https://github.com/Umio-Yasuno/amdgpu_top)
