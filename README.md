# Kit — Notes techniques

Notes de travail sur les tests d'intrusion et l'outillage associé.

## Contenu

- **[RAPPORT.md](RAPPORT.md)** — Rapport complet :
  - Sliver : définition, architecture, composants, ports
  - Ce que Sliver n'est pas (pas un scanner)
  - Usage en local sur un projet Laravel + microservices (lab Docker)
  - Pourquoi une URL de projet déployé ne peut pas être « fournie » à Sliver
  - Ce que Sliver reçoit réellement (paquet `Register`, tâches)
  - Chaîne complète d'un test, scanners par phase
  - Outils disponibles sur Kali vs Ubuntu, installation et équivalents Docker
  - Recon avec une IP seule : ports, outils, limites vhost/SNI
  - Vérification de l'environnement (gh, Go, Sliver)
  - Recommandations et cadre légal

## Résumé en trois points

1. **Sliver n'analyse pas une cible.** C'est un framework de post-exploitation : il reçoit des connexions d'implants déjà déployés, pas une URL à attaquer.
2. **Une IP seule donne du recon réseau**, pas de l'intrusion automatique. L'applicatif (Laravel : authz, IDOR, mass assignment) exige une URL et une session authentifiée.
3. **Les scanners ne remplacent pas le manuel.** ZAP/nuclei trouvent les failles génériques ; la logique métier et les contrôles d'accès se testent à la main.

## Cadre

Outils offensifs à n'utiliser que sur des systèmes possédés ou couverts par une autorisation écrite.
