# Architecture Complète — JARVIS Cluster + OpenClaw + Claude Code

## Vue d'ensemble

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         LA CRÉATRICE — M1 (192.168.1.85)                        │
│                    Ryzen 5700X3D · 48GB RAM · 6 GPUs · Ubuntu                  │
│                                                                                 │
│  ┌──────────────────────────────────────────────────────────────────────────┐   │
│  │                        CLAUDE CODE (Sonnet 4.6)                          │   │
│  │                    Session TTY3 (Ctrl+Alt+F3)                            │   │
│  │  RÔLE : Orchestration uniquement — délègue TOUT aux LLMs locaux         │   │
│  └──────────────────────────┬───────────────────────────────────────────────┘   │
│                             │ lm-ask.sh / claudelm / MCP                        │
│  ┌──────────────────────────▼───────────────────────────────────────────────┐   │
│  │              TOKEN ECONOMY — DÉCISION D'ESCALADE                         │   │
│  │                                                                          │   │
│  │  Tâche simple (grep, status, résumé)  ──────→  lm-ask.sh [0 token]      │   │
│  │  Code routinier / refactor            ──────→  lm-ask.sh --big [0]      │   │
│  │  Reasoning / debug logique            ──────→  lm-ask.sh --reason [0]   │   │
│  │  Agent spécialisé (trading, sys...)   ──────→  OpenClaw agent [0]       │   │
│  │  Web volumineuse                      ──────→  WebFetch + lm-ask.sh [0] │   │
│  │  Décision archi / debug critique      ──────→  Claude lui-même [💸]     │   │
│  └──────────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## Cluster LLM — Nœuds et Modèles

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                            CLUSTER LLM JARVIS                                   │
│                                                                                 │
│  ┌─────────────────── M1 (192.168.1.85:1234) ──────────────────┐               │
│  │  GPU: RTX2060(125W) + 4×GTX1660 + RTX3080(250W) [Vulkan]   │               │
│  │                                                              │               │
│  │  ├─ qwen/qwen3.5-9b                [PRIMARY FAST]           │               │
│  │  ├─ deepseek/deepseek-r1-0528      [REASONING]              │               │
│  │  ├─ qwen3.5-27b-claude-4.6-distil  [OPUS DISTILLED ⭐]      │               │
│  │  ├─ glm-4.7-flash-claude-distill   [ULTRA FAST OPUS]        │               │
│  │  ├─ google/gemma-4-26b-a4b         [VISION]                 │               │
│  │  ├─ openai/gpt-oss-20b             [GPT COMPAT]             │               │
│  │  └─ claudegpt-code-logic-debugger  [DEBUG SPÉCIALISÉ]       │               │
│  └──────────────────────────────────────────────────────────────┘               │
│                                                                                 │
│  ┌─────────────────── M2 (192.168.1.26:1234) ──────────────────┐               │
│  │  GPU: dédiés                                                 │               │
│  │                                                              │               │
│  │  ├─ qwen/qwen3.5-9b                [WORKER SYNC]            │               │
│  │  ├─ qwen/qwen3.5-35b-a3b           [BIG TASKS]              │               │
│  │  ├─ deepseek/deepseek-r1-0528      [REASONING]              │               │
│  │  ├─ zai-org/glm-4.7-flash          [ULTRA FAST]             │               │
│  │  ├─ openai/gpt-oss-20b             [GPT COMPAT]             │               │
│  │  └─ qwen/qwen3-8b                  [LIGHT]                  │               │
│  └──────────────────────────────────────────────────────────────┘               │
│                                                                                 │
│  ┌─────────────────── OL1 (127.0.0.1:11434) ───────────────────┐               │
│  │  Ollama local                                                │               │
│  │  ├─ gemma3:4b         [FALLBACK LÉGER]                      │               │
│  │  ├─ deepseek-r1:7b   [REASONING LOCAL]                      │               │
│  │  ├─ qwen2.5:1.5b     [ULTRA TINY]                           │               │
│  │  ├─ llama3.2:latest  [LLAMA]                                │               │
│  │  └─ kimi-k2.5:cloud  [WEB SEARCH]                           │               │
│  └──────────────────────────────────────────────────────────────┘               │
│                                                                                 │
│  ┌─────────────────── CLOUD GRATUITS ──────────────────────────┐               │
│  │  Groq:       llama-3.3-70b, gemma2-9b, llama-3.1-8b        │               │
│  │  OpenRouter: deepseek-r1:free, qwen3-235b:free, llama:free  │               │
│  │  HuggingFace: Llama-3.2-3B, Phi-3.5-mini                   │               │
│  │  xAI:        grok-4-reasoning, grok-3, grok-3-mini          │               │
│  └──────────────────────────────────────────────────────────────┘               │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## Stratégie Dual Parallel (lm-ask.sh)

