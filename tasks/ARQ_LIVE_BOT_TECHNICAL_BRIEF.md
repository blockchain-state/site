# ARQ LIVE BOT � Technical Brief
**Version:** 1.0  
**Date:** 17.05.2026  
**Purpose:** Full technical context for continued development by any AI model or developer

---

## 1. HARDWARE & OS

| Component | Spec |
|-----------|------|
| Machine | Dell Inspiron 15 7590 |
| CPU | Intel i9-9980HK (8c/16t, up to 5.0 GHz) |
| RAM | 32 GB |
| GPU | NVIDIA GeForce GTX 1650 (4GB VRAM, CUDA 7.5) |
| Storage | 1TB NVMe SSD |
| OS | Windows 11 |
| Linux | WSL2 Ubuntu 22.04 |

---

## 2. INFRASTRUCTURE STACK

### Docker containers (managed via `C:\n8n\docker-compose.yml`)

| Container | Image | Port | Status |
|-----------|-------|------|--------|
| `n8n` | n8nio/n8n:latest | 5678 | ? Running |
| `arq_postgres` | postgres:16 | 5432 (internal) | ? Running |
| `arq_chroma` | chromadb/chroma:latest | 8000 | ? Running |
| `open-webui` | open-webui | 3000 | ? Running |

### Docker network
All containers are in the same network: **`arq_net`** (bridge driver)  
Containers can reach each other by **container name** as hostname.

### PostgreSQL credentials
```
Host:     postgres   (container name, inside Docker network)
Port:     5432
Database: arq_db
User:     arq_user
Password: ${POSTGRES_PASSWORD}
SSL:      Disabled
```

### ChromaDB
```
Internal URL: http://chromadb:8000
External URL: http://localhost:8000
API version:  v2 (not v1!)
Heartbeat:    GET /api/v2/heartbeat
```

---

## 3. DOCKER-COMPOSE.YML (current)

```yaml
services:
  n8n:
    image: n8nio/n8n:latest
    container_name: n8n
    restart: unless-stopped
    ports:
      - "5678:5678"
    environment:
      - N8N_HOST=0.0.0.0
      - N8N_PORT=5678
      - N8N_PROTOCOL=http
      - NODE_ENV=production
      - N8N_ENCRYPTION_KEY=${N8N_ENCRYPTION_KEY}
      - NODE_OPTIONS=--max-old-space-size=8192
      - EXECUTIONS_TIMEOUT=7200
      - EXECUTIONS_TIMEOUT_MAX=14400
      - EXECUTIONS_DATA_SAVE_ON_ERROR=all
      - EXECUTIONS_DATA_SAVE_ON_SUCCESS=all
      - N8N_METRICS=true
      - WEBHOOK_URL=https://<CURRENT_CLOUDFLARE_URL>
      - N8N_EDITOR_BASE_URL=https://<CURRENT_CLOUDFLARE_URL>
    networks:
      - arq_net
    deploy:
      resources:
        limits:
          cpus: "8"
          memory: 8G
    volumes:
      - n8n_data:/home/node/.n8n
      - C:/n8n/local_files:/files

  postgres:
    image: postgres:16
    container_name: arq_postgres
    restart: unless-stopped
    environment:
      - POSTGRES_DB=arq_db
      - POSTGRES_USER=arq_user
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
    networks:
      - arq_net
    volumes:
      - postgres_data:/var/lib/postgresql/data
    deploy:
      resources:
        limits:
          cpus: "4"
          memory: 4G

  chromadb:
    image: chromadb/chroma:latest
    container_name: arq_chroma
    restart: unless-stopped
    ports:
      - "8000:8000"
    networks:
      - arq_net
    volumes:
      - chroma_data:/chroma/chroma
    deploy:
      resources:
        limits:
          cpus: "2"
          memory: 4G

networks:
  arq_net:
    driver: bridge

volumes:
  n8n_data:
  postgres_data:
  chroma_data:
```

> ?? **Important:** Replace `<CURRENT_CLOUDFLARE_URL>` with the actual tunnel URL each time cloudflared restarts (see Section 6).

---

## 4. LOCAL AI (OLLAMA)

Ollama runs in WSL2 Ubuntu, NOT in Docker. It detected the GTX 1650 and uses GPU.

```
API endpoint: http://host.docker.internal:11434  (from Docker containers)
              http://localhost:11434              (from WSL/Windows)
```

### Installed models

| Model | Size | Purpose |
|-------|------|---------|
| `mistral:7b-instruct` | 4.4 GB | Main LLM, reasoning |
| `nomic-embed-text:latest` | 274 MB | Embeddings for ChromaDB |

### Useful commands (WSL)
```bash
ollama list          # show installed models
ollama ps            # check running models + GPU usage
ollama serve &       # start ollama service (if not running)
```

---

## 5. N8N CONFIGURATION

### Access
```
Local:    http://localhost:5678
Public:   https://<CLOUDFLARE_URL>
Version:  2.18.5
```

### Credentials configured in n8n

