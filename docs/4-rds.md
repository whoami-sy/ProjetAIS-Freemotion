# RDS : publication RemoteApp sécurisée

[← Retour au projet](../README.md)

[← Serveur de fichiers](3-serveur-fichiers.md) · [DMZ →](5-dmz.md)


## Introduction

Le serveur **PRS-RDS-01** (`10.9.75.21`) met à disposition des utilisateurs des applications publiées en **RemoteApp** : l'application s'exécute sur le serveur, mais s'affiche sur le poste client comme une application locale. Les utilisateurs y accèdent par le portail web **RDWeb**, à l'adresse `https://PRS-RDS-01.freemotion.fr/RDWeb`.

Publier une application ne suffit pas si la connexion n'est pas correctement sécurisée. Un certificat auto-signé génère des avertissements de sécurité sur chaque poste client : les utilisateurs prennent l'habitude de les ignorer, ce qui ouvre la voie à une interception de la connexion. La publication repose donc sur trois éléments :

- un **certificat TLS** signé par une autorité de certification interne, approuvée automatiquement par tous les postes du domaine ;
- un **contrôle d'accès** selon le modèle **AGDLP**, déjà appliqué aux autres ressources de l'infrastructure ;
- un **accès simplifié** par un raccourci déployé sur le bureau des seuls utilisateurs autorisés.

---

## Autorité de certification interne

### Une CA dédiée, distincte de celle du VPN

Le certificat TLS du serveur est signé par une autorité de certification interne créée sur pfSense, **`Freemotion_Interne_CA`**. Elle est distincte de **`CA_VPN`**, qui authentifie le [tunnel IPsec inter-site](1-reseau.md).

Ce cloisonnement des usages limite l'impact d'un incident : la compromission ou le renouvellement de l'une des deux autorités n'affecte pas la confiance établie par l'autre.

| Élément             | Valeur                     |
|---------------------|----------------------------|
| CA racine           | `Freemotion_Interne_CA`    |
| Certificat serveur  | `cert-prs-rds-01`          |
| Nom couvert         | `PRS-RDS-01.freemotion.fr` |
| Format d'export     | PKCS#12 (`.p12`)           |

![Création de la CA interne Freemotion_Interne_CA](images/rds/01-creation-ca-interne.png)

![Création du certificat serveur cert-prs-rds-01](images/rds/02-creation-certificat-serveur.png)

### Import sur le serveur RDS

Le certificat serveur, sa clé privée et le certificat de la CA sont exportés dans un fichier PKCS#12 unique, qui regroupe tout le nécessaire à l'import sous Windows. Ce fichier est importé dans le magasin de certificats de l'ordinateur local de `PRS-RDS-01`. La CA `Freemotion_Interne_CA` rejoint alors les autorités de certification racines de confiance de la machine.

![Certificat importé avec sa clé privée sur PRS-RDS-01](images/rds/03-certificat-importe.png)

### Distribution de la CA par GPO

Pour que l'ensemble du parc approuve le certificat sans avertissement, la CA racine est déployée sur tous les ordinateurs du domaine par la GPO **`Déploiement_CA_Interne`**, liée à l'OU `_Sites` :

`Configuration ordinateur > Paramètres de sécurité > Stratégies de clés publiques > Autorités de certification racines de confiance`

Ce mécanisme évite d'importer un certificat sur chaque poste : tout certificat signé par cette CA est désormais approuvé automatiquement. La console `certlm.msc` d'un poste client confirme la présence de `Freemotion_Interne_CA`, avec un nom et une date de validité corrects.

![GPO Déploiement_CA_Interne : import de la CA racine](images/rds/04-gpo-deploiement-ca.png)

---

## Liaison du certificat au déploiement RDS

### Portail RDWeb (IIS)

Le portail RDWeb repose sur IIS. La liaison HTTPS du site *Default Web Site*, sur le port `443`, est associée au nom d'hôte `PRS-RDS-01.freemotion.fr` et au certificat signé par la CA interne. Toute connexion au portail présente ainsi un certificat approuvé, et non un certificat auto-signé.

![Liaison du certificat TLS au portail RDWeb dans IIS](images/rds/05-liaison-iis.png)

### Rôles RDS

IIS ne couvre que le portail web. Le certificat est donc aussi associé explicitement à chaque rôle du déploiement, dans les propriétés de déploiement de la collection (onglet *Certificats*) :

| Rôle                                        | Flux protégé                                 |
|---------------------------------------------|----------------------------------------------|
| RD Connection Broker — *Enable Single Sign On* | Authentification unique entre les composants |
| RD Connection Broker — *Publishing*         | Signature des fichiers de connexion RemoteApp |
| RD Web Access                               | Portail RDWeb                                |

