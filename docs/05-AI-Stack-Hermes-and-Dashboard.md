# 05 — Edge AI Stack, Model Downloads, Hermes Agent, `ask` CLI & Homepage Dashboard

This guide covers deploying the local AI stack (`ai-iot-stack.yml`), pulling and scheduling **Ollama** LLM models (`gemma4:e2b`), installing the **NousResearch Hermes Agent CLI** with Cloudflare Service Token authentication, setting up the **`ask` + `glow`** terminal assistant, and configuring the **Homepage Dashboard (`Syntax Edge Hub`)**.

---

## 1. Deploy the Swarm AI Stack (`ai-iot-stack.yml`)

The AI stack deploys **Ollama** (`ai_stack_ollama:11434`) and **Open WebUI** (`ai_stack_open-webui:8080`, published on host port `3000`) onto the `hub_net` overlay network.

> **💡 CPU Throttling for 24/7 Stability:** In `ai-iot-stack.yml`, `OLLAMA_NUM_THREADS=3` caps LLM inference to 3 CPU threads on the Intel Core i3 processor, ensuring 1 thread is always reserved for RHEL OS tasks, Nginx Proxy Manager, AdGuard DNS, and Jellyfin.

```bash
# Deploy the AI stack
docker stack deploy -c ~/ai-iot-stack.yml ai_stack

# Verify services are running (1/1 replicas)
docker service ls --filter "name=ai_stack"
```

---

## 2. Pulling Ollama Models & Off-Peak Nightly Scheduled Downloads

### 2.1 Pull Models Immediately via `docker exec`
Because Ollama runs inside a Docker Swarm service (`ai_stack_ollama`), use `docker ps -q -f name=ai_stack_ollama` to dynamically target the running container ID:

```bash
# 1. Pull the lightweight Gemma 4 (2B) model
docker exec -it $(docker ps -q -f name=ai_stack_ollama) ollama pull gemma4:e2b

# 2. List all downloaded models in the ollama_data volume
docker exec -it $(docker ps -q -f name=ai_stack_ollama) ollama list

# 3. Test the model directly in the terminal (optional)
docker exec -it $(docker ps -q -f name=ai_stack_ollama) ollama run gemma4:e2b "Hello from syntax homelab!"
```

### 2.2 Schedule Off-Peak Nightly Model Downloads (`00:05 AM`)
To pull large LLM models overnight without consuming daytime home internet bandwidth, schedule a transient systemd timer with `systemd-run`:

```bash
sudo systemd-run --on-calendar="*-*-* 00:05:00" --unit=ollama-night-pull \
  /usr/bin/docker exec $(docker ps -q -f name=ai_stack_ollama) \
  sh -c "ollama pull gemma4:e2b && ollama pull hermes3:3b"

# Check active systemd timers
systemctl list-timers --all | grep ollama
```

### 2.3 Using Open WebUI (`https://ai.coinflow.lk`)
1. Open `https://ai.coinflow.lk` (or `http://192.168.1.3:3000`).
2. Create your admin account on first launch.
3. Because `OLLAMA_BASE_URL=http://ollama:11434` is set in `ai-iot-stack.yml`, Open WebUI automatically discovers `gemma4:e2b` and any other models inside `ai_stack_ollama`.

---

## 3. NousResearch Hermes Agent CLI & Cloudflare Service Token Setup

### 3.1 Install Prerequisite (`libatomic`) & Hermes Agent on RHEL 10.1
Node.js / prebuilt binaries bundled by Hermes require `libatomic` on RHEL 10.1:

```bash
# 1. Install libatomic dependency on RHEL 10.1
sudo dnf install -y libatomic

# 2. Run the official NousResearch Hermes Agent installer
curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash

# 3. Ensure ~/.local/bin is on PATH and verify version
export PATH="$HOME/.local/bin:$PATH"
hermes --version
```

### 3.2 Authenticate Hermes Against `https://ollama.coinflow.lk/v1`
When querying `https://ollama.coinflow.lk/v1` from a terminal or external laptop, Cloudflare Access blocks requests that don't include a valid **Service Token** header.