| Name | Type | Status |
|------|------|--------|
| Ollama account | Ollama | ? Connected (http://host.docker.internal:11434) |
| ARQ OpenAI test | OpenAI | ? Connected |
| ARQ Postgres IPv4 | PostgreSQL | ? Connected (host: postgres) |
| Telegram account | Telegram API | ? Connected |
| Anthropic | Anthropic API | ? Connected |

---

## 6. CLOUDFLARE TUNNEL (Public Access)

### Problem
Cloudflare free tunnel generates a **new random URL on every restart**.  
This URL must be:
1. Updated in `docker-compose.yml` (WEBHOOK_URL and N8N_EDITOR_BASE_URL)
2. Re-registered as Telegram webhook

### How to start tunnel (PowerShell)
```powershell
& "C:\Program Files (x86)\cloudflared\cloudflared.exe" tunnel --url http://localhost:5678
```

Output example:
```
Your quick Tunnel has been created! Visit it at:
https://gonna-rio-broad-ireland.trycloudflare.com
```

### After getting new URL � update Telegram webhook (WSL)
```bash
curl -X POST "https://api.telegram.org/bot<BOT_TOKEN>/setWebhook" \
  -H "Content-Type: application/json" \
  -d '{"url": "https://<NEW_CLOUDFLARE_URL>/webhook/<WEBHOOK_ID>/webhook"}'
```

### Permanent solution (TODO)
Set up a named Cloudflare tunnel with a custom domain to avoid URL changes on restart.  
Requires: Cloudflare account + domain.

---

## 7. TELEGRAM BOT (ARQ LIVE BOT)

| Property | Value |
|----------|-------|
| Bot name | ARQ live Bot |
| Creator | @BotFather |
| Token | Stored in n8n Telegram credential |
| Webhook path | `/webhook/<ID>/webhook` |

### How to check current webhook
```bash
curl "https://api.telegram.org/bot<TOKEN>/getWebhookInfo"
```

---

## 8. POSTGRESQL TABLES

### `arq_messages` (ARQ LIVE classifier output)
```sql
CREATE TABLE arq_messages (
    id SERIAL PRIMARY KEY,
    user_id BIGINT,
    username TEXT,
    message TEXT,
    category TEXT,
    sentiment TEXT,
    action_type TEXT,
    gpt_analysis TEXT,
    created_at TIMESTAMP DEFAULT NOW()
);
```

### `arq_ensemble` (Ensemble workflow output)
```sql
CREATE TABLE arq_ensemble (
    id SERIAL PRIMARY KEY,
    input TEXT,
    gpt_response TEXT,
    mistral_response TEXT,
    meta_analysis TEXT,
    created_at TIMESTAMP DEFAULT NOW()
);
```

---

## 9. N8N WORKFLOWS

### Workflow 1: ARQ LIVE � Telegram ? GPT ? PostgreSQL

**Status:** Published (active)  
**Purpose:** Classify incoming Telegram messages using GPT-4o-mini and save to PostgreSQL.

**Flow:**
```
Telegram Trigger
      ?
GPT Classifier (gpt-4o-mini)
   System prompt: classify message ? JSON
      ?
Save to PostgreSQL (arq_messages table)
      ?
Telegram Reply (send classification back to user)
```

**Current issue with GPT Classifier:**
- `Simplify Output`: must be OFF
- `Output Content as JSON`: must be ON
- Without this fix, responses parse incorrectly ? "none | noise" output

**System prompt for GPT Classifier:**
```
You are ARQ Protocol analyzer. Classify the message and respond ONLY in JSON format:
{
  "category": "quest|coordination|idea|action|feedback|noise",
  "sentiment": "positive|neutral|negative",
  "action_type": "?idea|?action|?person|none",
  "summary": "brief summary in English",
  "bs_relevance": 0-10
}
```

**PostgreSQL query in Save node:**
```sql
INSERT INTO arq_messages (user_id, username, message, category, sentiment, action_type, gpt_analysis)
VALUES ($1, $2, $3, $4, $5, $6, $7)
```

**Query parameters expression:**
```
={{ [$json.message.from.id, $json.message.from.username, $json.message.text, 
JSON.parse($node['GPT Classifier'].json.choices[0].message.content).category, 
JSON.parse($node['GPT Classifier'].json.choices[0].message.content).sentiment, 
JSON.parse($node['GPT Classifier'].json.choices[0].message.content).action_type, 
$node['GPT Classifier'].json.choices[0].message.content] }}
```

---

### Workflow 2: ARQ Ensemble � GPT vs Mistral

**Status:** Published  
**Purpose:** Send a question to GPT-4o-mini and Mistral 7b in parallel, then synthesize with Meta Analyzer.

**Flow:**
```
Webhook Input (POST /webhook/arq-ensemble)
      ?
    +-----------------+
    ?                 ?
GPT-4o-mini      Mistral 7b (Ollama)
(Architect)      (Local model)
    +-----------------+
         ?
    Merge Results
         ?
    Meta Analyzer (GPT-4o-mini)
         ?
    +---------+
    ?         ?
PostgreSQL   Response
(arq_ensemble)
```

**Test command (WSL):**
```bash
curl -X POST "http://localhost:5678/webhook/arq-ensemble" \
  -H "Content-Type: application/json" \
  -d '{"input": "Your question here"}'
```

---

## 10. WSL2 ENVIRONMENT

### Installed tools
```
Python:  3.11.9  (via pyenv)
Node.js: v24.15.0 (via nvm)
npm:     v11.12.1
Git:     configured, SSH key added to GitHub
         GitHub handle: 01ehex
```

### Useful aliases (in ~/.bashrc)
```bash
alias arq-status='docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"'
alias pg='docker exec -it arq_postgres psql -U arq_user -d arq_db'
alias n8n-logs='docker logs n8n -f'
alias chroma='curl http://localhost:8000/api/v2/heartbeat'
alias ollama-status='ollama ps'
```

### Apply aliases
```bash
source ~/.bashrc
```

---

## 11. KNOWN ISSUES & TODO

### ?? Active Issues

| Issue | Description | Fix |
|-------|-------------|-----|
| Cloudflare URL changes on restart | Every tunnel restart = new URL, must update webhook | Set up named tunnel with custom domain |
| GPT Classifier outputs "none/noise" | Simplify Output is ON, Output as JSON is OFF | Fix both toggles in n8n node |
| PostgreSQL empty (arq_messages) | Data not saving due to above parsing issue | Fix classifier first |

### ? TODO

1. **Fix GPT Classifier** � turn off Simplify Output, turn on Output as JSON
2. **Re-test ARQ LIVE** � send message to bot, verify PostgreSQL receives data
3. **Set up permanent Cloudflare tunnel** � requires domain + Cloudflare account
4. **Build War Room workflow** � GPT + Claude (Anthropic API) + Synthesizer ? Decision Ledger
5. **Add Anthropic Claude node** � for War Room Human Layer role
6. **Connect War Room to Decision Ledger** � auto-write to PostgreSQL after each session

---

## 12. WAR ROOM WORKFLOW (PLANNED)

### Goal
Orchestrate GPT (Architect) + Claude (Human Layer) + Synthesizer in parallel for each query, save results to Decision Ledger.

### Planned flow
```
Telegram /council <question>
         ?
    +---------+
    ?         ?
GPT-4o-mini   Claude Sonnet 4.6
(Architect)   (Human Layer)
(OpenAI API)  (Anthropic API)
    +---------+
         ?
    Synthesizer (GPT-4o or Claude Opus)
         ?
    +-----------------+
    ?                 ?
Telegram reply    PostgreSQL
(3 blocks)        arq_decisions table
```

### Decision Ledger table (to create)
```sql
CREATE TABLE arq_decisions (
    id SERIAL PRIMARY KEY,
    user_prompt TEXT,
    gpt_architect TEXT,
    claude_human_layer TEXT,
    synthesis TEXT,
    next_action TEXT,
    open_risks TEXT,
    meg_dev_decision TEXT,
    created_at TIMESTAMP DEFAULT NOW()
);
```

### Role prompts (finalized)
See `prompts/roles.md` in `arq_ai_war_room_mvp.zip` or request from project owner.

---

## 13. API KEYS LOCATION

All API keys are stored in **n8n Credentials** (encrypted).  
Do NOT store keys in files or environment variables outside n8n.

| Service | Credential name in n8n |
|---------|----------------------|
| OpenAI | ARQ OpenAI test |
| Anthropic | Anthropic |
| Telegram | Telegram account |

---

## 14. QUICK START CHECKLIST

When starting fresh session:

```powershell
# 1. Start Docker containers (PowerShell)
cd C:\n8n
docker-compose up -d

# 2. Start Cloudflare tunnel (PowerShell) 
& "C:\Program Files (x86)\cloudflared\cloudflared.exe" tunnel --url http://localhost:5678
# Note the new URL!

# 3. Update Telegram webhook (WSL) if URL changed
curl -X POST "https://api.telegram.org/bot<TOKEN>/setWebhook" \
  -H "Content-Type: application/json" \
  -d '{"url": "https://<NEW_URL>/webhook/<ID>/webhook"}'

# 4. Verify everything is running (WSL)
arq-status

# 5. Open n8n
# Browser: http://localhost:5678
```

---

## 15. PROJECT CONTEXT

This infrastructure serves the **ARQ / BS Protocol / Math.R** project:

- **ARQ** � entry and training layer for human-agent coordination cells
- **BS Protocol** � coordination rules for the Math.R environment  
- **Math.R** � digital coordination environment above the internet (mathr.ch)
- **War Room** � multi-LLM adversarial council (GPT + Claude + Synthesizer) for architectural decisions

The ARQ LIVE Bot is the **first operational implementation** of ARQ � a Telegram bot that classifies incoming messages, saves them to PostgreSQL, and will eventually feed into ChromaDB for semantic search and the War Room for adversarial analysis.

**Project owner:** Meg@Dev (Valais, Switzerland)  
**GitHub:** github.com/01ehex  
**Domain:** mathr.ch (Infomaniak, Swiss hosting)