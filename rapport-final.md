# Rapport final — [Nom du groupe] — Secteur [hôpital / eau / labo IA]

> **Gabarit imposé, noté sur 12.** Respectez les sections et les limites. Un rapport long et générique ne rapporte rien.
> Appuyez chaque affirmation sur des **artefacts à vous** : captures horodatées,
> extraits de logs avec vos IP et vos hostnames, extraits de config.
>
> Chaque section indique la séance qui la nourrit (voir
> [`deroule-seances.md`](deroule-seances.md)).

**Membres du groupe :** Maxime VANNIER - Vu Trung HOANG
**Date de rendu :** 22/11/2026

---

## 1. Architecture *(≤ 1 page)* · séances 1 et 2

![Schéma d'architecture attendu (séance 1)](Schéma_Infra_Seance1.png)

*Le schéma de l'architecture **réelle**, telle que vous l'avez découverte en
séance 2.*

![Schéma d'architecture réelle (séance 2)](archi_reelle.png)


*Les écarts entre les deux (si constatés), et pour chacun **pourquoi** il existe (flux oublié,
flux technique, flux que le client ignorait, flux inexpliqué…).*

| Écart constaté | Explication | Décision (autoriser / refuser / surveiller) |
|---|---|---|
|Machines manquantes (Jump et Wazuh)| non-évoqués dans le dossier client | rajoutées dans la matrice de flux, à surveiller |
|Flux NTP pour horloges synchronisés | non-évoqués dans les logiques métiers du client | autorisés |
|Flux de *historian* vers Internet | visible seulement depuis la machine concernée, utilisé pour mettre à jour son horloge en tant que master | à surveiller/refuser si solution autre |
|Flux métiers bidirectionnels| ambiguité dans le dossier client | autorisés |

## 2. Matrice de flux et durcissement *(≤ 1 page)* · séances 2 et 3

| # | Source | Destination | Port / proto | Justification | Preuve |
|---|---|---|---|---|---|
| 1 | Internet | jump | TCP/22 (exposé en 2202) | Point d'entrée unique d'administration | `ss`, incident SSH 06/11 |
| 2 | jump | scada-a, scada-b, historian, pupitre, wazuh | TCP/22 | Administration SSH | `ss -tnp` |
| 3 | jump, scada-a, scada-b, historian, pupitre | wazuh | TCP/1514 | Remontée des journaux (agents) | `ss -tnp` sur wazuh |
| 4 | jump, scada-a, scada-b, historian, pupitre | wazuh | TCP/1515 | Enrôlement des agents. Ouvert en permanence, limité aux 5 IP des agents | `ss -tulpn` wazuh |
| 5 | Internet (toutes les sources) | wazuh | TCP/443 | Dashboard Wazuh (exposé en 4402), protégé par authentification | `ss -tulpn` wazuh, journaux `IN-DROP` (`DPT=443`) |
| 6 | jump, scada-a, scada-b, pupitre, wazuh | historian | UDP/123 | Synchronisation horaire (historian = référence) | `chronyc clients`, `tcpdump` |
| 7 | historian | Internet (toutes adresses, **temporaire**) | UDP/123 | Obtention de l'heure de référence (aucune source interne). | `chronyc sources -n`, `tcpdump -i any -nn udp port 123` |
| 7 bis | historian | 9.9.9.9 (Quad9), 1.1.1.1 (Cloudflare) | UDP/53 (repli TCP/53) | **Temporaire** : résolution des noms des serveurs NTP. À supprimer quand les sources seront déclarées par IP | `resolvectl status`, `tcpdump -i any -nn port 53` |
| 8 | pupitre | historian | TCP/443 | Courbes (`GET /series`) | `courbes_access.log` |
| 9 | historian | scada-a, scada-b | TCP/443 | Collecte (`GET /registers`, toutes les 5 min) | `collecte.log` |
| 10 | pupitre | scada-a, scada-b | TCP/443 | `GET /registers`, consignes (`POST /setpoints`) | `passerelle_access.log`, `consignes.log` |

*Note sur le flux n°7: configuration temporaire puisqu'il serait judicieux de trouver une solution de mise à jour d'horloge sans passer par internet (par exemple par GPS) mais ce n'est pas possible dans le cadre de ce travail, avec les accès que nous avions. Nous avons donc laissé l'accès aux serveurs distants avec une résolution DNS car les IP de ces derniers varient.*

**Durcissement**:

*Durcissement (séance 3) : comment vous avez traduit la matrice en iptables, et
comment vous avez vérifié, grâce à vos journaux, que le client fonctionnait
toujours.*

Nous avons éditer le fichier /etc/iptables/rules.v4 afin d'appliquer nos politiques de blocages et avons rédigé chaque règles en fonction de la matrice ci-dessus avec une politique par défaut de DROP. Ci-dessous, un exemple des règles pour la machine *historian*:
```
# Règles iptables (IPv4) de cette historian, rechargées au démarrage par netfilter-persistent.

*filter
:INPUT DROP [0:0]
:FORWARD DROP [0:0]
:OUTPUT DROP [0:0]

# --- Base : loopback, paquets invalides, connexions etablies ---
-A INPUT -i lo -j ACCEPT
-A OUTPUT -o lo -j ACCEPT
-A INPUT -m conntrack --ctstate INVALID -j DROP
-A OUTPUT -m conntrack --ctstate INVALID -j DROP
-A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
-A OUTPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# Flux 2 : administration depuis jump
-A INPUT -p tcp -s 192.168.102.10 --dport 22 -j ACCEPT

# Flux 8 : courbes depuis pupitre
-A INPUT -p tcp -s 192.168.102.22 --dport 443 -j ACCEPT

# Flux 6 : serveur horaire pour jump, scada-a, scada-b, pupitre, wazuh
-A INPUT -p udp -s 192.168.102.10,192.168.102.20,192.168.102.22,192.168.102.24,192.168.102.30 --dport 123 -j ACCEPT

# Flux 9 : collecte vers scada-a et scada-b
-A OUTPUT -p tcp -d 192.168.102.20,192.168.102.24 --dport 443 -j ACCEPT

# Flux 3 et 4 : journaux et enrolement vers wazuh
-A OUTPUT -p tcp -d 192.168.102.30 --dport 1514 -j ACCEPT
-A OUTPUT -p tcp -d 192.168.102.30 --dport 1515 -j ACCEPT

# Flux 7 : NTP vers Internet (TEMPORAIRE : a restreindre aux IP des serveurs NTP)
-A OUTPUT -p udp --dport 123 -m comment --comment "TEMP flux7 NTP Internet" -j ACCEPT

# Flux 7 bis : DNS vers 9.9.9.9 et 1.1.1.1 (TEMPORAIRE : a supprimer apres passage des sources en IP)
-A OUTPUT -p udp -d 9.9.9.9,1.1.1.1 --dport 53 -m comment --comment "TEMP flux7bis DNS" -j ACCEPT
-A OUTPUT -p tcp -d 9.9.9.9,1.1.1.1 --dport 53 -m comment --comment "TEMP flux7bis DNS" -j ACCEPT

# --- Journalisation des refus (avant la politique DROP) ---
-A INPUT -m limit --limit 5/min --limit-burst 10 -j LOG --log-prefix "IN-DROP: "
-A OUTPUT -m limit --limit 5/min --limit-burst 10 -j LOG --log-prefix "OUT-DROP: "
COMMIT
```

## 3. Supervision et détection *(≤ 1 page)* · séances 2 et 3

*Les sources de journaux collectées dans votre SIEM (séance 2) : quelles
machines, quels journaux, pour voir quoi.*

| Source (machine) | Journal collecté | Ce qu'il permet de voir |
|---|---|---|
| … | … | … |

*Ce qui est **normal** chez votre client, puis vos règles de détection (séance
3) : pour chacune, source, logique et preuve de déclenchement (capture
horodatée). Indiquez aussi les ajustements de collecte faits en cours de
route.*

**Règle 1 — …** *Source · logique · preuve.*  **Règle 2 — …**  **Règle 3 — …**

## 4. Post-mortems d'incident *(≤ 2 page)* · séances 2 à 4

*Un post-mortem par incident que vous avez subi. À vous de juger ce qui mérite
d'y figurer pour qu'il soit utile. Pensez à ce dont aurait besoin quelqu'un qui
n'était pas là : ce qui s'est passé et quand, comment vous l'avez vu (ou pas),
ce que vous en concluez et ce que vous changez ensuite. Si un incident vous a
échappé, dites-le et expliquez comment vous compteriez le voir la prochaine
fois. Appuyez-vous sur vos artefacts (logs, captures).*

**Incident — [séance, date] — …** *(un bloc par incident)*

## 5. Prise de recul métier *(≤ 2 pages, ouverture)*

*La partie la plus valorisée du rapport. Avec le recul : quelles contraintes
du client vous ont gênés ? Quels sont les défauts de son infrastructure ?
Qu'est-ce que votre dispositif ne verrait pas ? Puis vos **suggestions
concrètes d'amélioration** : quoi changer, où, et ce que cela apporterait au
client (vous pouvez suggérer une nouvelle architecture). Ce n'est pas un exposé
de normes : c'est votre lecture de ce qui compte pour ce client précis.*

## 6. Journal de bord *(≤ 1 page)* · toutes les séances

- **Répartition** : *qui a fait quoi.*
- **Usage de l'IA** : *où elle vous a aidés, et ce que vous avez vérifié
  derrière. Fun fact lié à l'usage de l'IA durant le projet*

---

## Annexes *(libres, non notées)*

*Configurations complètes, extraits de logs bruts, captures supplémentaires.*
