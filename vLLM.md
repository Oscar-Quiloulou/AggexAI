# Guide vLLM : Installation et Utilisation sur Ubuntu

**vLLM** est une bibliothèque open-source conçue pour l'inférence et le service de grands modèles de langage (LLM) à haut débit et faible latence. Développée à l'origine à l'UC Berkeley, elle est aujourd'hui l'un des projets d'IA open-source les plus actifs, soutenu par une vaste communauté . Sa force réside dans des techniques comme **PagedAttention** et le **Continuous Batching**, qui permettent des débits jusqu'à 24 fois supérieurs aux approches standards .

Ce guide vous accompagne dans l'installation de vLLM sur un serveur Ubuntu, son utilisation en mode hors ligne (batch) et le déploiement d'un serveur compatible avec l'API OpenAI.

## Prérequis

Avant de commencer, assurez-vous que votre serveur Ubuntu respecte les conditions suivantes :

- **Système d'exploitation** : Linux (Ubuntu 22.04 ou 24.04 recommandé) .
- **Python** : Version 3.10 à 3.13 (3.12 recommandé) .
- **GPU (recommandé)** : Carte NVIDIA avec compute capability 7.0 ou supérieure (V100, T4, RTX 20xx, A100, H100, etc.) . vLLM supporte également les GPU AMD (ROCm), Intel (XPU) et les TPU Google .
- **CUDA** : Version 12.x ou 13.x pour les GPU NVIDIA .

## Étape 1 : Préparation de l'environnement Ubuntu

Connectez-vous à votre serveur et mettez à jour les paquets :

```bash
sudo apt update
sudo apt upgrade -y
```

Installez Python, pip et l'outil de création d'environnements virtuels :

```bash
sudo apt install -y python3 python3-pip python3-venv
```

Vérifiez la version de Python :

```bash
python3 --version
```

## Étape 2 : Installation de vLLM

vLLM recommande fortement l'utilisation de **`uv`**, un gestionnaire d'environnements Python très rapide, pour créer et gérer les environnements .

### Installation de `uv`

```bash
pip install uv
```

### Création de l'environnement virtuel

Créez un répertoire pour votre projet et placez-vous dedans :

```bash
mkdir ~/vllm_app
cd ~/vllm_app
```

Créez un environnement virtuel avec Python 3.12 :

```bash
uv venv --python 3.12 --seed
source .venv/bin/activate
```

### Installation de vLLM (NVIDIA CUDA)

Installez vLLM avec la détection automatique du backend CUDA :

```bash
uv pip install vllm --torch-backend=auto
```

Pour une version CUDA spécifique (par exemple CUDA 12.6) :

```bash
uv pip install vllm --torch-backend=cu126
```

> **Note sur l'installation** : L'option `--torch-backend=auto` permet à `uv` de sélectionner automatiquement l'index PyTorch approprié en inspectant la version de votre pilote CUDA . Pour les GPU AMD ROCm, utilisez l'index supplémentaire : `uv pip install vllm --extra-index-url https://wheels.vllm.ai/rocm/` .

### Vérification de l'installation

```bash
python -c "import vllm; print(vllm.__version__)"
```

## Étape 3 : Utilisation de vLLM en mode hors ligne (Batch Inference)

Avec vLLM installé, vous pouvez générer du texte pour une liste de prompts en une seule fois. C'est idéal pour le traitement par lots de grands volumes de données .

Créez un fichier `offline_inference.py` :

```python
from vllm import LLM, SamplingParams

# Définir les prompts
prompts = [
    "Bonjour, je m'appelle",
    "Le président des États-Unis est",
    "La capitale de la France est",
    "L'avenir de l'IA est",
]

# Configurer les paramètres d'échantillonnage
sampling_params = SamplingParams(temperature=0.8, top_p=0.95)

# Initialiser le moteur vLLM avec un petit modèle pour tester
llm = LLM(model="facebook/opt-125m")

# Générer les sorties
outputs = llm.generate(prompts, sampling_params)

# Afficher les résultats
for output in outputs:
    prompt = output.prompt
    generated_text = output.outputs[0].text
    print(f"Prompt: {prompt!r}, Generated text: {generated_text!r}")
```

Exécutez le script :

```bash
python offline_inference.py
```

**Explication**  :
- `LLM` : Classe principale pour l'inférence hors ligne.
- `SamplingParams` : Configure les paramètres de génération (température, top_p, etc.).
- Par défaut, vLLM utilise les paramètres recommandés par le créateur du modèle (via `generation_config.json`). Pour utiliser les paramètres par défaut de vLLM, ajoutez `generation_config="vllm"` lors de l'initialisation.
- vLLM télécharge automatiquement les modèles depuis Hugging Face.

