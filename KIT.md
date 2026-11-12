# README — Machine confidentielle sur disque externe (Kali + Tor)

> Guide pas à pas pour installer et configurer un système portable, chiffré et cloisonné sur un disque externe.
> À utiliser dans le respect de la loi : recherche en sécurité, protection de la vie privée, apprentissage.

---

## 0. Principes à garder en tête

1. **L'anonymat total n'existe pas.** On réduit les traces, on ne les supprime pas.
2. **Les erreurs de comportement trahissent plus souvent que les failles techniques** (même pseudo, compte perso ouvert une fois, fichier avec métadonnées).
3. **Kali n'est pas conçu pour l'anonymat** (pas amnésique, outils bruyants). On le durcit, ou on l'utilise derrière **Whonix** (section 8). Pour un anonymat plus solide et simple, **Tails** est souvent le meilleur choix (section 14).
4. **Une machine = une identité.** Ce disque ne sert qu'à l'usage confidentiel. Aucun compte, fichier ou habitude de ta vie normale dessus.

### Hypothèses de ce README
- Système : **Kali Linux** installé sur un **disque/SSD externe**, chiffré (LUKS).
- Réseau : tout le trafic sortant passe par **Tor**.
- Ordinateur hôte : un PC classique que tu peux démarrer depuis USB.

---

## 1. Matériel nécessaire

| Élément | Remarque |
|---|---|
| Disque/SSD externe (≥ 128 Go) | USB 3.0 minimum. **Il sera entièrement effacé.** |
| Clé USB (≥ 8 Go) | Pour l'installeur Kali (temporaire). |
| Deuxième support chiffré | Pour la sauvegarde de l'en-tête LUKS (section 13). |
| PC de démarrage | Idéalement sans données sensibles, avec le Wi-Fi/Bluetooth désactivables. |

---

## 2. Préparer l'installeur

### 2.1 Télécharger et **vérifier** l'ISO
Télécharge l'ISO « Installer » depuis le site officiel de Kali (kali.org), puis vérifie l'intégrité :

```bash
# Dans le dossier de téléchargement (adapte les noms de fichiers)
sha256sum -c SHA256SUMS --ignore-missing

# Vérification de la signature (clé officielle Kali)
curl -fsSL https://archive.kali.org/archive-key.asc | gpg --import
gpg --verify SHA256SUMS.gpg SHA256SUMS
```
Compare l'empreinte de la clé avec celle publiée dans la documentation officielle Kali.
Si la vérification échoue : **ne pas installer**, retélécharger.

### 2.2 Créer la clé USB bootable
```bash
lsblk                       # IDENTIFIE la clé USB (ex. /dev/sdb) — ne te trompe pas !
sudo dd if=kali-linux-XXXX-installer-amd64.iso of=/dev/sdX bs=4M status=progress conv=fsync
```
> ⚠️ Une erreur de périphérique dans `dd` efface le mauvais disque. Vérifie la taille avec `lsblk`.

---

## 3. Installer Kali sur le disque externe

### 3.1 Avant de démarrer
- **Débranche le disque interne** si c'est possible (le plus sûr), sinon repère-le précisément.
- Branche : clé d'installation + disque externe.
- Démarre sur la clé (touche de boot : F12 / F2 / Échap selon le PC). Choisis **Graphical install** (ou *Install*).

### 3.2 Choix à l'installation
| Étape | Choix recommandé |
|---|---|
| Langue / clavier | Langue neutre (ex. English) pour limiter le fingerprinting. Clavier selon ton besoin. |
| Fuseau horaire | **UTC** si proposé (sinon corriger après, section 5). |
| Nom de machine (hostname) | Générique : `host` (**pas** ton nom, pas le nom du PC). |
| Nom d'utilisateur | Générique : `user` (**pas** ton prénom). |
| Mot de passe utilisateur | Fort et unique. |
| Partitionnement | **Guided – use entire disk and set up encrypted LVM** |
| Disque cible | **Uniquement le disque externe.** Vérifie le nom et la taille. |
| Phrase de passe LUKS | Longue : **6 mots aléatoires minimum** (méthode diceware). Jamais réutilisée. |
| Logiciels | Environnement léger (Xfce) + utilitaires standards. **Évite les gros métapaquets** (`kali-linux-default`, etc.) : installe les outils au besoin. |
| GRUB | Installer sur le **disque externe** (pas sur le disque interne). En UEFI, active l'option *« Force GRUB installation to the EFI removable media path »* pour que le disque démarre sur d'autres PC. |