Edit `~/.hermes/config.yaml` and add your Cloudflare Service Token headers (created in **[03-Cloudflare-NPM-and-AdGuard-DNS.md](./03-Cloudflare-NPM-and-AdGuard-DNS.md#22-configure-access-policies--jellyfin-mobile-app-bypass)**):

```yaml
model:
  provider: custom
  default: gemma4:e2b
  base_url: "https://ollama.coinflow.lk/v1"
  extra_headers:
    CF-Access-Client-Id: "<CF_SERVICE_TOKEN_CLIENT_ID_PLACEHOLDER>"
    CF-Access-Client-Secret: "<CF_SERVICE_TOKEN_CLIENT_SECRET_PLACEHOLDER>"
```

Launch the interactive agent:
```bash
hermes chat
```

---

## 4. Terminal AI Assistant (`ask` + `glow`) Setup on RHEL 10.1

In addition to local Ollama models, `syntax` includes the `/usr/local/bin/ask` CLI helper for instant shell command synthesis (`ask "query"`) and formatted markdown explanations (`ask -e "query"`).

### 4.1 Install `glow` Markdown Renderer (Charm RPM Repo)
```bash
sudo tee /etc/yum.repos.d/charm.repo > /dev/null << 'EOF'
[charm]
name=Charm
baseurl=https://repo.charm.sh/yum/
enabled=1
gpgcheck=1
gpgkey=https://repo.charm.sh/yum/gpg.key
EOF

sudo dnf makecache
sudo dnf install -y glow jq curl
```

### 4.2 Install `/usr/local/bin/ask` Script
Create `/usr/local/bin/ask` and make it executable (`sudo chmod +x /usr/local/bin/ask`), then add your provider environment variables (using placeholders) to `~/.bashrc`:

```bash
# Append to ~/.bashrc (replace <PLACEHOLDER> with your actual API key on the server)
cat << 'EOF' >> ~/.bashrc

# Ask CLI Configuration (Gemini / DeepSeek)
export ASK_PROVIDER="gemini"                # Options: "gemini" or "deepseek"
export GEMINI_API_KEY="<GEMINI_API_KEY_PLACEHOLDER>"
export GEMINI_MODEL="gemini-3.5-flash-lite"

export DEEPSEEK_API_KEY="<DEEPSEEK_API_KEY_PLACEHOLDER>"
export DEEPSEEK_MODEL="deepseek-chat"

export PATH="$HOME/.local/bin:$PATH"
EOF

source ~/.bashrc
```

> **⚠️ Bash vs. Zsh Gotcha (`nocorrect`):** Never copy `alias ask='nocorrect ask'` from a Zsh `.zshrc` into RHEL's `~/.bashrc`. `nocorrect` is a Zsh-only keyword and causes `bash: nocorrect: command not found` on RHEL!

### 4.3 Test `ask` CLI
```bash
ask --status
ask "find folder size of Moana in current directory"
ask -e "default package installer in rhel 10 like apt in ubuntu"
```

---

## 5. Homepage Dashboard (`Syntax Edge Hub`) Deployment & Git Backup

### 5.1 Deploy the Homepage Stack (`homepage-stack.yml`)
Homepage runs on port `3001` (`https://home.coinflow.lk`) and mounts:
- `/var/run/docker.sock:/var/run/docker.sock:ro` (with `swarm: true` in `~/homepage/config/docker.yaml` to monitor Swarm services)
- `/:/host:ro` (to display real-time disk usage for `/` and `/mnt/win_1tb` via `~/homepage/config/widgets.yaml`)

```bash
docker stack deploy -c ~/homepage-stack.yml homepage
docker service ls --filter "name=homepage"
```

### 5.2 Key Homepage Gotchas
1. **`HOMEPAGE_ALLOWED_HOSTS` (`403 Forbidden` Fix):**
   Homepage blocks requests whose `Host` header is not explicitly listed in `HOMEPAGE_ALLOWED_HOSTS`. In `homepage-stack.yml`, this is set to:
   ```yaml
   environment:
     - HOMEPAGE_ALLOWED_HOSTS=home.coinflow.lk,localhost:3001
   ```
2. **Sanitizing `services.yaml` Before Pushing to GitHub:**
   Because `~/homepage/config/services.yaml` is tracked in Git via `.gitignore`, always ensure the `cloudflared` widget's `accountid`, `tunnelid`, and `key` fields are left blank (`""`) or use `{{HOMEPAGE_VAR_...}}` environment variables before committing.
