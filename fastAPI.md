# Guide FastAPI : Installation et Déploiement sur Ubuntu

FastAPI est un framework web moderne et rapide pour Python, basé sur les annotations de type standard. Il est particulièrement adapté à la création d'API performantes et génère automatiquement une documentation interactive. Ce guide vous montre comment l'installer et le déployer sur un serveur Ubuntu pour une utilisation en production.

## Prérequis

- Un serveur sous Ubuntu (20.04, 22.04 ou 24.04) avec un utilisateur non-root disposant des droits `sudo`
- Une connexion SSH active
- Un nom de domaine configuré (optionnel, pour Nginx)

## Étape 1 : Préparation du système

Connectez-vous à votre serveur et mettez à jour les paquets :

```bash
sudo apt update
sudo apt upgrade -y
```

Installez Python, pip et l'outil de création d'environnements virtuels :

```bash
sudo apt install -y python3 python3-pip python3-venv
```

Vérifiez que Python est bien installé :

```bash
python3 --version
```

## Étape 2 : Création de l'environnement virtuel

Créez un répertoire pour votre projet et placez-vous dedans :

```bash
mkdir ~/fastapi_app
cd ~/fastapi_app
```

Créez un environnement virtuel isolé :

```bash
python3 -m venv venv
```

Activez-le :

```bash
source venv/bin/activate
```

Votre prompt devrait maintenant afficher `(venv)` au début.

## Étape 3 : Installation de FastAPI

Installez FastAPI avec toutes les dépendances optionnelles (Uvicorn, Gunicorn, etc.) :

```bash
pip install fastapi[all] gunicorn
```

- `fastapi[all]` installe FastAPI avec Uvicorn (serveur ASGI) et d'autres outils utiles
- `gunicorn` est le gestionnaire de processus pour la production, qui utilisera des workers Uvicorn

## Étape 4 : Création d'une application minimale

Créez un fichier `main.py` :

```bash
nano main.py
```

Ajoutez le code suivant :

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
async def home():
    return {"message": "Hello World"}
```

Sauvegardez avec `Ctrl+O`, puis `Ctrl+X`.

## Étape 5 : Test rapide avec Uvicorn

Lancez temporairement Uvicorn pour vérifier que tout fonctionne :

```bash
uvicorn main:app --host 0.0.0.0 --port 8000
```

Ouvrez un autre terminal et testez :

```bash
curl http://localhost:8000
```

Vous devriez voir `{"message":"Hello World"}`. Arrêtez le serveur avec `Ctrl+C`.

## Étape 6 : Configuration de Gunicorn pour la production

Gunicorn gère plusieurs workers Uvicorn pour tirer parti de tous les cœurs CPU.

Créez un fichier de configuration `gunicorn_conf.py` :

```bash
nano gunicorn_conf.py
```

Contenu recommandé :

```python
import multiprocessing

# Adresse d'écoute (socket Unix pour Nginx)
bind = "unix:/home/VOTRE_UTILISATEUR/fastapi_app/gunicorn.sock"

# Nombre de workers (formule recommandée)
workers = multiprocessing.cpu_count() * 2 + 1

# Classe worker pour ASGI (FastAPI)
worker_class = "uvicorn.workers.UvicornWorker"

# Niveau de log
loglevel = "info"

# Fichiers de logs
accesslog = "/home/VOTRE_UTILISATEUR/fastapi_app/access.log"
errorlog = "/home/VOTRE_UTILISATEUR/fastapi_app/error.log"
```

Remplacez `VOTRE_UTILISATEUR` par votre nom d'utilisateur réel.

## Étape 7 : Création du service systemd

Pour que l'application démarre automatiquement au boot et redémarre en cas de crash, créez un service systemd :

```bash
sudo nano /etc/systemd/system/fastapi_app.service
```

Contenu :

```ini
[Unit]
Description=FastAPI Application
After=network.target

[Service]
User=VOTRE_UTILISATEUR
Group=www-data
WorkingDirectory=/home/VOTRE_UTILISATEUR/fastapi_app
Environment="PATH=/home/VOTRE_UTILISATEUR/fastapi_app/venv/bin"
ExecStart=/home/VOTRE_UTILISATEUR/fastapi_app/venv/bin/gunicorn -c gunicorn_conf.py main:app

[Install]
WantedBy=multi-user.target
```

Activez et démarrez le service :

```bash
sudo systemctl daemon-reload
sudo systemctl start fastapi_app
sudo systemctl enable fastapi_app
```

Vérifiez le statut :

```bash
sudo systemctl status fastapi_app
```

## Étape 8 : Configuration de Nginx (reverse proxy)

Nginx servira d'interface publique et transmettra les requêtes à Gunicorn.

Installez Nginx :

```bash
sudo apt install -y nginx
```

Créez une configuration pour votre site :

```bash
sudo nano /etc/nginx/sites-available/fastapi_app
```

Contenu (remplacez `votre-domaine.com`) :

```nginx
server {
    listen 80;
    server_name votre-domaine.com;

    location / {
        proxy_pass http://unix:/home/VOTRE_UTILISATEUR/fastapi_app/gunicorn.sock;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Activez le site :

```bash
sudo ln -s /etc/nginx/sites-available/fastapi_app /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

## Étape 9 : Vérification

Testez l'accès à votre application :

```bash
curl http://votre-domaine.com
```

Vous devriez obtenir `{"message":"Hello World"}`.

Accédez également à la documentation interactive générée automatiquement :
- **Swagger UI** : `http://votre-domaine.com/docs`
- **ReDoc** : `http://votre-domaine.com/redoc`

## Bonnes pratiques

- **Nombre de workers** : La formule `(2 × CPU) + 1` est un bon point de départ. Pour les charges CPU-intensives, réduisez à un worker par CPU.
- **Sécurité** : En production, envisagez de désactiver ou protéger `/docs` et `/redoc` avec une authentification basique.
- **Opérations bloquantes** : Si vous utilisez des pilotes de base de données synchrones, exécutez-les dans un thread pool avec `run_in_executor` pour ne pas bloquer la boucle asynchrone.
- **HTTPS** : Utilisez Certbot pour obtenir un certificat SSL gratuit :
  ```bash
  sudo apt install -y certbot python3-certbot-nginx
  sudo certbot --nginx -d votre-domaine.com
  ```

## Résumé de l'architecture

```
Internet → Nginx (reverse proxy) → Gunicorn (gestionnaire) → Workers Uvicorn (ASGI) → FastAPI
```

Cette architecture permet de gérer efficacement les connexions simultanées tout en gardant le processus robuste et redémarrable automatiquement.