```
Claude Code / OpenClaw / User
           │
           ▼
      lm-ask.sh
           │
    ┌──────┴──────┐  4 slots SIMULTANÉS (par défaut)
    │             │
    ├── M1/qwen3.5-9b ─────────────────────────────┐
    ├── M1/deepseek-r1 ────────────────────────────┤
    ├── M2/qwen3.5-9b ─────────────────────────────┤──→ Premier non-vide → sortie
    └── M2/deepseek-r1 ────────────────────────────┘
           │ (si tous échouent après 120s)
           ▼
        OL1/gemma3:4b

Options :
  --reason  → force deepseek-r1 sur M1+M2 (4 slots reasoning)
  --big     → M1:qwen9b + M2:qwen35b (puissance max)
  --fast    → qwen9b sur M1+M2 uniquement
  --seq     → séquentiel M1→M2→OL1 (debug seulement)
```

---

## Conteneurs Docker — Stack JARVIS (15 actifs)

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         DOCKER CONTAINERS — M1                                  │
│                                                                                 │
│  INFRASTRUCTURE                                                                 │
│  ├─ jarvis-n8n          :5678  [Workflow automation — LLM routing webhook]      │
│  ├─ jarvis-pipeline     :9742  [Pipeline principal — trading/monitoring]        │
│  ├─ jarvis-ws           :—     [WebSocket hub interne]                          │
│  ├─ jarvis-proxy        :18800 [Proxy LLM unifié — route M1/M2/OL1]            │
│  └─ jarvis-ollama       :11434 [Ollama runtime local]                           │
│                                                                                 │
│  COWORK ENGINE                                                                  │
│  ├─ jarvis-cowork-engine      [Cycle 300s — dispatch tâches auto]              │
│  ├─ jarvis-cowork-dispatcher  [Routing tâches → agents]                        │
│  └─ jarvis-cowork-mcp         [Bridge MCP ↔ cowork]                            │
│                                                                                 │
│  OPENCLAW (gateway TUI + 69 agents)                                             │
│  ├─ openclaw-gateway    :18789 [Gateway WS — point d'entrée TUI]               │
│  ├─ openclaw-sbx-*            [Sandbox containers par agent]                   │
│  └─ openclaw-ui         :28789 [Control UI]                                     │
│                                                                                 │
│  TRADING                                                                        │
│  └─ jarvis-tradingview  :5555  [TradingView clone — MEXC+CoinEx]               │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## OpenClaw — 69 Agents Spécialisés

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                        OPENCLAW AGENT REGISTRY                                  │
│                                                                                 │
│  CORE                                                                           │
│  main · master · sys-ops · linux-admin · ai-engine · cluster-mgr               │
│                                                                                 │
│  AUTOMATION                                                                     │
│  automation · monitoring · ops-sre · voice-engine · comms                      │
│                                                                                 │
│  BUSINESS                                                                       │
│  browser-ops · linkedin-agent · codeur-agent · mail-agent                      │
│                                                                                 │
│  COWORK DEV                                                                     │
│  cowork-codegen · cowork-testing · cowork-deploy · cowork-docs                 │
│  cowork-monitor · cowork-git · cowork-refactor                                 │
│                                                                                 │
│  OMEGA (spécialisés haute qualité)                                              │
│  omega-analysis · omega-dev · omega-docs · omega-security                      │
│  omega-system · omega-voice                                                     │
│                                                                                 │
│  JARVIS INTERNES                                                                │
│  jarvis-cron · jarvis-memory · + 40 autres agents domaine                      │
│                                                                                 │
│  MODÈLE PAR DÉFAUT : lmstudio-m1/qwen/qwen3.5-9b                               │
│  FALLBACKS : m2/qwen → m1/r1 → m2/r1 → ol/gemma3                              │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## n8n Workflow — LLM Dual Routing

```
POST /webhook/llm-ask
        │
        ├──────────────────────────┬──────────────────────────┐
        │                          │                          │
   M1/qwen3.5-9b            M1/deepseek-r1            M2/qwen3.5-9b
   HTTP Request              HTTP Request              HTTP Request
   (192.168.1.85:1234)       (192.168.1.85:1234)       (192.168.1.26:1234)
        │                          │                          │
        └──────────────────────────┴──────────────────────────┘
                                   │
                            Merge (Race — premier gagne)
                                   │
                            Respond to Webhook
                                   │
                            → Résultat JSON : {response, model, latency}
```

---

## Flux Claude Code — Économie de Tokens

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│              WORKFLOW CLAUDE CODE — DÉCISION AUTOMATIQUE                        │
│                                                                                 │
│  User → Claude Code                                                             │
│              │                                                                  │
│              ▼                                                                  │
│         [ANALYSE TÂCHE]                                                         │
│              │                                                                  │
│    ┌─────────┼──────────────────────────────────────────┐                      │
│    │         │                           │               │                      │
│    ▼         ▼                           ▼               ▼                      │
│  Bash/    lm-ask.sh               OpenClaw           Claude                    │
│  Grep/    (local LLM)             Agent              (Opus)                    │
│  Glob                                                                           │
│  [0 tok]  [0 tok]                 [0 tok]            [💸 tok]                  │
│    │         │                       │                   │                      │
│    │    ┌────┴────┐                  │                   │                      │
│    │    │ M1+M2   │                  │          Uniquement pour :               │
│    │    │ 4 slots │                  │          · décisions archi               │
│    │    │ dual    │                  │          · debug critique                │
│    │    └─────────┘                  │          · synthèse finale               │
│    │         │                       │                   │                      │
│    └─────────┴───────────────────────┴───────────────────┘                      │
│                                      │                                          │
│                               Réponse à User                                    │
│                          [Attribution: M1/qwen3] etc.                           │
└─────────────────────────────────────────────────────────────────────────────────┘

RÈGLES ESCALADE :
  1. Bash/Grep/Glob  → fichiers, regex, status, comptage          [toujours]
  2. lm-ask.sh       → résumé, classification, extract JSON        [défaut]
  3. lm-ask.sh --big → code, refactor, doc (qwen3.5-35b)          [si complexe]
  4. lm-ask.sh --reason → debug logique, analyse (deepseek-r1)    [si reasoning]
  5. OpenClaw agent  → tâche spécialisée (trading, mail, sys)     [si domaine]
  6. Claude Opus     → archi, debug critique, décision finale      [ultime]

ANTI-PATTERNS INTERDITS :
  ✗ Lire 10 fichiers pour résumer → déléguer à lm-ask.sh
  ✗ Réponses verbeuses avec tableaux décoratifs
  ✗ Répéter le même grep 3× → mémoriser dans le message
  ✗ Résumer du texte lu → déléguer
```

---

## GPU — Power Limits et Température

```
┌──────────────────────────────────────────────────────────────┐
│              GPUs — La Créatrice (6 cartes)                  │
│                                                              │
│  GPU  Modèle           VRAM    Power Limit  Rôle             │
│  ──────────────────────────────────────────────────────────  │
│  GPU0 RTX 2060         12GB    125W (min)   Primary LM Studio │
│  GPU1 GTX 1660 SUPER   6GB     défaut       Vulkan pool       │
│  GPU2 GTX 1660 SUPER   6GB     défaut       Vulkan pool       │
│  GPU3 GTX 1660 SUPER   6GB     défaut       Vulkan pool       │
│  GPU4 GTX 1660 SUPER   6GB     défaut       Vulkan pool       │
│  GPU5 RTX 3080         10GB    250W         Big models/CUDA   │
│                                                              │
│  Appliquer au démarrage :                                    │
│    sudo nvidia-smi -i 0 -pl 125                              │
│    sudo nvidia-smi -i 5 -pl 250                              │
└──────────────────────────────────────────────────────────────┘
```

---

## MCP Servers

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           MCP SERVERS ACTIFS                                    │
│                                                                                 │
│  jarvis           → Orchestration cluster principal                             │
│  jarvis-m1        → Contrôle nœud M1 (LM Studio local)                        │
│  jarvis-ol1       → Contrôle OL1 (Ollama local)                               │
│  filesystem       → Accès ~/jarvis/ + /home/turbo                             │
│  browseros        → Browser automation (56 outils + Chrome CDP :9222)         │
│  chrome-devtools  → Chrome DevTools Protocol direct                            │
│  github           → GitHub API (repos, issues, PRs)                           │
│  playwright       → Tests browser automatisés                                  │
│  google-calendar  → Calendrier                                                 │
│  canva            → Design automation                                          │
│  notion           → Knowledge base                                             │
│  context7         → Docs libs (React, Next.js, Prisma…)                       │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## Commandes Rapides

```bash
# Déléguer au dual M1+M2 (4 slots parallèles)
lm-ask.sh "ta question"

# Forcer reasoning deepseek-r1
lm-ask.sh --reason "analyse ce code"

# Forcer gros modèle (qwen3.5-35b sur M2)
lm-ask.sh --big "tâche complexe"

# Terminal LLM gratuit (branché M1)
claudelm

# Agent OpenClaw spécialisé
docker exec openclaw-sbx-agent-trading "scan MEXC"

# Power limits GPU (à relancer après reboot)
sudo nvidia-smi -i 0 -pl 125   # RTX 2060
sudo nvidia-smi -i 5 -pl 250   # RTX 3080

# OpenClaw TUI — changer de modèle
/model m1       # qwen3.5-9b M1
/model m1-r1    # deepseek-r1 M1
/model m2       # qwen3.5-9b M2
/model big      # qwen3.5-35b M2
/model cloud    # kimi web search

# n8n webhook test
curl -X POST http://127.0.0.1:5678/webhook/llm-ask \
  -H "Content-Type: application/json" \
  -d '{"prompt":"test dual routing"}'
```

---

## Fichiers de Configuration

| Fichier | Emplacement | Description |
|---------|-------------|-------------|
| OpenClaw config | `~/.openclaw/openclaw.json` | 69 agents, 9 providers, gateway |
| lm-ask.sh | `~/jarvis/scripts/lm-ask.sh` | Script dual parallel |
| MCP servers | `mcp/mcp-servers.json` | 12 MCP endpoints |
| n8n workflow | `n8n/workflow_llm_routing.json` | LLM routing webhook |
| Docker compose | `docker/docker-compose.yml` | n8n + services |
| TradingView clone | `~/tv_clone/` | Flask :5555 MEXC+CoinEx |
| CLAUDE.md global | `~/.claude/CLAUDE.md` | Règles économie tokens |
| CLAUDE.md projet | `~/jarvis/CLAUDE.md` | Auto-triggers + agents |
