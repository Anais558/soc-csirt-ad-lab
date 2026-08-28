# Incident 01 — Kerberoasting sur le compte de service svc_sql

## Résumé
Extraction et tentative de compromission du hash Kerberos du compte de service `svc_sql`, exploitant la faille structurelle du protocole Kerberos qui permet à tout utilisateur authentifié de demander un ticket de service (TGS) pour n'importe quel compte possédant un SPN.

## Contexte
- **Domaine :** labo.local
- **Compte attaquant utilisé :** jdupont (utilisateur standard, sans privilèges)
- **Compte ciblé :** svc_sql (SPN : `MSSQLSvc/labo.local:1433`)
- **Machine attaquante :** Kali Linux (192.168.117.137)
- **Cible :** DC Windows Server 2022 (192.168.117.10)
- **Outils utilisés :** Impacket (`GetUserSPNs.py`), John the Ripper

## Explication de la vulnérabilité

Kerberos permet à tout compte authentifié du domaine de demander un ticket de service (TGS) pour n'importe quel compte disposant d'un SPN — c'est un comportement normal et voulu du protocole, pas une faille de configuration. Le TGS retourné est chiffré avec le hash du mot de passe du compte de service ciblé.

Un attaquant peut donc extraire ce ticket chiffré sans privilèges particuliers, puis tenter de le casser hors ligne (attaque par dictionnaire ou brute force), sans jamais générer de tentative de connexion échouée détectable par les mécanismes classiques (verrouillage de compte, alertes d'échec d'authentification).

## Déroulement de l'attaque

### 1. Extraction du hash Kerberos

Commande utilisée depuis Kali :
```bash
impacket-GetUserSPNs labo.local/jdupont:'<password>' -dc-ip 192.168.117.10 -request -outputfile svc_sql.hash
```

Résultat : ticket TGS du compte `svc_sql` récupéré avec succès (format `$krb5tgs$23$...`), sans qu'aucune authentification échouée ne soit générée sur le DC.

**Difficulté technique rencontrée :** plusieurs échecs initiaux dus à un désaccord d'horloge (`KRB_AP_ERR_SKEW`) entre Kali et le DC — Kerberos exige que les horloges des machines communicantes soient synchronisées à quelques minutes près. Résolu en synchronisant manuellement l'heure UTC de Kali sur celle du DC.

### 2. Tentative de crack — mot de passe fort

Mot de passe initial du compte `svc_sql` : complexe (majuscule, chiffre, caractère spécial).

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt svc_sql.hash
```

**Résultat :** échec — 744 356 mots de passe testés (liste rockyou.txt), aucune correspondance trouvée. Session terminée en 19 secondes sans résultat.

### 3. Tentative de crack — mot de passe faible

Le mot de passe du compte `svc_sql` a ensuite été volontairement réinitialisé à `Password123` pour démontrer l'impact d'une politique de mot de passe faible.

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt svc_sql_weak.hash
```

**Résultat :** mot de passe trouvé **instantanément** (0 seconde), confirmé via `john --show`.

## Tableau comparatif

| Scénario | Mot de passe | Résultat du crack | Temps |
|---|---|---|---|
| Mot de passe fort | Complexe (maj/min/chiffre/symbole) | Non craqué | 19s pour tester 744 356 combinaisons |
| Mot de passe faible | `Password123` | Craqué | Instantané |

## Analyse et conclusion

Cette démonstration met en évidence deux points distincts :

1. **La vulnérabilité Kerberoasting est structurelle** : elle ne dépend pas d'une mauvaise configuration mais du fonctionnement même de Kerberos couplé à l'existence de comptes de service avec SPN. Elle ne peut pas être totalement éliminée, seulement atténuée.

2. **La robustesse du mot de passe est le facteur déterminant de l'impact réel** : un mot de passe fort rend l'attaque inefficace en pratique, même si l'extraction du hash réussit. Un mot de passe faible ou courant rend l'attaque triviale à exploiter.

## Recommandations de remédiation

- Imposer des mots de passe longs et complexes sur tous les comptes de service (idéalement générés aléatoirement, 25+ caractères)
- Utiliser des comptes de service gérés (**gMSA** — Group Managed Service Accounts), dont le mot de passe est automatiquement généré et changé par AD, rendant le Kerberoasting inefficace
- Mettre en place une rotation régulière des mots de passe de comptes de service
- Détecter les demandes anormales de TGS avec chiffrement RC4 (type 23), souvent signe d'une tentative de Kerberoasting (les services modernes utilisent AES) — voir `05-attacks-and-detection.md` pour la détection via Wazuh

## Détection (à compléter une fois Wazuh en place)

Voir `docs/05-attacks-and-detection.md` pour l'analyse des logs Windows correspondant à cette attaque (Event ID 4769 avec type de chiffrement RC4) et la règle de détection Wazuh associée.