Sans cette étape, le portail serait approuvé mais le lancement de l'application afficherait encore un avertissement, signé par un certificat différent.

![Certificat associé à chaque rôle du déploiement RDS](images/rds/06-certificats-roles-rds.png)

---

## Publication des applications et contrôle d'accès

### Applications publiées

Deux applications sont publiées en RemoteApp pour valider le fonctionnement de la collection :

- **Informations système**, outil natif de Windows, qui sert de vérification simple ;
- **PDFsam**, application tierce de gestion de fichiers PDF installée manuellement sur le serveur, qui confirme que la publication fonctionne aussi pour une application métier.

### Contrôle d'accès (modèle AGDLP)

L'accès à la collection suit le modèle **AGDLP**, déjà appliqué à la [stratégie de réplication des mots de passe du RODC](2-annuaire.md) et aux [partages du serveur de fichiers](3-serveur-fichiers.md) :

| Niveau                  | Objet                          | Rôle                                                  |
|-------------------------|--------------------------------|-------------------------------------------------------|
| Groupe global           | `GG_Acces_RDP`                 | Utilisateurs métier autorisés à utiliser la RemoteApp |
| Groupe local de domaine | `GDL_RDS_QuickSession_Access`  | Contient les groupes globaux autorisés                |
| Permission              | Collection RDS                 | Accès accordé uniquement au GDL                       |

Ouvrir l'accès à un nouveau service consiste donc à ajouter son groupe global au GDL, sans toucher à la configuration de la collection.

![Gestion des accès à la collection RDS](images/rds/07-acces-agdlp.png)

---

## Raccourci RemoteApp déployé par GPO

Les utilisateurs n'ont pas à connaître l'URL du portail : un raccourci est déployé sur leur bureau par la GPO de préférences **`Raccourcis_RemoteApp_Bureau_Utilisateur`**.

| Paramètre            | Valeur                                                     |
|----------------------|------------------------------------------------------------|
| Emplacement          | `Configuration utilisateur > Préférences > Paramètres Windows > Raccourcis` |
| Nom du raccourci     | « Ressource Applicatif »                                   |
| Cible                | URL `https://PRS-RDS-01.freemotion.fr/RDWeb`               |
| Liaison              | OU `_Sites`                                                |
| Filtrage de sécurité | `GDL_RDS_QuickSession_Access` uniquement                   |
| Délégation           | *Utilisateurs authentifiés* en lecture seule               |

Le filtrage de sécurité réserve le raccourci aux utilisateurs autorisés. *Utilisateurs authentifiés* conserve le droit de lecture, nécessaire à l'évaluation de la GPO, sans que son application soit étendue à tout le domaine. La même logique est appliquée à la [GPO de redirection de dossiers](3-serveur-fichiers.md).

---

## Tests de validation

| Test                                         | Résultat                                                               |
|----------------------------------------------|------------------------------------------------------------------------|
| Présence de la CA racine sur un poste client | `Freemotion_Interne_CA` présente dans les autorités racines de confiance |
| Accès au portail RDWeb en HTTPS              | Aucun avertissement de certificat                                      |
| Connexion avec un utilisateur autorisé       | Ressources RemoteApp affichées et lancées                              |
| Déploiement du raccourci                     | « Ressource Applicatif » présent sur le bureau, portail accessible     |

**Portail RDWeb sans avertissement de certificat**

![Portail RDWeb ouvert en HTTPS sans avertissement](images/rds/08-test-portail-rdweb.png)

**Raccourci présent sur le bureau d'un utilisateur membre de `GG_Acces_RDP`**

![Raccourci « Ressource Applicatif » déployé et fonctionnel](images/rds/09-test-raccourci-gpo.png)

---

## Conclusion

La publication RemoteApp de `PRS-RDS-01` est sécurisée de bout en bout. Le certificat TLS, signé par la CA interne `Freemotion_Interne_CA`, est associé au portail IIS comme à chaque rôle du déploiement. La CA racine est distribuée automatiquement à tout le parc par GPO, ce qui supprime les avertissements de certificat sans manipulation sur les postes. L'accès est contrôlé par le modèle AGDLP, et un raccourci filtré par groupe simplifie l'usage pour les seuls utilisateurs autorisés.

Les tests confirment le fonctionnement complet : portail accessible en HTTPS sans avertissement, application lancée par un utilisateur autorisé et raccourci correctement déployé. L'autorité de certification reste toutefois portée par pfSense, sans liste de révocation publiée ni renouvellement automatique. Une PKI Windows (AD CS) avec inscription automatique des certificats constituerait l'évolution naturelle de ce dispositif.

---

[← Serveur de fichiers](3-serveur-fichiers.md) · [Sommaire](../README.md) · [DMZ →](5-dmz.md)
