# GPU Power Limits — La Créatrice

| GPU | Index | Max TDP | Limit appliqué | Motif |
|-----|-------|---------|----------------|-------|
| RTX 2060 | GPU0 | 184W | **125W** | Chauffait à 83°C |
| GTX 1660 SUPER ×4 | GPU1-4 | 125W | par défaut | Stable |
| RTX 3080 | GPU5 | 320W | **250W** | Sécurité |

## Appliquer au démarrage
```bash
sudo nvidia-smi -i 0 -pl 125
sudo nvidia-smi -i 5 -pl 250
```

## Ajouter à /etc/rc.local ou systemd
```ini
[Service]
ExecStartPost=nvidia-smi -i 0 -pl 125
ExecStartPost=nvidia-smi -i 5 -pl 250
```
