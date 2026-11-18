| # | Source | Destination | Port | Rôle |
|---|---|---|---|---|
| 1 | Internet | jump | TCP/22 (ext. 2202) | Entrée d'administration |
| 2 | jump | 5 autres machines | TCP/22 | Administration SSH |
| 3 | 5 agents | wazuh | TCP/1514 | Remontée des journaux |
| 4 | 5 agents | wazuh | TCP/1515 | Enrôlement (ouvert en permanence) |
| 5 | Internet | wazuh | TCP/443 (ext. 4402) | Dashboard (authentifié) |
| 6 | jump, scada-a/b, pupitre, wazuh | historian | UDP/123 | Heure (historian = référence) |
| 7 | historian | Internet | UDP/123 | NTP externe (**temporaire**) |
| 7 bis | historian | 9.9.9.9, 1.1.1.1 | UDP/53 | DNS du NTP (**temporaire**) |
| 8 | pupitre | historian | TCP/443 | Courbes |
| 9 | historian | scada-a, scada-b | TCP/443 | Collecte toutes les 5 min |
| 10 | pupitre | scada-a, scada-b | TCP/443 | Registres et consignes |