### 3.3 Premier démarrage
- Retire la clé d'installation.
- Démarre sur le disque externe, entre la phrase de passe LUKS, connecte-toi.

---

## 4. Durcissement du système (premier démarrage)

```bash
# Mises à jour
sudo apt update && sudo apt full-upgrade -y

# Désactiver les services bavards / inutiles
sudo systemctl disable --now bluetooth avahi-daemon cups cups-browsed 2>/dev/null

# Ne jamais utiliser la veille / hibernation (on éteint la machine)
sudo systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target
```

Bonnes pratiques :
- **Ne monte jamais** le disque interne du PC hôte (`lsblk` pour contrôler). Désactive le montage automatique dans le gestionnaire de fichiers.
- Désactive la webcam et le micro dans le BIOS/UEFI si possible.
- Mets un **mot de passe BIOS/UEFI** sur le PC hôte si c'est le tien.
- Éteins toujours proprement (`poweroff`), ne laisse pas la session ouverte sans surveillance.

---

## 5. Identifiants de la machine

### 5.1 Fuseau horaire et hostname
```bash
sudo timedatectl set-timezone UTC
sudo hostnamectl set-hostname host
sudo sed -i 's/^127.0.1.1.*/127.0.1.1\thost/' /etc/hosts
```

### 5.2 Adresse MAC aléatoire (Wi-Fi et Ethernet)
Crée le fichier `/etc/NetworkManager/conf.d/00-macrandomize.conf` :

```ini
[device]
wifi.scan-rand-mac-address=yes

[connection]
wifi.cloned-mac-address=random
ethernet.cloned-mac-address=random
connection.stable-id=${CONNECTION}/${BOOT}
```
```bash
sudo systemctl restart NetworkManager
```
Vérifie après connexion : `ip link` → l'adresse MAC doit différer de celle d'origine du matériel.

Ne pas envoyer le nom de machine au réseau (DHCP), pour chaque connexion enregistrée :
```bash
nmcli connection show
sudo nmcli connection modify "NOM_DE_LA_CONNEXION" ipv4.dhcp-send-hostname no
```

### 5.3 Désactiver IPv6 (source de fuites)
Crée `/etc/sysctl.d/99-privacy.conf` :
```
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1
```
```bash
sudo sysctl --system
```

---

## 6. Réseau : tout passer par Tor + « kill switch »

### 6.1 Installer et lancer Tor
```bash
sudo apt install -y tor torsocks nftables
sudo systemctl enable --now tor@default
```
Tor écoute en local sur `127.0.0.1:9050` (SOCKS).

### 6.2 Kill switch : bloquer tout trafic qui ne passe pas par Tor
Remplace le contenu de `/etc/nftables.conf` :

```nft
#!/usr/sbin/nft -f
flush ruleset

table inet filter {
  chain input {
    type filter hook input priority 0; policy drop;
    iif "lo" accept
    ct state established,related accept
  }
  chain forward {
    type filter hook forward priority 0; policy drop;
  }
  chain output {
    type filter hook output priority 0; policy drop;
    oif "lo" accept
    meta skuid "debian-tor" accept        # seul le service Tor peut sortir
    udp sport 68 udp dport 67 accept      # DHCP
    ct state established,related accept
  }
}
```
Teste d'abord, puis active au démarrage :
```bash
sudo nft -f /etc/nftables.conf
sudo systemctl enable --now nftables
```

Effet : une application qui « oublie » le proxy ne peut **pas** sortir. Les applications doivent utiliser le proxy SOCKS `127.0.0.1:9050` (ou `torsocks`).

### 6.3 Mises à jour via Tor
```bash
sudo apt -o Acquire::http::Proxy="socks5h://127.0.0.1:9050" \
         -o Acquire::https::Proxy="socks5h://127.0.0.1:9050" update
sudo apt -o Acquire::http::Proxy="socks5h://127.0.0.1:9050" \
         -o Acquire::https::Proxy="socks5h://127.0.0.1:9050" full-upgrade
```

