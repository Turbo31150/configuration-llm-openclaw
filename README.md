# Configuration LLM OpenClaw — JARVIS Cluster

Configuration complète du routing LLM dual M1+M2 pour OpenClaw et Claude Code.

## Cluster

| Nœud | Host | Modèles chargés |
|---|---|---|
| M1 | `127.0.0.1:1234` | qwen3.5-9b, deepseek-r1-0528, qwen2.5-coder-14b |
| M2 | `192.168.1.26:1234` | qwen3.5-9b, deepseek-r1-0528, qwen3.5-35b, glm-4.7-flash, gpt-oss-20b |
| OL1 | `127.0.0.1:11434` | gemma3:4b, qwen3:1.7b, deepseek-r1:7b, kimi-k2.5:cloud |

## Stratégie

**Dual Parallel** — chaque action lance 4 requêtes simultanées (M1×2 + M2×2), le premier résultat non-vide gagne.

## Fichiers

```
├── openclaw/
│   ├── openclaw.json       # Config principale OpenClaw (providers, aliases, fallbacks)
│   ├── lm-ask.sh           # Script délégation LLM (dual parallel)
│   └── coinex_skill.json   # Skill CoinEx Futures pour OpenClaw
├── n8n/
│   └── workflow_llm_routing.json  # Workflow n8n — importer via UI
├── docker/
│   └── docker-compose.yml  # Lancer n8n + services
├── mcp/
│   └── mcp-servers.json    # Config MCP servers LM Studio + Ollama
└── docs/
    └── SCHEMA.md            # Schéma architecture complet + commandes
```

## Démarrage rapide

```bash
# 1. Lancer n8n
docker compose -f docker/docker-compose.yml up -d

# 2. Importer workflow n8n (UI: http://localhost:5678)
# Workflows → Import → n8n/workflow_llm_routing.json

# 3. Tester routing dual
echo "test" | lm-ask.sh "dis bonjour"

# 4. OpenClaw TUI
openclaw  # puis /model m1 ou /model r1
```
