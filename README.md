# Encapsuler SSH dans SSL/TLS avec stunnel sur Debian / Ubuntu

> Connexions SSH via le port 443 (SSL) à l'aide de Stunnel

- [Introduction](#introduction)
- [Configuration côté serveur](#configuration-côté-serveur-serveur-distant)
- [Configuration côté client](#configuration-côté-client-machine-locale)

> 📝 Vous pouvez également lire l'article détaillé sur mon blog (article en anglais 🇬🇧) :
[Tunnel SSH Connections Over SSL Using Stunnel](https://www.damian-freelance.com/blog/tunnelling-ssh-over-ssl-on-debian)

## Introduction

Certains réseaux (pays, entreprises, universités, Wi-Fi publics) bloquent les connexions SSH sortantes sur le port 22, tout en autorisant le trafic HTTPS sur le port 443.

**stunnel** permet d’encapsuler une connexion SSH dans une couche SSL/TLS, la rendant indiscernable du trafic HTTPS classique.

### Cas d'usage typiques
- Accéder à vos serveurs depuis un réseau restrictif.
- Maintenir l'accès SSH depuis un Wi-Fi public filtré.
- Contourner les limitations de certains FAI.
- Sécuriser davantage vos connexions SSH avec une couche SSL supplémentaire.

### Avantages
- Permet de contourner les firewalls qui bloquent le port SSH (22).
- Utilise le port 443 (HTTPS) généralement autorisé.
- Double chiffrement : SSL + SSH, pour une sécurité renforcée.
- Simple et rapide à mettre en place (sans changer votre configuration SSH existante).


### 🔄 Schéma du flux

```
┌─────────────────┐          ┌─────────────────┐
│  Client         │          │  Serveur        │
│  (localhost)    │          │  (VPS)          │
└────────┬────────┘          └────────┬────────┘
         │                            │
         │  1. SSH → localhost:2200   │
         │     ↓                      │
         │  stunnel client            │
         │     ↓                      │
         │  2. SSL/TLS → VPS:443 ────→│
         │                            │
         │                     stunnel serveur
         │                            ↓
         │                     3. SSH → localhost:22
         │                            │
         │  ←──────────────────────────
         │     Connexion établie
```



# Tunnel SSH via SSL avec stunnel — Tutoriel pour Linux

Ce tutoriel couvre l'installation et la configuration complète de stunnel4, côté serveur et côté client, sur des systèmes Linux Debian / Ubuntu.

## Configuration côté serveur (serveur distant)

### 1. Installer stunnel

```bash
sudo apt install stunnel4
```


### 2. Créer un certificat SSL

```bash
# Créer un certificat avec une clé RSA de 2048 bits et remplir les champs demandés.
cd ~
openssl genrsa -out stunnel.key 2048
openssl req -new -x509 -key stunnel.key -out stunnel.crt -days 3650

# Réponses suggérées :
# Country Name: FR
# Common Name: l'IP publique ou le nom de domaine de votre serveur (facultatif)

# Fusionner la clé et le certificat :
cat stunnel.crt stunnel.key > stunnel.pem
sudo mv stunnel.pem /etc/stunnel/stunnel.pem

# Sécuriser les permissions :
sudo chmod 600 /etc/stunnel/stunnel.pem
sudo chown root:root /etc/stunnel/stunnel.pem
```

Vous disposez maintenant du fichier : `/etc/stunnel/stunnel.pem`


### 3. Configurer stunnel pour tunneliser le port 443 (HTTPS) vers le port SSH (22)

Créer le fichier de configuration :

```bash
sudo nano /etc/stunnel/stunnel.conf
```

Y insérer ce contenu :

```bash
pid = /var/run/stunnel.pid
cert = /etc/stunnel/stunnel.pem

[ssh]
accept = 443
connect = 127.0.0.1:22
```

Explication :
`accept = 443` : stunnel écoute sur le port 443 (HTTPS).
`connect = 127.0.0.1:22` : dès qu'une connexion SSL est reçue, elle est redirigée vers le port SSH local (22).


### 4. Activer stunnel

Modifier le fichier :

```bash
sudo nano /etc/default/stunnel4
```

Ajouter/Modifier `ENABLED` à :

```bash
ENABLED=1
```

### 5. Démarrer le service stunnel

```bash
# Démarrer le service :
sudo systemctl start stunnel4.service

# L'activer au démarrage du serveur :
sudo systemctl enable stunnel4

# Vérifier son statut :
sudo systemctl status stunnel4.service
```


### 6.Vérifier que stunnel écoute bien sur le port 443

Utilisez la commande suivante :

```bash
sudo lsof -i :443
```

Elle doit retourner quelque chose comme :

```bash
COMMAND  PID USER FD   TYPE DEVICE SIZE/OFF NODE NAME
stunnel4 771 root 9u  IPv4   5542      0t0  TCP *:https (LISTEN)
```



## Configuration côté client (machine locale)

### 1. Installer stunnel

```bash
sudo apt install stunnel4
```


### 2. Créer la configuration client

```bash
# Si dossier `~/.config/stunnel` n'existe pas encore, faire :
mkdir ~/.config/stunnel

# Créer le fichier :
nano ~/.config/stunnel/ssh-client.conf
```

Y insérer ce contenu :

```bash
client = yes
foreground = yes

[ssh]
accept = 127.0.0.1:2200
connect = IP_PUBLIQUE_DU_SERVEUR:443
```

Important :
Remplacez `IP_PUBLIQUE_DU_SERVEUR` par l'adresse IP public de votre serveur.


### 3. Activer stunnel

Modifier le fichier :

```bash
sudo nano /etc/default/stunnel4
```

Ajouter/Modifier `ENABLED` à :

```bash
ENABLED=1
```


### 4. Lancer le tunnelSSL

Dans un terminal :

```bash
sudo stunnel ~/.config/stunnel/ssh-client.conf
```


### 5. Se connecter via SSH à travers le tunnel

Et dans un autre terminal :

```bash
ssh -p 2200 root@localhost
```

PS : Si sur votre serveur, l'authentification par mot de passe est désactivée, utilisez votre clé privée SSH (avec l'option `-i`) :

```bash
ssh -p 2200 -i /YOUR_PATH/.ssh/id_rsa_root root@localhost
```