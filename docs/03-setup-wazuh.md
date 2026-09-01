# Étape 3 — Installation de Wazuh (SIEM)

## Objectif
Mettre en place la brique de détection du labo : un SIEM capable d'ingérer les logs du DC et de révéler les traces de l'attaque Kerberoasting menée à l'étape précédente.

## Environnement
- **Hyperviseur :** VMware Workstation Pro
- **OS :** Ubuntu Server 24.04.4 LTS
- **Réseau :** NAT temporaire pendant l'installation (accès Internet requis), puis Host-only une fois Wazuh opérationnel
- **Wazuh version :** 4.9.2 (installation "all-in-one" : indexer + manager + dashboard)

## 1. Choix et préparation de la VM

Une première tentative a été faite sur **Ubuntu 26.04 LTS** (version la plus récente proposée par défaut sur le site Ubuntu au moment du téléchargement). L'installation de Wazuh a échoué de façon récurrente à l'étape du manager (`wazuh-keystore: No such file or directory`) — cause identifiée : Ubuntu 26.04 n'est pas encore dans la liste des systèmes officiellement supportés par Wazuh 4.9.2 (Ubuntu 16.04 à 24.04).

**Résolution :** réinstallation de la VM avec **Ubuntu Server 24.04.4 LTS**, version stable et officiellement supportée.

## 2. Accès réseau pendant l'installation

Contrairement au DC et à Kali, Wazuh nécessite un accès Internet pour télécharger ses paquets. La carte réseau a donc été temporairement basculée en **NAT** le temps de l'installation, avant d'être repassée en **Host-only** (même VMnet que le DC et Kali) une fois Wazuh opérationnel.

**Point d'attention :** l'IP avait été fixée en statique lors de l'installation initiale — un passage en NAT à chaud ne suffit pas à obtenir une nouvelle IP tant que la configuration netplan reste figée en `addresses: [...]`. Il a fallu éditer manuellement `/etc/netplan/00-installer-config.yaml` pour repasser en `dhcp4: true` avant que l'accès Internet fonctionne.

## 3. Espace disque — difficulté récurrente

L'installation de Wazuh (indexer + manager + dashboard) nécessite significativement plus d'espace disque que prévu initialement (le disque de 20-30 Go alloué au départ s'est révélé insuffisant, provoquant un échec systématique à l'étape du dashboard avec l'erreur `disk full`).

**Résolution :** le disque virtuel a été agrandi à **65 Go**, avec extension de la partition LVM sous-jacente :
```bash
sudo growpart /dev/sda 3
sudo pvresize /dev/sda3
sudo lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
sudo resize2fs /dev/ubuntu-vg/ubuntu-lv
```

## 4. Installations interrompues et nettoyage

Plusieurs tentatives d'installation ont été interrompues (coupure de courant de la machine hôte en cours d'installation), laissant le système dans un état partiellement installé avec des scripts de désinstallation eux-mêmes corrompus (`dpkg --purge` échouant avec `exit status 127` sur le script `prerm` de `wazuh-manager`).

**Résolution :** neutralisation manuelle des scripts de maintenance cassés avant de purger proprement :
```bash
sudo bash -c 'echo "#!/bin/bash" > /var/lib/dpkg/info/wazuh-manager.prerm'
sudo bash -c 'echo "exit 0" >> /var/lib/dpkg/info/wazuh-manager.prerm'
sudo dpkg --purge --force-all wazuh-manager
sudo rm -rf /var/ossec /etc/wazuh-indexer /var/lib/wazuh-indexer /etc/wazuh-dashboard /etc/filebeat
```

Un nettoyage trop poussé a également fait disparaître l'entrée du dépôt APT de Wazuh (`/etc/apt/sources.list.d/wazuh.list`), provoquant une erreur `Unable to locate package wazuh-indexer`. Le dépôt a été réenregistré manuellement :
```bash
curl -o wazuh-key.gpg https://packages.wazuh.com/key/GPG-KEY-WAZUH
sudo gpg --dearmor -o /usr/share/keyrings/wazuh.gpg wazuh-key.gpg
echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" | sudo tee /etc/apt/sources.list.d/wazuh.list
sudo apt update
```

## 5. Installation réussie

```bash
sudo bash wazuh-install.sh -a
```

Résultat : les trois composants (indexer, manager, dashboard) installés et démarrés avec succès. Identifiants du dashboard (`admin` + mot de passe généré) récupérés et sauvegardés en lieu sûr.

![Installation Wazuh réussie](../screenshots/wazuh/01-install-success.png)

## Résultat de l'étape

Wazuh est opérationnel, accessible via son dashboard web. Prochaine étape : ajouter le DC comme agent surveillé, puis rejouer l'attaque Kerberoasting pour valider la détection (voir `04-setup-sysmon.md` et `05-attacks-and-detection.md`).

## Difficultés rencontrées — synthèse

| Problème | Cause | Solution |
|---|---|---|
| Échec récurrent à l'étape manager | Ubuntu 26.04 non supporté par Wazuh 4.9.2 | Réinstallation sur Ubuntu 24.04 LTS |
| Pas d'accès Internet après passage en NAT | IP statique figée dans la config netplan | Édition manuelle du fichier netplan vers `dhcp4: true` |
| Échec du dashboard — "disk full" | Disque virtuel trop petit (20-30 Go) pour les 3 composants Wazuh | Agrandissement du disque à 65 Go + extension LVM |
| `dpkg --purge` échoue (exit 127) | Coupure de courant ayant corrompu le script `prerm` du package | Neutralisation manuelle du script avant purge forcée |
| `Unable to locate package wazuh-indexer` | Dépôt APT Wazuh supprimé lors d'un nettoyage trop large | Réenregistrement manuel de la clé GPG et du dépôt APT |