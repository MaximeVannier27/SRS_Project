# Matrice de flux : Régie des Eaux de la Vallée du Cern

Réseau `192.168.102.0/24` · jump `.10` · scada-a `.20` · historian `.21` · pupitre `.22` · scada-b `.24` · wazuh `.30`

## 1. Flux autorisés

| # | Source | Destination | Port / proto | Justification | Preuve |
|---|---|---|---|---|---|
| 1 | Internet | jump | TCP/22 (exposé en 2202) | Point d'entrée unique d'administration | `ss`, incident SSH 06/11 |
| 2 | jump | scada-a, scada-b, historian, pupitre, wazuh | TCP/22 | Administration SSH | `ss -tnp` |
| 3 | jump, scada-a, scada-b, historian, pupitre | wazuh | TCP/1514 | Remontée des journaux (agents) | `ss -tnp` sur wazuh |
| 4 | jump, scada-a, scada-b, historian, pupitre | wazuh | TCP/1515 | Enrôlement des agents. Ouvert en permanence, limité aux 5 IP des agents | `ss -tulpn` wazuh |
| 5 | Poste admin (via passerelle `.1`) | wazuh | TCP/443 | Dashboard Wazuh (exposé en 4402) | `ss -tulpn` wazuh |
| 6 | jump, scada-a, scada-b, pupitre, wazuh | historian | UDP/123 | Synchronisation horaire (historian = référence) | `chronyc clients`, `tcpdump` |
| 7 | historian | Internet (toutes adresses, **temporaire**) | UDP/123 | Obtention de l'heure de référence (aucune source interne). Provisoire : restriction aux IP des serveurs NTP à faire ultérieurement | `chronyc sources -n`, `tcpdump -i any -nn udp port 123` |
| 7 bis | historian | 9.9.9.9 (Quad9), 1.1.1.1 (Cloudflare) | UDP/53 (repli TCP/53) | **Temporaire** : résolution des noms des serveurs NTP. À supprimer quand les sources seront déclarées par IP | `resolvectl status`, `tcpdump -i any -nn port 53` |
| 8 | pupitre | historian | TCP/443 | Courbes (`GET /series`) | `courbes_access.log` |
| 9 | historian | scada-a, scada-b | TCP/443 | Collecte (`GET /registers`, toutes les 5 min) | `collecte.log` |
| 10 | pupitre | scada-a, scada-b | TCP/443 | `GET /registers`, consignes (`POST /setpoints`) | `passerelle_access.log`, `consignes.log` |


Hors périmètre:


| # | Source | Destination | Port / proto | Justification | Preuve |
|---|---|---|---|---|---|
| - | scada-a | 192.168.102.200 | TCP/502 | Modbus/TCP vers automates Hydrelec (hors périmètre : injoignables dans le lab, flux autorisé et surveillé) | `automates.log` |
| - | scada-b | 192.168.102.201 | TCP/4840 | OPC UA vers automates Aquatron (hors périmètre : injoignables dans le lab, flux autorisé et surveillé) | `automates.log` |

Tout le reste est refusé et journalisé. Sur chaque machine : loopback et connexions établies (`ESTABLISHED,RELATED`) autorisés.

**Note sur les flux 7 et 7 bis.** Historian est la référence horaire du réseau et se synchronise sur des serveurs NTP publics (directive `pool` ou `server` à confirmer, voir section 3), sans authentification : NTS n'est pas utilisé (`chronyc authdata`). Un relais NTP interne ou une antenne GPS n'étant pas envisageables en tant que prestataire, son accès Internet est réduit par iptables à UDP/123 et au DNS vers ses deux résolveurs publics. Dans un premier temps, UDP/123 est ouvert vers toutes les adresses, faute d'avoir encore figé les serveurs : c'est plus large que la cible, et ce port peut servir de canal de sortie en cas de compromission d'historian. Étape suivante (échéance à fixer) : choisir 4 serveurs stables déclarés par IP (`server`), restreindre UDP/123 à ces IP (iptables filtre des adresses et non des noms) et supprimer le flux 7 bis. Risques résiduels : NTP en UDP non authentifié (un serveur compromis ou usurpé pourrait décaler l'heure de tout le réseau et fausser la corrélation des journaux dans Wazuh) et adresses de serveurs susceptibles de changer (revue périodique nécessaire).

**Note sur le flux 4.** Le port d'enrôlement reste ouvert en permanence, pour permettre un ré-enrôlement sans modifier le pare-feu. Il est limité côté wazuh aux 5 IP des agents. Risque résiduel : une machine compromise parmi les 5 pourrait enrôler de faux agents. Mesure recommandée : mot de passe d'enrôlement (section `<auth>` de `ossec.conf`, option `use_password`).

