# Matrice de flux : Régie des Eaux de la Vallée du Cern

Réseau `192.168.102.0/24` · jump `.10` · scada-a `.20` · historian `.21` · pupitre `.22` · scada-b `.24` · wazuh `.30`

## 1. Flux autorisés

| # | Source | Destination | Port / proto | Justification | Preuve |
|---|---|---|---|---|---|
| 1 | Internet | jump | TCP/22 (exposé en 2202) | Point d'entrée unique d'administration | `ss`, incident SSH 06/11 |
| 2 | jump | scada-a, scada-b, historian, pupitre, wazuh | TCP/22 | Administration SSH | `ss -tnp` |
| 3 | jump, scada-a, scada-b, historian, pupitre | wazuh | TCP/1514 | Remontée des journaux (agents) | `ss -tnp` sur wazuh |
| 4 | les 5 agents | wazuh | TCP/1515 | Enrôlement des agents (à restreindre) | `ss -tulpn` wazuh |
| 5 | Poste admin (via passerelle `.1`) | wazuh | TCP/443 | Dashboard Wazuh (exposé en 4402) | `ss -tulpn` wazuh |
| 6 | jump, scada-a, scada-b, pupitre, wazuh | historian | UDP/123 | Synchronisation horaire (historian = référence) | `chronyc clients`, tcpdump |
| 7 | pupitre | historian | TCP/443 | Courbes (`GET /series`) | `courbes_access.log` |
| 8 | historian | scada-a, scada-b | TCP/443 | Collecte (`GET /registers`, toutes les 5 min) | `collecte.log` |
| 9 | pupitre | scada-a, scada-b | TCP/443 | `GET /registers`, consignes (`POST /setpoints`) | `passerelle_access.log`, `consignes.log` |
| 10 | scada-a | 192.168.102.200 | TCP/502 | Modbus/TCP vers automates Hydrelec | `automates.log` |
| 11 | scada-b | 192.168.102.201 | TCP/4840 | OPC UA vers automates Aquatron | `automates.log` |

Tout le reste est refusé et journalisé. Sur chaque machine : loopback et connexions établies (`ESTABLISHED,RELATED`) autorisés.

## 2. Vue par machine (base des iptables)

| Machine | Entrées | Sorties |
|---|---|---|
| jump | TCP/22 depuis partout | TCP/22 vers les 5 autres ; TCP/1514 vers wazuh ; UDP/123 vers historian |
| scada-a | TCP/22 depuis jump ; TCP/443 depuis historian, pupitre | TCP/1514 vers wazuh ; UDP/123 vers historian ; TCP/502 vers `.200` |
| scada-b | TCP/22 depuis jump ; TCP/443 depuis historian, pupitre | TCP/1514 vers wazuh ; UDP/123 vers historian ; TCP/4840 vers `.201` |
| historian | TCP/22 depuis jump ; TCP/443 depuis pupitre ; UDP/123 depuis jump, scada-a, scada-b, pupitre, wazuh | TCP/443 vers scada-a, scada-b ; TCP/1514 vers wazuh |
| pupitre | TCP/22 depuis jump | TCP/443 vers historian, scada-a, scada-b ; UDP/123 vers historian ; TCP/1514 vers wazuh |
| wazuh | TCP/22 depuis jump ; TCP/1514 et 1515 depuis les 5 agents ; TCP/443 depuis le poste admin ; TCP/55000 depuis localhost seulement | UDP/123 vers historian |

## 3. Points à vérifier avant d'écrire les iptables

- [ ] **Source horaire de historian** : `chronyc sources -v` (sinon bloquer UDP/123 sortant le fait dériver).
- [ ] **DNS** : `resolvectl status | head -n 15` (savoir si un DNS externe est nécessaire).
- [ ] **IP source du dashboard** : `sudo ss -tn state established '( sport = :443 )'` sur wazuh, dashboard ouvert (sinon risque de s'enfermer dehors).

## 4. Écarts avec le schéma de séance 1

| Écart | Explication | Décision |
|---|---|---|
| NTP vers historian absent du schéma ; jump et wazuh en sont aussi clients | Flux technique, historian est la référence horaire | Autoriser |
| Flux jump et wazuh absents du schéma | Dispositif de la mission (administration, supervision) | Autoriser |
| Automates `.200` et `.201` injoignables dans le lab | Réseau de terrain hors périmètre | Autoriser le flux, surveiller |
| Compte HTTPS partagé (`svc-hydrelec`, `svc-aquatron`) entre historian et pupitre | Choix du client, attribution impossible par compte | Surveiller (source IP) |
| `verify=False` dans `collecte.py` | Certificats auto-signés des passerelles | Recommander une CA interne |
