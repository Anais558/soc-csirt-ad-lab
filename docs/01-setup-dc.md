# Étape 1 — Montage du contrôleur de domaine (DC)

## Objectif
Mettre en place un contrôleur de domaine Active Directory servant de cible pour les attaques simulées, avec un compte de service volontairement vulnérable au Kerberoasting.

## Environnement
- **Hyperviseur :** VMware Workstation Pro
- **OS :** Windows Server 2022 Standard Evaluation (Desktop Experience)
- **Réseau :** Host-only (isolé du réseau physique)
- **Domaine créé :** `labo.local`

## 1. Création de la VM

Configuration retenue :
- Disque : 40 Go (allocation dynamique / thin provisioned)
- RAM : ~5 Go
- CPU : 2 cœurs
- Réseau : Host-only

**Point d'attention :** VMware propose par défaut le mode "Easy Install", qui automatise l'installation via un fichier de réponse généré. Ce mode a provoqué une erreur bloquante (*"Windows ne trouve pas le Contrat de licence Microsoft"*) avec l'ISO Evaluation sans clé de produit.

**Solution appliquée :** recréation de la VM avec l'option **"I will install the operating system later"**, puis montage manuel de l'ISO dans les paramètres CD/DVD. L'installation s'est ensuite déroulée normalement en mode manuel (sélection d'édition, licence, partitionnement).


## 2. Installation de l'OS

- Édition choisie : **Windows Server 2022 Standard Evaluation (Desktop Experience)** — interface graphique conservée pour faciliter la manipulation d'AD
- Partitionnement : disque unique, installation personnalisée

## 3. Configuration réseau (IP statique)

Un contrôleur de domaine nécessite une IP fixe (DNS et AD en dépendent).

Paramètres appliqués sur la carte Ethernet0 :
- **Adresse IP :** 192.168.50.10
- **Masque de sous-réseau :** 255.255.255.0
- **Passerelle :** aucune (réseau isolé)
- **DNS préféré :** 127.0.0.1 (le DC sera son propre serveur DNS)

![Configuration IP statique](../screenshots/dc/02-network-config.png)

Vérification via `ipconfig` : IP correctement appliquée.

## 4. Installation du rôle AD DS

Ajout du rôle **Services AD DS** via le Gestionnaire de serveur (Ajouter des rôles et fonctionnalités).

![Installation AD DS](../screenshots/dc/03-ad-ds-install.png)

## 5. Promotion en contrôleur de domaine

- Création d'une nouvelle forêt : `labo.local`
- Niveau fonctionnel : par défaut
- Mot de passe DSRM défini
- Redémarrage automatique du serveur à l'issue de la configuration

![Promotion en DC](../screenshots/dc/04-domain-controller-promotion.png)

## 6. Création des comptes utilisateurs de test

4 comptes créés dans `Users` :
- `jdupont` (Jean Dupont)
- `msmith` (Marie Smith)
- `adupuis` (Alice Dupuis)
- `svc_sql` (compte de service — cible SPN)

Pour chaque compte : mot de passe défini, expiration désactivée, changement au premier login désactivé (simplification pour environnement de labo).

![Comptes créés](../screenshots/dc/05-users-created.png)

## 7. Configuration du SPN (Kerberoasting)

Le compte `svc_sql` reçoit un Service Principal Name, le rendant volontairement vulnérable à une attaque Kerberoasting (technique classique de compromission AD).

Commandes utilisées :
```powershell
setspn -A MSSQLSvc/labo.local:1433 labo\svc_sql
setspn -L svc_sql
```

Résultat : SPN `MSSQLSvc/labo.local:1433` correctement enregistré.

![SPN configuré](../screenshots/dc/06-spn-configured.png)

## Résultat de l'étape

Le DC est opérationnel : domaine `labo.local` fonctionnel, comptes de test créés, cible Kerberoasting prête pour la phase d'attaque (voir `02-setup-kali.md` et `05-attacks-and-detection.md`).

## Difficultés rencontrées

| Problème | Cause | Solution |
|---|---|---|
| Erreur "Contrat de licence introuvable" | VMware Easy Install incompatible avec ISO Evaluation sans clé | Installation manuelle via "I will install the operating system later" |
| Écran Boot Manager au démarrage | Pas d'ordre de boot par défaut sur le disque vide | Sélection manuelle de "EFI VMware Virtual SATA CDROM Drive" |