## 2. Vue par machine (base des iptables)

| Machine | Entrées | Sorties |
|---|---|---|
| jump | TCP/22 depuis partout | TCP/22 vers les 5 autres ; TCP/1514 et 1515 vers wazuh ; UDP/123 vers historian |
| scada-a | TCP/22 depuis jump ; TCP/443 depuis historian, pupitre | TCP/1514 et 1515 vers wazuh ; UDP/123 vers historian ; TCP/502 vers `.200` |
| scada-b | TCP/22 depuis jump ; TCP/443 depuis historian, pupitre | TCP/1514 et 1515 vers wazuh ; UDP/123 vers historian ; TCP/4840 vers `.201` |
| historian | TCP/22 depuis jump ; TCP/443 depuis pupitre ; UDP/123 depuis jump, scada-a, scada-b, pupitre, wazuh | TCP/443 vers scada-a, scada-b ; TCP/1514 et 1515 vers wazuh ; UDP/123 vers Internet (temporaire) ; UDP/53 et TCP/53 vers 9.9.9.9 et 1.1.1.1 (temporaire) |
| pupitre | TCP/22 depuis jump | TCP/443 vers historian, scada-a, scada-b ; UDP/123 vers historian ; TCP/1514 et 1515 vers wazuh |
| wazuh | TCP/22 depuis jump ; TCP/1514 et 1515 depuis les 5 agents ; TCP/443 depuis le poste admin ; TCP/55000 depuis localhost seulement | UDP/123 vers historian |

## 3. Points à vérifier avant d'écrire les iptables

- [x] **NTS** : non utilisé (`chronyc authdata` à zéro). Pas de flux TCP/4460.
- [x] **DNS d'historian** : résolveurs publics 9.9.9.9 et 1.1.1.1 (`resolvectl status`). Flux 7 bis.
- [x] **Enrôlement (flux 4)** : TCP/1515 ouvert en permanence.
- [ ] **DNS des autres machines** : `resolvectl status | head -n 15` sur jump, scada-a, scada-b, pupitre, wazuh. Aucun flux DNS n'est autorisé pour elles : sans résolveur interne, leurs résolutions seraient coupées.
- [ ] **Directive NTP d'historian** : `sudo grep -rEi "^(server|pool)" /etc/chrony/ /etc/chrony.conf`. Avec `pool`, les adresses changent à chaque résolution.
- [ ] **Adresses des serveurs NTP** (étape suivante) : `chronyc sources -n`, puis choix de 4 serveurs stables et passage en `server IP iburst`.
- [ ] **Mot de passe d'enrôlement** : `sudo grep -A15 "<auth>" /var/ossec/etc/ossec.conf` sur wazuh (`use_password`).
- [ ] **IP source du dashboard** : `sudo ss -tn state established '( sport = :443 )'` sur wazuh, dashboard ouvert (sinon risque de s'enfermer dehors).
- [ ] **DHCP et IPv6 sur chaque machine** : `ip -4 a show eth0` (mention `dynamic` : prévoir UDP 67/68), `ip -6 a` (si IPv6 actif : ip6tables équivalent ou désactivation).

## 4. Écarts avec le schéma de séance 1

| Écart | Explication | Décision |
|---|---|---|
| NTP vers historian absent du schéma ; jump et wazuh en sont aussi clients | Flux technique, historian est la référence horaire | Autoriser |
| Flux jump et wazuh absents du schéma | Dispositif de la mission (administration, supervision) | Autoriser |
| Historian conserve une sortie UDP/123 vers Internet et utilise des DNS publics | Aucune source de temps interne ; relais NTP et antenne GPS impossibles dans le périmètre prestataire | Autoriser temporairement ; restreindre aux IP de serveurs NTP stables et supprimer le flux 7 bis à l'étape suivante (échéance à fixer) ; recommander au client un serveur NTP interne |
| Port d'enrôlement TCP/1515 ouvert en permanence | Ré-enrôlement possible sans modifier le pare-feu | Autoriser, limité aux 5 IP des agents ; recommander un mot de passe d'enrôlement |
| Automates `.200` et `.201` injoignables dans le lab | Réseau de terrain hors périmètre | Autoriser le flux, surveiller |
| Compte HTTPS partagé (`svc-hydrelec`, `svc-aquatron`) entre historian et pupitre | Choix du client, attribution impossible par compte | Surveiller (source IP) |
| `verify=False` dans `collecte.py` | Certificats auto-signés des passerelles | Recommander une CA interne |