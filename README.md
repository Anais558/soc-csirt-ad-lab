# SOC/CSIRT Lab — Active Directory Attack & Detection

## Objectif
Simuler un environnement SOC minimal pour comprendre et documenter le cycle 
complet : attaque → détection → investigation → réponse à incident.

## Architecture
[Lien vers docs/00-architecture.md + schéma réseau]

## Stack technique
- Windows Server 2022 (Contrôleur de domaine)
- Kali Linux (simulation d'attaques)
- Wazuh (SIEM / détection)
- Sysmon (logs enrichis)
- TheHive (gestion d'incidents) — optionnel

## Progression
- [x] Réseau isolé configuré
- [ ] DC Windows Server monté
- [ ] Kali configuré
- [ ] Wazuh installé
- [ ] Sysmon déployé
- [ ] Premier incident détecté et documenté

## Documentation détaillée
Voir le dossier `docs/` pour le pas-à-pas complet de chaque brique.

## Incidents traités
Voir `incident-reports/` pour les fiches d'investigation.