### 6.4 Limites connues
- Les **portails captifs** (Wi-Fi d'hôtel, cafés) sont bloqués par le kill switch : c'est un point faible de cette méthode (Tails/Whonix gèrent mieux).
- Ton fournisseur d'accès voit que tu utilises Tor. Si Tor est bloqué, configure des **bridges obfs4** dans `/etc/tor/torrc`.
- Tor ne protège pas contre la corrélation de trafic par un adversaire qui voit les deux bouts.

---

## 7. Navigateur et applications

### 7.1 Tor Browser (le seul navigateur pour l'usage anonyme)
```bash
sudo apt install -y torbrowser-launcher
```
- Dans les paramètres du launcher, active **« Download over system Tor »**.
- Avec le kill switch, le Tor intégré à Tor Browser est bloqué ; lance-le en utilisant le Tor système :
  ```bash
  TOR_SKIP_LAUNCH=1 TOR_SOCKS_HOST=127.0.0.1 TOR_SOCKS_PORT=9050 torbrowser-launcher
  ```
- Niveau de sécurité : **Safer** ou **Safest**.
- **Ne pas** : redimensionner/maximiser la fenêtre, installer des extensions, changer les polices, activer le mode plein écran.
- Ne jamais ouvrir de documents téléchargés (PDF, Office) pendant que tu es en ligne.

### 7.2 Autres applications
- Passe-les par `torsocks <commande>` ou configure le proxy SOCKS5 `127.0.0.1:9050`.
- Évite le **BitTorrent** sur Tor (fuite d'IP + surcharge du réseau).
- Pas de plugins, pas de Flash/Java, pas d'applications qui appellent « à la maison » (télémétrie, synchronisation cloud).

---

## 8. Option fortement recommandée : Whonix en machine virtuelle

Principe : deux VM. **Whonix-Gateway** force tout le trafic dans Tor, **Whonix-Workstation** n'a accès qu'à la Gateway. Même si une application est compromise, elle ne connaît pas l'IP réelle ni le vrai matériel (la VM présente un matériel générique).

```bash
sudo apt install -y qemu-system libvirt-daemon-system virt-manager
sudo usermod -aG libvirt $USER      # puis se déconnecter / reconnecter
```
1. Télécharge **Whonix (version KVM)** depuis whonix.org et **vérifie la signature** comme pour Kali.
2. Importe les deux VM (Gateway puis Workstation) dans `virt-manager`.
3. Démarre toujours **la Gateway d'abord**, puis la Workstation.
4. Fais tes activités anonymes **dans la Workstation**.
5. Pour utiliser des outils Kali : une VM Kali branchée sur le réseau interne Whonix (suivre la documentation Whonix à jour).

Avantage : le fingerprinting matériel (marque, modèle, GPU) devient largement inutile pour identifier ton vrai PC.

---

## 9. Métadonnées et hygiène d'identité

### 9.1 Nettoyer les fichiers avant tout partage
```bash
sudo apt install -y mat2 libimage-exiftool-perl
mat2 --show fichier.jpg      # voir les métadonnées
mat2 fichier.jpg             # crée fichier.cleaned.jpg
exiftool fichier.jpg         # inspection détaillée (GPS, auteur, appareil, logiciel)
```

### 9.2 Cloisonnement
- Pseudos, e-mails, mots de passe, avatars **jamais** réutilisés depuis ta vie normale.
- Pas de connexion à un compte personnel depuis cette machine, même « une seule fois ».
- Style d'écriture : attention aux habitudes (formules, fautes récurrentes, horaires d'activité).
- Utilise un gestionnaire de mots de passe local chiffré (ex. KeePassXC) dont la base reste sur ce disque.
- Paiements : pas de moyen de paiement lié à ton identité.

---

## 10. Checklist de vérification (à faire après l'installation)

| Test | Commande / action | Résultat attendu |
|---|---|---|
| Disque chiffré | `lsblk -f` | Volume de type `crypto_LUKS` |
| MAC différente | `ip link` | Différente de l'adresse matérielle réelle |
| Pas d'IPv6 | `ip -6 addr` | Aucune adresse globale |
| Hostname générique | `hostnamectl` | `host`, fuseau UTC |
| Tor actif | `systemctl status tor@default` | `active (running)` |
| Passage par Tor | `torsocks curl -s https://check.torproject.org/api/ip` | `"IsTor":true` |
| Kill switch | Stopper Tor : `sudo systemctl stop tor@default` puis `curl -m 5 https://example.com` | **Échec** (aucun accès) ; relancer Tor ensuite |
| Pas de fuite DNS | `cat /etc/resolv.conf`, test sur dnsleaktest.com **dans Tor Browser** | Aucun résolveur de ton FAI |
| Fingerprinting | Tor Browser → coveryourtracks.eff.org | Empreinte « commune » aux utilisateurs Tor |
| Disque interne | `lsblk` | **Non monté** |

---

## 11. Routine d'utilisation

**Avant de commencer**
1. Brancher le disque externe, démarrer dessus.
2. Vérifier que Tor est actif et le kill switch en place (`sudo nft list ruleset`).
3. Lancer Tor Browser (ou la Workstation Whonix).

**Pendant**
- Une seule identité par session. Aucune connexion perso.
- Aucun fichier personnel importé.

**Après**
- Fermer les applications, éteindre avec `poweroff`.
- Garder le disque physiquement en sécurité (il est chiffré, mais ça ne remplace pas la prudence).

**Maintenance (chaque semaine environ)**
- Mises à jour du système (section 6.3), de Tor Browser et de Whonix.

---

## 12. Erreurs fatales à éviter

- ❌ Se connecter à un compte personnel (mail, réseaux sociaux, banque) depuis cette machine.
- ❌ Réutiliser un pseudo, un mot de passe ou une photo de ta vie normale.
- ❌ Envoyer un fichier sans avoir nettoyé ses métadonnées.
- ❌ Utiliser un autre navigateur que Tor Browser pour l'activité anonyme.
- ❌ Installer des extensions ou modifier Tor Browser.
- ❌ Monter le disque interne du PC hôte ou copier des fichiers dessus.
- ❌ Désactiver le kill switch « juste pour une minute ».
- ❌ Laisser la machine allumée et déverrouillée sans surveillance.
- ❌ Oublier la phrase de passe LUKS sans sauvegarde (données perdues définitivement).

---

## 13. Sauvegarde et récupération

```bash
# Sauvegarde de l'en-tête LUKS (ADAPTE /dev/sdX3 à la partition chiffrée : lsblk -f)
sudo cryptsetup luksHeaderBackup /dev/sdX3 --header-backup-file luks-header.img
```
- Stocke `luks-header.img` sur un **autre support chiffré**, **jamais** sur le même disque.
- Si l'en-tête LUKS est endommagé, cette sauvegarde est la seule façon de récupérer les données.
- Note la phrase de passe dans un endroit physique sûr, ou apprends-la par cœur.
- Pour supprimer rapidement l'accès aux données en cas de besoin : effacer l'en-tête LUKS rend le disque illisible (action irréversible).

---

## 14. Alternative simple : Tails sur clé USB

Si ton besoin est avant tout l'anonymat (pas les outils de pentest) :
1. Télécharge **Tails** depuis tails.net et vérifie-le.
2. Installe-le sur une clé USB (≥ 8 Go) avec l'assistant officiel.
3. Démarre dessus : tout passe par Tor, la MAC est randomisée, **rien n'est écrit sur le disque**, tout s'efface à l'extinction.
4. Optionnel : **Persistent Storage** chiffré pour garder quelques fichiers/paramètres.

Tails règle d'office la plupart des points de ce README (MAC, kill switch, DNS, fingerprinting).

---

## Récapitulatif rapide

1. Vérifier et préparer l'ISO → 2. Installer Kali **chiffré** sur le disque externe →
3. Durcir (services, hostname, UTC, IPv6 off, MAC random) →
4. Tor + kill switch nftables → 5. Tor Browser (ou Whonix en VM) →
6. Nettoyer les métadonnées, cloisonner les identités → 7. Passer la checklist →
8. Sauvegarder l'en-tête LUKS → 9. Respecter la routine et éviter les erreurs fatales.
