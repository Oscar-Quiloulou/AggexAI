# Guide d'Intégration : vLLM + FastAPI + WireGuard + PyTools

Ce guide décrit comment combiner vLLM, FastAPI, WireGuard et un wrapper PyTools pour créer un système d'IA complet et sécurisé. Aucun code n'est fourni, uniquement les spécifications et exigences.

## Vue d'ensemble de l'architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    RÉSEAU PUBLIC (Internet)                      │
└─────────────────────────────────────────────────────────────────┘
                              │
                              │ Tunnel WireGuard (UDP 51820)
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    SERVEUR UBUNTU (GPU)                          │
│                                                                  │
│  ┌────────────────┐    ┌────────────────┐    ┌───────────────┐  │
│  │   FastAPI      │───▶│     vLLM       │───▶│     GPU       │  │
│  │   (API REST)   │    │ (Inférence LLM)│    │  (CUDA/ROCm)  │  │
│  └────────────────┘    └────────────────┘    └───────────────┘  │
│         │                       │                                │
│         │              ┌────────┴────────┐                       │
│         │              │  PagedAttention │                       │
│         │              │ Continuous Batch│                       │
│         │              └─────────────────┘                       │
│         │                                                        │
│  ┌──────┴───────────────────────────────────────────────┐      │
│  │              Réseau WireGuard (10.8.0.0/24)           │      │
│  └──────────────────────────────────────────────────────┘      │
└─────────────────────────────────────────────────────────────────┘
                              │
                              │
┌─────────────────────────────┴───────────────────────────────────┐
│                    CLIENTS DISTANTS (Via VPN)                   │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐  │
│  │  PyTools    │  │  App Mobile │  │  Autre Application      │  │
│  │  (Wrapper)  │  │  (Chat)     │  │  (SDK OpenAI)           │  │
│  └─────────────┘  └─────────────┘  └─────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

## Partie 1 : Intégration vLLM + FastAPI

### Principe

vLLM expose nativement un serveur HTTP compatible avec l'API OpenAI . FastAPI sert de couche d'orchestration pour :
- Gérer les sessions et le contexte conversationnel
- Enrichir les requêtes (system prompts, tools)
- Ajouter l'authentification
- Router vers d'autres services (RAG, bases de données)

### Deux approches possibles

**Approche A : FastAPI comme proxy intelligent**

FastAPI reçoit les requêtes, les enrichit, puis appelle le serveur vLLM via le SDK OpenAI.

| Composant | Rôle |
|-----------|------|
| Client → FastAPI | Requêtes avec contexte, historique, métadonnées |
| FastAPI → vLLM | Appel HTTP `/v1/chat/completions` avec SDK OpenAI |
| vLLM → FastAPI | Réponse compatible OpenAI |
| FastAPI → Client | Réponse enrichie (streaming, logs, etc.) |

**Approche B : Utilisation directe de vLLM comme moteur dans FastAPI**

FastAPI importe directement `LLM` de vLLM et gère l'inférence en interne .

| Avantage | Inconvénient |
|----------|--------------|
| Contrôle total | Un seul processus, pas de scaling horizontal |
| Pas de latence réseau | Dépendance forte entre API et GPU |
| Idéal pour prototypage | Moins adapté à la production |

### Recommandation

Pour un système multi-utilisateurs avec contexte et outils, utilisez **l'approche A**. vLLM gère l'inférence optimisée, FastAPI gère la logique applicative.

### Gestion du contexte conversationnel

Le contexte est envoyé dans le tableau `messages` de l'API Chat Completions  :

| Rôle | Contenu |
|------|---------|
| `system` | Instructions globales, persona, contraintes |
| `user` | Message utilisateur |
| `assistant` | Réponse précédente du modèle |
| `tool` | Résultat d'un appel d'outil |

**Stratégies de gestion du contexte :**
- **Fenêtre glissante** : Limiter le nombre de messages (ex: 10 derniers échanges)
- **Résumé automatique** : Compresser les anciens échanges via un LLM
- **Session ID** : Utiliser le paramètre `session_id` supporté par vLLM 

### Tool Calling (Appel d'outils)

vLLM supporte le tool calling compatible OpenAI  :

