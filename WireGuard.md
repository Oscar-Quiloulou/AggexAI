# Guide WireGuard : Installation et Déploiement Gratuit sur Ubuntu

WireGuard est un protocole VPN moderne, open-source et gratuit, intégré directement dans le noyau Linux depuis la version 5.6 . Il utilise une cryptographie de pointe et offre des performances supérieures à OpenVPN ou IPsec. Ce guide explique comment l'installer et le configurer sur un serveur Ubuntu, sans frais de licence.

## Prérequis

- Un serveur Ubuntu (VPS, cloud ou machine dédiée) avec un utilisateur non-root disposant des droits `sudo`
- Une adresse IP publique ou un nom de domaine
- Le port UDP 51820 ouvert sur le pare-feu (par défaut)

> **Note sur le "gratuit"** : WireGuard lui-même est entièrement gratuit et open-source. Les seuls coûts éventuels proviennent de l'hébergement (VPS). Des options gratuites existent : offres "Always Free" d'Oracle Cloud, crédits d'essai AWS/GCP, ou serveur personnel .

## Étape 1 : Installation de WireGuard

Connectez-vous à votre serveur et mettez à jour les paquets :

```bash
sudo apt update
sudo apt upgrade -y
```

Installez WireGuard depuis les dépôts officiels Ubuntu :

```bash
sudo apt install -y wireguard
```

Vérifiez l'installation :

```bash
wg --version
```

Vous devriez voir quelque chose comme `wireguard-tools v1.0.20210914` .

## Étape 2 : Génération des clés du serveur

WireGuard utilise des paires de clés cryptographiques (publique/privée) pour authentifier les pairs.

Générez la clé privée du serveur :

```bash
wg genkey | sudo tee /etc/wireguard/server_private.key
```

Restreignez les permissions du fichier :

```bash
sudo chmod 600 /etc/wireguard/server_private.key
```

Dérivez la clé publique à partir de la clé privée :

```bash
sudo cat /etc/wireguard/server_private.key | wg pubkey | sudo tee /etc/wireguard/server_public.key
```

**Notez bien la clé publique** : elle sera nécessaire pour configurer les clients .

## Étape 3 : Configuration du serveur WireGuard

Identifiez le nom de votre interface réseau principale :

```bash
ip -o -4 route show to default | awk '{print $5}'
```

Notez le résultat (souvent `eth0`, `ens3`, `enp1s0`, etc.).

Créez le fichier de configuration :

```bash
sudo nano /etc/wireguard/wg0.conf
```

Ajoutez la configuration suivante en remplaçant les valeurs :

```ini
[Interface]
Address = 10.8.0.1/24
SaveConfig = true
PrivateKey = VOTRE_CLE_PRIVE_SERVEUR
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o INTERFACE_RESEAU -j MASQUERADE
PreDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o INTERFACE_RESEAU -j MASQUERADE
ListenPort = 51820
```

Remplacez :
- `VOTRE_CLE_PRIVE_SERVEUR` par le contenu de `/etc/wireguard/server_private.key`
- `INTERFACE_RESEAU` par le nom de votre interface (ex: `eth0`)

**Explication des directives**  :
- `Address` : IP privée du serveur WireGuard
- `PostUp`/`PreDown` : règles NAT pour router le trafic des clients vers Internet
- `ListenPort` : port UDP d'écoute

## Étape 4 : Activation du routage IP

Pour que les clients puissent accéder à Internet via le VPN, le serveur doit transférer les paquets.

Activez le forwarding IPv4 :

