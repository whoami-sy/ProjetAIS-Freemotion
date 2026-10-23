# Perspectives d'évolution

[← Retour au projet](../README.md)

[← Supervision](6-supervision.md) 

## Introduction

Le périmètre déployé couvre le socle réseau, l'identité, les services, la DMZ et la supervision. Plusieurs briques prévues dans l'architecture cible n'ont pas été mises en œuvre, faute de temps et de ressources sur l'hyperviseur mutualisé, dont le quota mémoire imposait de prioriser les services cœur.

Cette page distingue trois types de perspectives :

- les **briques de l'architecture cible** conçues mais non déployées ;
- les **mesures de durcissement** qui prolongent la sécurisation existante ;
- les **améliorations identifiées pendant les tests** du périmètre déployé.

---

## Briques de l'architecture cible non déployées

| Brique                         | Rôle prévu                                                                                                                         | Prérequis                                      |
|--------------------------------|------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------|
| **Sauvegarde Veeam (3-2-1)**   | Sauvegarde régulière des contrôleurs de domaine, du serveur de fichiers et du serveur RDS vers un stockage local, puis une copie externalisée, avec une politique de rétention définie | Stockage secondaire local et externalisé |
| **Volet cloud**                | Split-DNS pour les services exposés et stockage objet destiné à la copie externalisée des sauvegardes                             | Partenariat cloud ou nom de domaine public     |
| **GLPI**                       | Gestion de parc (inventaire, tickets) sur une machine dédiée                                                                       | Aucun, brique autonome                         |
| **Grafana**                    | Tableaux de bord consolidés croisant les métriques Zabbix et les alertes Wazuh sur une interface unique                            | API Zabbix et API de l'indexeur Wazuh          |
| **Microsoft 365, WSUS, déploiement automatisé des postes** | Messagerie, gestion des mises à jour et industrialisation du poste de travail                        | Licences et volet cloud                        |

La sauvegarde constitue la lacune la plus importante du périmètre actuel. Les mécanismes de haute disponibilité protègent contre la panne d'un équipement, mais pas contre la suppression, la corruption ou le chiffrement des données par un rançongiciel. La sauvegarde devrait en outre s'accompagner de **tests de restauration périodiques**, seuls à même de prouver l'intégrité des sauvegardes, au-delà de la simple réussite des tâches planifiées.

---

## Durcissement

| Mesure                                    | Objectif                                                                                                       | Prérequis                                 |
|-------------------------------------------|----------------------------------------------------------------------------------------------------------------|-------------------------------------------|
| **Windows LAPS**                          | Rotation automatique et gestion centralisée du mot de passe administrateur local des postes et serveurs       | Extension du schéma AD                    |
| **Modèle d'administration par tiers**     | Cloisonnement des comptes d'administration par niveau (annuaire, serveurs, postes), afin qu'un compte compromis sur un poste ne donne pas accès aux contrôleurs de domaine | Révision des groupes et des GPO           |
| **Bastions d'accès (Guacamole)**          | Point d'accès unique et journalisé aux serveurs de la DMZ et aux consoles d'administration, sans accès d'administration exposé directement | Règle pfSense dédiée MGMT → DMZ           |
| **MFA sur les bastions**                  | Authentification multi-facteur, cohérente avec le principe d'accès nominatif déjà appliqué                     | Bastions déployés au préalable            |
| **VPN client-to-site**                    | Accès distant des administrateurs et des utilisateurs nomades, avec authentification multi-facteur            | Profil VPN dédié sur pfSense, distinct du tunnel inter-site |
| **Scan de vulnérabilités (Nmap, Nessus)** | Recherche de vulnérabilités sur l'infrastructure et la DMZ, en complément de l'audit PingCastle de l'annuaire | Aucun                                     |
| **PKI Windows (AD CS)**                   | Remplacement de la CA pfSense par une PKI avec liste de révocation et inscription automatique des certificats | Serveur dédié, idéalement avec CA racine hors ligne |

---

## Améliorations identifiées pendant les tests

Les tests du périmètre déployé ont mis en évidence plusieurs points d'amélioration, documentés dans les pages correspondantes.

| Constat                                                                                                  | Amélioration                                                                                    | Page                            |
|----------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|---------------------------------|
| Le nœud pfSense BACKUP de Lyon apparaît indisponible dans Zabbix, faute de porter la VIP du tunnel        | Dépendance de déclencheur fondée sur le rôle CARP du nœud actif, pour supprimer l'alerte attendue | [Supervision](6-supervision.md) |
| PingCastle signale l'usage récent du compte Administrateur intégré (+20 points)                          | Désactivation ou renommage du compte, administration exclusivement par comptes nominatifs       | [Annuaire](2-annuaire.md)       |
| Les annonces VRRP de Keepalived sont authentifiées par un mot de passe transmis en clair                 | Annonces VRRP en unicast entre les deux nœuds HAProxy                                           | [DMZ](5-dmz.md)                 |
| Les certificats émis par `Freemotion_Interne_CA` et `CA_VPN` n'ont pas de suivi d'expiration             | Suivi et alerte avant expiration, ou renouvellement automatisé                                  | [RDS](4-rds.md)                 |
| La détection Suricata et le filtrage pfBlockerNG sont configurés mais pas encore éprouvés par un test dédié | Scénarios de test : scan Nmap pour Suricata, requête vers un domaine listé pour le DNSBL        | [Réseau](1-reseau.md)           |
| Les alertes Zabbix et Wazuh restent consultées séparément                                                | Création automatique de tickets GLPI à partir des déclencheurs                                  | [Supervision](6-supervision.md) |

---

## Priorisation

| Priorité | Perspectives                                               | Justification                                                     |
|----------|------------------------------------------------------------|-------------------------------------------------------------------|
| **1**    | Sauvegarde Veeam et tests de restauration                  | Seule protection contre la perte ou le chiffrement des données    |
| **2**    | Windows LAPS, compte Administrateur intégré, scan de vulnérabilités | Mesures rapides à mettre en œuvre, à fort impact sur la sécurité  |
| **3**    | Bastions avec MFA, modèle d'administration par tiers       | Réduction de la surface d'attaque des accès d'administration      |
| **4**    | GLPI, Grafana, intégration des alertes                     | Amélioration de l'exploitation au quotidien                       |
| **5**    | Volet cloud, Microsoft 365, PKI AD CS                      | Chantiers plus lourds, dépendants de ressources externes          |

---

## Conclusion

Le périmètre déployé constitue un socle complet et éprouvé, mais il ne couvre pas l'ensemble de l'architecture cible. La sauvegarde en reste la principale lacune, et elle doit être traitée en priorité. Les mesures de durcissement prolongent ensuite la démarche engagée sur l'annuaire et la DMZ. Enfin, les améliorations issues des tests montrent que chaque brique validée a aussi permis d'identifier ses propres limites.

---

[← Supervision](6-supervision.md) · [Sommaire](../README.md)
