# Étape 2 — Montage de la VM Kali Linux (attaquant)

## Objectif
Mettre en place une machine attaquante capable de communiquer avec le DC pour simuler des attaques Active Directory (Kerberoasting, etc.).

## Environnement
- **Hyperviseur :** VMware Workstation Pro
- **OS :** Kali Linux
- **Réseau :** Host-only, même VMnet que le DC

## 1. Tentatives d'installation depuis une ISO officielle

Plusieurs tentatives d'installation depuis une ISO Kali "Installer" ont échoué à l'étape **"Choisir et installer des logiciels"** (échec de configuration, cause probable : ISO corrompue ou incomplète lors du téléchargement).

**Point d'attention rencontré au passage :** lors de la configuration réseau de l'installateur, un avertissement "aucune route par défaut" est normal sur un réseau Host-only isolé — répondre "Oui, continuer sans route par défaut" est le bon choix pour ce contexte (l'ISO Installer complète contient tous les paquets nécessaires en local).

## 2. Changement de méthode : import d'une VM pré-construite (OVA)

Pour éviter les erreurs répétées d'installation, une VM Kali pré-construite au format **OVA** a été importée directement dans VMware (**File → Open**).

**Difficulté rencontrée :** une première tentative d'import (image OVA exportée depuis VirtualBox) a échoué avec une erreur `capacity mismatch for disk` — incohérence entre la taille de disque déclarée dans le fichier OVF et la taille réelle après conversion `.vdi` → `.vmdk`. Une seconde image OVA a été utilisée avec succès.

## 3. Problème d'affichage du curseur

Après import, le curseur de la souris n'apparaissait pas dans la fenêtre de la VM — symptôme classique de l'absence de VMware Tools (l'image importée avait été construite à l'origine pour VirtualBox, pas VMware).

## 4. Configuration réseau et connectivité avec le DC

**Difficulté principale rencontrée :** après configuration de Kali en réseau **Host-only**, la connectivité avec le DC échouait (`ping: Network is unreachable`) malgré une IP obtenue correctement en DHCP.

**Diagnostic :**
```bash
ip a
# → eth0 : 192.168.117.137/24 (DHCP)
```
Le DC était configuré manuellement avec une IP `192.168.50.10`, dans un sous-réseau différent de celui réellement utilisé par le VMnet Host-only de VMware (`192.168.117.0/24`). Bien que les deux VMs soient sur le même VMnet, elles ne pouvaient pas communiquer car elles n'étaient pas dans le même sous-réseau.

**Résolution :** l'IP statique du DC a été corrigée en `192.168.117.10` (voir `01-setup-dc.md`) pour correspondre au sous-réseau réel attribué par VMware.

## 5. Validation de la connectivité

Depuis Kali :
```bash
ping 192.168.117.10
```

Résultat : ping réussi, connectivité DC ↔ Kali établie.

![Ping réussi vers le DC](../screenshots/kali/01-ping-dc-success.png)

## Résultat de l'étape

Kali est opérationnel et peut communiquer avec le DC sur le réseau isolé `192.168.117.0/24`. L'environnement est prêt pour la phase d'attaque (Kerberoasting sur le compte `svc_sql`).

## Difficultés rencontrées — synthèse

| Problème | Cause | Solution |
|---|---|---|
| Échec installation depuis ISO | ISO probablement corrompue/incomplète | Import d'une VM pré-construite (OVA) |
| Erreur "capacity mismatch" à l'import OVA | Incohérence de taille disque lors de la conversion VirtualBox → VMware | Utilisation d'une image OVA alternative |
| Curseur invisible | VMware Tools absents (image d'origine VirtualBox) | Installation de `open-vm-tools` |
| Ping impossible vers le DC | DC et Kali sur des sous-réseaux différents malgré le même VMnet | Alignement de l'IP du DC sur le sous-réseau réel (`192.168.117.0/24`) |