```bash
echo "net.ipv4.ip_forward = 1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

Vérifiez :

```bash
cat /proc/sys/net/ipv4/ip_forward
```

Le résultat doit être `1` .

## Étape 5 : Configuration du pare-feu

Autorisez le port WireGuard et SSH :

```bash
sudo ufw allow 22/tcp
sudo ufw allow 51820/udp
sudo ufw enable
```

Vérifiez les règles :

```bash
sudo ufw status verbose
```

Si vous utilisez un pare-feu cloud (AWS, GCP, Oracle), ouvrez également le port UDP 51820 dans la console de gestion .

## Étape 6 : Démarrage de WireGuard

Activez et démarrez l'interface :

```bash
sudo wg-quick up wg0
```

Activez le démarrage automatique au boot :

```bash
sudo systemctl enable wg-quick@wg0
```

Vérifiez l'état :

```bash
sudo wg show
```

Vous devriez voir l'interface `wg0` avec son port d'écoute et sa clé publique .

## Étape 7 : Création d'un client

### Générer les clés du client

Sur le serveur (ou localement), générez une paire de clés pour le client :

```bash
wg genkey | tee client1_private.key | wg pubkey > client1_public.key
sudo chmod 600 client1_private.key
```

### Créer la configuration client

Créez un fichier `client1.conf` :

```ini
[Interface]
PrivateKey = CLE_PRIVE_CLIENT
Address = 10.8.0.2/24
DNS = 1.1.1.1

[Peer]
PublicKey = CLE_PUBLIQUE_SERVEUR
Endpoint = IP_PUBLIQUE_SERVEUR:51820
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
```

Remplacez :
- `CLE_PRIVE_CLIENT` par le contenu de `client1_private.key`
- `CLE_PUBLIQUE_SERVEUR` par le contenu de `/etc/wireguard/server_public.key`
- `IP_PUBLIQUE_SERVEUR` par l'IP publique de votre serveur

### Ajouter le client sur le serveur

Ajoutez le bloc `[Peer]` dans `/etc/wireguard/wg0.conf` :

```ini
[Peer]
PublicKey = CLE_PUBLIQUE_CLIENT
AllowedIPs = 10.8.0.2/32
```

Remplacez `CLE_PUBLIQUE_CLIENT` par le contenu de `client1_public.key`.

Rechargez la configuration :

```bash
sudo wg-quick down wg0
sudo wg-quick up wg0
```

## Étape 8 : Alternative automatisée

Pour simplifier le processus, un script d'installation automatique est disponible  :

```bash
wget -O wireguard.sh https://get.vpnsetup.net/wg
sudo bash wireguard.sh --auto
```

Ce script gère automatiquement :
- L'installation de WireGuard
- La génération des clés
- La configuration du serveur et des clients
- La génération de QR codes pour les appareils mobiles
- La gestion des utilisateurs (ajout, suppression, liste)

## Vérification et test

Connectez un client en important le fichier `client1.conf` dans l'application WireGuard (disponible sur Windows, macOS, iOS, Android, Linux).

Une fois connecté, vérifiez que votre IP publique a changé :

```bash
curl ifconfig.me
```

L'IP affichée devrait être celle de votre serveur .

## Résumé de l'architecture

```
Client (téléphone/PC) → Tunnel WireGuard chiffré → Serveur Ubuntu → Internet
```

WireGuard utilise le "Cryptokey Routing" : chaque clé publique est associée à une ou plusieurs adresses IP autorisées. Le serveur route les paquets des clients vers Internet via NAT, tout en maintenant une connexion chiffrée de bout en bout .

## Bonnes pratiques

- **Sauvegardez vos clés** : Si vous perdez la clé privée du serveur, tous les clients devront être reconfigurés.
- **Utilisez un DNS fiable** : `1.1.1.1` (Cloudflare) ou `8.8.8.8` (Google) dans la config client.
- **PersistentKeepalive** : Indispensable si le client est derrière un NAT (25 secondes est une valeur standard) .
- **Multi-clients** : Ajoutez simplement un nouveau bloc `[Peer]` pour chaque appareil, avec une IP différente (`10.8.0.3`, `10.8.0.4`, etc.) .
- **Surveillance** : Vérifiez régulièrement l'état avec `sudo wg show` pour voir les pairs connectés.