| Élément | Description |
|---------|-------------|
| `tools` | Liste des fonctions avec schémas JSON |
| `tool_choice` | `auto`, `required`, ou nom spécifique |
| `--tool-call-parser` | Flag vLLM pour le parser (ex: `qwen3_coder`, `hermes`) |
| `--enable-auto-tool-choice` | Active la détection automatique |

**Exigences pour le tool calling :**
- Le modèle doit être fine-tuné pour le function calling
- Le parser doit correspondre au format du modèle
- Les schémas JSON doivent être valides

### Requirements techniques

| Composant | Version minimum | Notes |
|-----------|-----------------|-------|
| Python | 3.10+ (3.12 recommandé) | Pour FastAPI et vLLM |
| FastAPI | 0.100+ | Support async natif |
| Uvicorn | 0.23+ | Serveur ASGI |
| openai (SDK) | 1.0+ | Client pour appeler vLLM |
| vllm | 0.6+ | Dernières versions recommandées |
| pydantic | 2.0+ | Validation des schémas |

---

## Partie 2 : Intégration WireGuard

### Principe

WireGuard crée un tunnel VPN chiffré entre les clients et le serveur. Le serveur vLLM/FastAPI n'est **jamais exposé publiquement** .

### Architecture réseau

```
┌─────────────────────────────────────────────────────────────┐
│                    RÉSEAU VPN (10.8.0.0/24)                  │
│                                                              │
│   Serveur: 10.8.0.1                                          │
│   Client A: 10.8.0.2                                         │
│   Client B: 10.8.0.3                                         │
│   PyTools: 10.8.0.4                                          │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Configuration serveur

| Élément | Valeur |
|---------|--------|
| Interface | `wg0` |
| Adresse serveur | `10.8.0.1/24` |
| Port | UDP 51820 |
| NAT | Activé pour le trafic sortant |

### Configuration pare-feu

Le pare-feu doit **bloquer l'accès public** à FastAPI et vLLM, et **autoriser uniquement le VPN**  :

| Port | Source | Action |
|------|--------|--------|
| 8000 (FastAPI) | wg0 uniquement | ACCEPT |
| 8000 (FastAPI) | eth0 (public) | DROP |
| 51820 (WireGuard) | Toute source | ACCEPT |
| 22 (SSH) | wg0 uniquement | ACCEPT |

### Requirements techniques

| Composant | Version | Notes |
|-----------|---------|-------|
| wireguard-tools | 1.0+ | Paquet Ubuntu |
| Noyau Linux | 5.6+ | WireGuard intégré |
| systemd | 245+ | Pour `wg-quick@wg0` |

### Split Tunneling (Recommandé)

Pour éviter que tout le trafic client passe par le VPN, configurez `AllowedIPs = 10.8.0.0/24` côté client . Seul le trafic vers le serveur d'inférence passe par le tunnel.

---

## Partie 3 : PyTools (Wrapper IA)

### Définition

PyTools est une bibliothèque Python qui sert de **couche d'abstraction** entre l'application cliente et l'API FastAPI/vLLM. Elle encapsule :
- La gestion de session et contexte
- L'appel aux outils (tools)
- Le streaming des réponses
- La gestion des erreurs et retries

### Rôle dans l'architecture

```
Application → PyTools → FastAPI (via VPN) → vLLM
```

### Modules requis

| Module | Rôle | Dépendance externe |
|--------|------|-------------------|
| `client` | Client HTTP vers FastAPI | `httpx` ou `openai` |
| `session` | Gestion de session et contexte | `pydantic` |
| `tools` | Définition et exécution locale des outils | `jsonschema` |
| `streaming` | Gestion du streaming SSE | `httpx-sse` |
| `config` | Configuration (URL, clés, timeouts) | `pydantic-settings` |
| `errors` | Gestion des erreurs et retries | `tenacity` (optionnel) |

### Fonctionnalités attendues

**Gestion de session :**
- Création d'un ID de session
- Persistance du contexte conversationnel
- Limitation de la fenêtre de contexte (token counting)

**Gestion des outils :**
- Définition des outils locaux avec schémas JSON
- Exécution locale des fonctions quand le LLM les appelle
- Retour du résultat au LLM pour complétion

**Streaming :**
- Support du streaming token par token
- Callbacks pour chaque chunk reçu
- Gestion des événements `tool_calls` en streaming 

**Configuration :**
- URL du serveur FastAPI (via VPN, ex: `http://10.8.0.1:8000/v1`)
- Clé API (même si vLLM l'ignore, le SDK en exige une) 
- Timeouts, retries, backoff

### Requirements Python

| Package | Version | Usage |
|---------|---------|-------|
| `openai` | ≥1.0 | Client API compatible |
| `httpx` | ≥0.25 | Requêtes HTTP async |
| `httpx-sse` | ≥0.4 | Streaming SSE |
| `pydantic` | ≥2.0 | Modèles de données |
| `pydantic-settings` | ≥2.0 | Configuration |
| `tenacity` | ≥8.0 | Retries (optionnel) |
| `tiktoken` | ≥0.5 | Comptage de tokens (optionnel) |

### Configuration minimale

```toml
# pytools.toml
[server]
base_url = "http://10.8.0.1:8000/v1"  # IP VPN du serveur
api_key = "EMPTY"
timeout = 300

[model]
default = "meta-llama/Llama-3-8B-Instruct"
max_tokens = 2048
temperature = 0.7

[session]
max_history_tokens = 8192
strategy = "sliding_window"  # ou "summarize"
```

---

## Partie 4 : Exigences globales

### Matériel serveur

| Composant | Minimum | Recommandé |
|-----------|---------|------------|
| GPU | NVIDIA 8GB VRAM | NVIDIA 24GB+ (A5000, A100) |
| RAM | 16 GB | 64 GB+ |
| CPU | 4 cœurs | 8+ cœurs |
| Stockage | 50 GB SSD | 500 GB NVMe |
| Réseau | 100 Mbps | 1 Gbps |

### Système d'exploitation

| Élément | Version |
|---------|---------|
| Ubuntu | 22.04 LTS ou 24.04 LTS |
| Noyau | ≥5.6 (pour WireGuard) |
| CUDA | 12.x (NVIDIA) |
| Pilote NVIDIA | ≥525 |

### Sécurité

| Mesure | Implémentation |
|--------|----------------|
| Chiffrement transport | WireGuard (ChaCha20) |
| Authentification | Clés WireGuard + clé API FastAPI |
| Isolation réseau | Aucune exposition publique |
| Mise à jour | `unattended-upgrades` activé |

### Performance

| Métrique | Cible |
|----------|-------|
| Time To First Token (TTFT) | <500 ms |
| Throughput | >100 tokens/s |
| Utilisation GPU | >80% |
| Latence VPN ajoutée | <1 ms  |

---

## Résumé des requirements par composant

| Composant | Requirements principaux |
|-----------|------------------------|
| **vLLM** | Python 3.10+, CUDA 12.x, GPU NVIDIA 7.0+, `uv`, `vllm[all]` |
| **FastAPI** | `fastapi`, `uvicorn`, `pydantic≥2.0`, `openai≥1.0` |
| **WireGuard** | `wireguard-tools`, noyau ≥5.6, `iptables` pour NAT |
| **PyTools** | `openai≥1.0`, `httpx`, `pydantic`, `httpx-sse` |
| **Système** | Ubuntu 22.04+, GPU ≥8GB, RAM ≥16GB, SSD ≥50GB |

---

## Flux d'une requête complète

```
1. Client (PyTools) → Chiffre et envoie via tunnel WireGuard
2. WireGuard → Déchiffre et route vers FastAPI (10.8.0.1:8000)
3. FastAPI → Reçoit, valide, enrichit le contexte
4. FastAPI → Appelle vLLM via SDK OpenAI (localhost:8000/v1)
5. vLLM → Génère avec PagedAttention + Continuous Batching
6. vLLM → Retourne la réponse (streaming ou complète)
7. FastAPI → Post-traite, log, formate
8. FastAPI → Renvoie au client via tunnel WireGuard
9. Client (PyTools) → Reçoit, déchiffre, affiche
```

Cette architecture combine la performance de vLLM, la flexibilité de FastAPI, la sécurité de WireGuard, et l'abstraction de PyTools pour créer un système d'IA privé, performant et sécurisé.