## Étape 4 : Déploiement du serveur OpenAI-Compatible

vLLM peut être déployé comme un serveur HTTP implémentant l'API OpenAI. Cela en fait un remplacement direct pour les applications utilisant l'API OpenAI .

### Démarrage du serveur

Lancez le serveur avec un modèle (par exemple, un petit modèle pour tester) :

```bash
vllm serve facebook/opt-125m
```

Pour un modèle plus grand, configurez la mémoire GPU et la longueur maximale :

```bash
vllm serve meta-llama/Llama-3-8B-Instruct \
  --gpu-memory-utilization 0.9 \
  --max-model-len 8192 \
  --port 8000 \
  --host 0.0.0.0
```

### Test du serveur

Dans un autre terminal, testez le serveur avec `curl` :

```bash
curl http://localhost:8000/v1/models
```

Vous devriez voir la liste des modèles disponibles. Testez une complétion de chat :

```bash
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "facebook/opt-125m",
    "messages": [{"role": "user", "content": "Bonjour!"}]
  }'
```

### Utilisation avec le client OpenAI Python

Vous pouvez utiliser le SDK officiel OpenAI pour interagir avec le serveur vLLM  :

```python
from openai import OpenAI

client = OpenAI(
    base_url="http://localhost:8000/v1",
    api_key="EMPTY"  # vLLM n'exige pas de clé API
)

response = client.chat.completions.create(
    model="facebook/opt-125m",
    messages=[{"role": "user", "content": "Explique le calcul quantique."}]
)

print(response.choices[0].message.content)
```

## Étape 5 : Configuration avancée (Optionnel)

### Parallélisme tensoriel pour les grands modèles

Pour les modèles de 30B à 70B paramètres, utilisez le parallélisme tensoriel pour répartir le modèle sur plusieurs GPU  :

```bash
vllm serve meta-llama/Llama-2-70b-hf \
  --tensor-parallel-size 4 \
  --quantization awq \
  --port 8000
```

### Métriques et surveillance

Activez les métriques Prometheus pour la production  :

```bash
vllm serve meta-llama/Llama-3-8B-Instruct \
  --enable-prefix-caching \
  --enable-metrics \
  --metrics-port 9090 \
  --port 8000
```

Les métriques sont exposées sur le port 9090 :

```bash
curl http://localhost:9090/metrics | grep vllm
```

Métriques clés à surveiller  :
- `vllm:time_to_first_token_seconds` : Latence du premier token.
- `vllm:num_requests_running` : Requêtes actives.
- `vllm:gpu_cache_usage_perc` : Utilisation du cache KV.

### Déploiement avec Docker

vLLM fournit des images Docker pré-construites  :

```bash
docker run --gpus all \
    -v ~/.cache/huggingface:/root/.cache/huggingface \
    --env "HF_TOKEN=<votre_token_hf>" \
    -p 8000:8000 \
    --ipc=host \
    vllm/vllm-openai:latest \
    --model Qwen/Qwen2.5-1.5B-Instruct
```

## Résumé de l'architecture

```
Client (OpenAI SDK, curl) → Serveur vLLM (API OpenAI) → Moteur vLLM (PagedAttention, Continuous Batching) → GPU
```

vLLM utilise **PagedAttention** pour gérer efficacement la mémoire du cache KV et **Continuous Batching** pour maximiser l'utilisation du GPU, permettant un débit bien supérieur aux méthodes traditionnelles .

## Bonnes pratiques

- **Toujours utiliser `uv`** : Plus rapide et plus fiable que pip pour les dépendances complexes de vLLM .
- **Environnement virtuel dédié** : vLLM compile de nombreux noyaux CUDA, créant des incompatibilités binaires avec d'autres versions de CUDA ou PyTorch. Isolez toujours vLLM dans son propre environnement .
- **Chat templates** : Pour les modèles Instruct/Chat, assurez-vous que le tokenizer inclut un chat template. Sinon, spécifiez-le manuellement avec `--chat-template` .
- **Test de charge** : Avant la production, vérifiez les performances avec Locust ou un outil similaire. Ciblez un TTFT (Time To First Token) inférieur à 500 ms et une utilisation GPU supérieure à 80% .
- **Quantification** : Pour les modèles volumineux sur GPU limités, utilisez la quantification (AWQ, GPTQ, FP8) avec `--quantization awq` .

vLLM est un outil puissant qui transforme un serveur Ubuntu en une plateforme d'inférence LLM performante et compatible avec l'écosystème OpenAI.
