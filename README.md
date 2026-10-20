# Freemotion - Refonte d'un système d'information multisite

> Projet annuel du Bachelor AIS · 2025 - 2026 · Projet scolaire · Déployé

![pfSense](https://img.shields.io/badge/pfSense-212121?style=flat) ![Active Directory](https://img.shields.io/badge/Active_Directory-0078D4?style=flat) ![Windows Server](https://img.shields.io/badge/Windows_Server-0078D4?style=flat) ![Nginx](https://img.shields.io/badge/Nginx-009639?style=flat) ![HAProxy](https://img.shields.io/badge/HAProxy-106DA9?style=flat) ![Zabbix](https://img.shields.io/badge/Zabbix-D40000?style=flat) ![Wazuh](https://img.shields.io/badge/Wazuh-3595F6?style=flat)

## Contexte

Freemotion est un loueur fictif de véhicules électriques de 50 collaborateurs, avec un siège à Paris (40 personnes) et une succursale qui ouvre à Lyon (10 personnes). Son système d'information repose sur un réseau plat, sans pare-feu dédié ni segmentation. L'authentification n'est pas centralisée, aucune supervision n'existe et rien ne relie les deux sites de façon sécurisée.

Dans le cadre du BAC+3 AIS, j'ai conçu, déployé et documenté seule la refonte complète de cette infrastructure, en justifiant chaque choix technique au regard des bonnes pratiques et des recommandations de l'ANSSI.

## Démarche

L'infrastructure a été construite par couches, chacune s'appuyant sur la précédente :

1. un **socle réseau segmenté et redondant** : deux clusters pfSense en haute disponibilité et un tunnel chiffré entre les sites ;
2. une **identité centralisée** sur les deux sites : Active Directory avec un contrôleur en lecture seule à Lyon ;
3. des **services** qui reposent sur cette identité : fichiers, bureaux à distance, adressage ;
4. une **zone exposée** isolée et hautement disponible ;
5. une **couche transverse de supervision et de détection**.

L'ensemble a été maquetté sur un hyperviseur Proxmox mutualisé, avec un quota de 32 à 40 Go de RAM, les deux sites étant simulés par des bridges dédiés. Le détail est donné dans la page [Architecture](docs/0-architecture.md).

## Réalisations

**Réseau et sécurité périmétrique.** 

Chaque site dispose de sa propre paire pfSense en CARP, avec synchronisation des états (pfSync) et de la configuration (XML-RPC) sur un lien dédié. 
Les deux sites sont reliés par un tunnel IPsec IKEv2 authentifié par certificats, qui porte deux phases 2 : l'une pour les LAN, l'autre restreinte aux flux de supervision. 
Suricata inspecte le trafic en coupure sur le WAN, et pfBlockerNG filtre en amont par origine géographique et par réputation DNS. 

> → [Réseau](docs/1-reseau.md)

**Identité.** 

Le domaine `freemotion.fr` s'appuie sur deux contrôleurs à Paris et un RODC à Lyon. La structure d'OU est créée par script, la stratégie de réplication des mots de passe est stricte et le DHCP est centralisé avec un relais vers Lyon. Un audit PingCastle a fait passer le score de risque de 60 à 30. 

> → [Annuaire](docs/2-annuaire.md)

**Services.** 

Les partages du serveur de fichiers suivent le modèle AGDLP. La page couvre aussi la redirection des dossiers des itinérants et FSRM (quotas, filtrage, classification, audit). Une publication RemoteApp est sécurisée par une autorité de certification interne distribuée par GPO. 

> → [Serveur de fichiers](docs/3-serveur-fichiers.md) · [RDS](docs/4-rds.md)

**DMZ.** 

Trois serveurs Nginx sont répartis en round-robin par une paire HAProxy rendue hautement disponible par Keepalived (VRRP). Des règles pfSense interdisent tout rebond vers le LAN et le réseau de management. 

> → [DMZ](docs/5-dmz.md)

**Supervision et détection.** 

Zabbix surveille la disponibilité de 16 hôtes, le service HAProxy et l'état du tunnel VPN. Wazuh collecte les journaux de 11 agents et la télémétrie des pare-feux par syslog, et sa réponse active a été validée sur une attaque par force brute SSH. 

> → [Supervision](docs/6-supervision.md)

## Résultats

Chaque brique a été validée par un test de bout en bout, preuve à l'appui :

| Domaine                     | Test | Résultat |
|-----------------------------|------------------------------------------------------|--------------------------------------------------------|
| Haute disponibilité pfSense | Bascule volontaire du Master, ping continu           | Aucune perte de paquet                                 |
| VPN inter-site              | Bascule CARP pendant le trafic                       | Tunnel rétabli automatiquement sur le nouveau Master   |
| Active Directory            | Audit PingCastle                                     | Score de risque passé de 60 à 30                       |
| Serveur de fichiers         | Accès par groupe, quota, fichier interdit            | Conforme au modèle, événements journalisés             |
| DMZ                         | Panne d'un serveur web, puis du reverse proxy Master | Service maintenu                                       |
| Supervision                 | Arrêt de HAProxy, coupure du VPN                     | Détection en quelques secondes, résolution automatique |
| Détection                   | Force brute SSH (Hydra)                              | Attaquant bloqué automatiquement                       |

## Documentation technique

| # | Page | Contenu |
|---|---------------------------------------------------|---|
| 0 | [Architecture](docs/0-architecture.md)            | Maquette Proxmox, zones, adressage |
| 1 | [Réseau](docs/1-reseau.md)                        | pfSense HA/CARP, IPsec IKEv2, Suricata, pfBlockerNG |
| 2 | [Annuaire](docs/2-annuaire.md)                    | AD multisite, RODC, PRP, DHCP, PingCastle |
| 3 | [Serveur de fichiers](docs/3-serveur-fichiers.md) | AGDLP, redirection, FSRM |
| 4 | [RDS](docs/4-rds.md)                              | RemoteApp, PKI interne |
| 5 | [DMZ](docs/5-dmz.md)                              | Nginx, HAProxy, Keepalived |
| 6 | [Supervision](docs/6-supervision.md)              | Zabbix, Wazuh, réponse active |
