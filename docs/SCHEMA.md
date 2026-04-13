# Schéma Architecture LLM — OpenClaw Dual Routing

## Cluster LLM

```
┌─────────────────────────────────────────────────────────────────┐
│                    JARVIS CLUSTER LLM                           │
│                                                                 │
│  ┌──────────── M1 (127.0.0.1:1234) ────────────┐               │
│  │  RTX 2060 + 4×GTX 1660 + RTX 3080 (Vulkan)  │               │
│  │  ├─ qwen/qwen3.5-9b          [PRIMARY FAST]  │               │
│  │  ├─ deepseek/deepseek-r1     [REASONING]     │               │
│  │  └─ qwen/qwen2.5-coder-14b   [CODE]          │               │
│  └──────────────────────────────────────────────┘               │
│                          │                                      │
│  ┌──────────── M2 (192.168.1.26:1234) ─────────┐               │
│  │  GPU dédiés                                  │               │
│  │  ├─ qwen/qwen3.5-9b          [WORKER SYNC]  │               │
│  │  ├─ deepseek/deepseek-r1     [REASONING]    │               │
│  │  ├─ qwen/qwen3.5-35b-a3b     [BIG TASKS]    │               │
│  │  ├─ zai-org/glm-4.7-flash    [ULTRA FAST]   │               │
│  │  └─ openai/gpt-oss-20b       [GPT COMPAT]   │               │
│  └──────────────────────────────────────────────┘               │
│                          │                                      │
│  ┌──────────── OL1 (127.0.0.1:11434) ──────────┐               │
│  │  Ollama local                                 │               │
│  │  ├─ gemma3:4b    [fallback léger]             │               │
│  │  ├─ qwen3:1.7b   [ultra tiny]                 │               │
│  │  ├─ deepseek-r1:7b                            │               │
│  │  └─ kimi-k2.5:cloud [web search]              │               │
│  └──────────────────────────────────────────────┘               │
└─────────────────────────────────────────────────────────────────┘
```

## Stratégie Dual Parallel (lm-ask.sh)

```
User/Claude/OpenClaw
        │
        ▼
   lm-ask.sh
        │
   ┌────┴────┐ (4 slots simultanés)
   │         │
   ├── M1/qwen ──────────────────────┐
   ├── M1/r1  ──────────────────────┤
   ├── M2/qwen ─────────────────────┤─→ Premier non-vide → sortie
   └── M2/r1  ──────────────────────┘
        │ (si tous échouent)
        ▼
      OL1/gemma3:4b
```

## OpenClaw Routing

```
OpenClaw TUI
    │
    ├─ primary: lmstudio-m1/qwen3.5-9b  (alias: m1)
    ├─ fallback 1: lmstudio-m2/qwen3.5-9b (alias: m2)
    ├─ fallback 2: lmstudio-m1/deepseek-r1 (alias: m1-r1)
    ├─ fallback 3: lmstudio-m2/deepseek-r1 (alias: m2-r1)
    └─ fallback 4: ollama/gemma3:4b (alias: ol)
```

## n8n Workflow — LLM Dual Routing

```
Webhook POST /llm-ask
        │
        ├──────────────────────────────────────┐
        │                                      │
   M1/qwen3.5-9b                        M1/deepseek-r1
   M2/qwen3.5-9b                        M2/deepseek-r1
        │                                      │
        └──────────┬───────────────────────────┘
                   ▼
          Merge (premier gagne)
                   │
                   ▼
          Respond to Webhook
```

## Commandes rapides

```bash
# Déléguer une tâche (dual M1+M2)
lm-ask.sh "ta question"

# Forcer reasoning deepseek-r1
lm-ask.sh --reason "analyse ce code"

# Forcer gros modèle M2 35B
lm-ask.sh --big "tâche complexe"

# Séquentiel (debug)
lm-ask.sh --seq "question"

# OpenClaw TUI — changer de modèle
/model m1-r1   # deepseek reasoning
/model m2      # qwen M2
/model big     # 35B M2
/model cloud   # kimi web
```

## Fichiers de config

| Fichier | Emplacement |
|---|---|
| OpenClaw main | `~/.openclaw/openclaw.json` |
| lm-ask.sh | `~/jarvis/scripts/lm-ask.sh` |
| CoinEx skill | `~/jarvis-cowork/coinex_skill.json` |
| n8n workflow | `n8n/workflow_llm_routing.json` |
| Docker compose | `docker/docker-compose.yml` |
| MCP servers | `mcp/mcp-servers.json` |

## n8n — Import du workflow

1. Ouvrir http://localhost:5678
2. Menu → Workflows → Import from file
3. Sélectionner `n8n/workflow_llm_routing.json`
4. Activer le workflow
5. Tester : `curl -X POST http://localhost:5678/webhook/llm-ask -d '{"prompt":"test"}'`

## Clé API n8n (générée)

```
N8N_API_KEY=YOUR_N8N_API_KEY
N8N_URL=http://localhost:5678
```
