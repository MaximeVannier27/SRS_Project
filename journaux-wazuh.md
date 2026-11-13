# Journaux et Wazuh : fiche d'aide, groupe 2

*Remise au groupe 2 en séance 3.*

Deux usages des journaux, dans l'ordre de la séance :

1. **Vérifier que votre filtrage ne casse rien** (§ 1) ;
2. **Détecter** ce qui sort de l'usage normal, avec Wazuh (§ 2).

## 1. Les journaux de votre client

Réseau `192.168.102.0/24`. En plus du journal système (`journalctl`), les
applications du client tiennent leurs propres journaux :

| Machine | Journal | Contenu |
|---|---|---|
| `scada-a`, `scada-b` | `/var/log/nginx/passerelle_access.log` | accès à l'API de la passerelle |
| `scada-a` | `/var/log/hydrelec/consignes.log` | réglages appliqués par la passerelle |
| `scada-a` | `/var/log/hydrelec/automates.log` | état de la liaison avec les automates |
| `scada-b` | `/var/log/aquatron/consignes.log` | réglages appliqués par la passerelle |
| `scada-b` | `/var/log/aquatron/automates.log` | état de la liaison avec les automates |
| `historian` | `/var/log/historisation/collecte.log` | collectes de mesures sur les passerelles |
| `historian` | `/var/log/nginx/courbes_access.log` | accès aux courbes |
| `pupitre` | `/var/log/pupitre/conduite.log` | consultations, courbes et réglages du pupitre |

**Ce sont eux qui vous disent si le filtrage a cassé quelque chose.** Avant de
poser une règle iptables, regardez ce que chaque journal écrit en temps normal
(`sudo tail -f <journal>`) ; après, vérifiez que l'activité continue : mêmes
échanges, même rythme, pas d'erreur ni de connexion refusée. Un journal qui
se tait ou qui se met à signaler des échecs est un flux que votre matrice a
oublié.

Les applications conservent aussi leurs **données** :

| Machine | Données | Contenu |
|---|---|---|
| `scada-a` | `/var/lib/hydrelec/etat.json` | réglages en cours de la passerelle |
| `scada-b` | `/var/lib/aquatron/etat.json` | réglages en cours de la passerelle |
| `historian` | `/var/lib/historisation/mesures.db` | base des mesures collectées |

La surveillance d'intégrité (§ 2.2) peut couvrir celles qui ne doivent pas
changer, ou seulement d'une façon précise.

## 2. Détecter avec Wazuh

### 2.1 Ce que Wazuh collecte déjà

Par défaut, chaque agent envoie le **journal système** (SSH, `sudo`,
sessions, services), les installations de paquets, et surveille l'intégrité
de quelques répertoires système (`/etc`, `/bin`, `/usr/bin`…) toutes les 12
heures. Vérifiez-le dans les blocs `<localfile>` et `<syscheck>` de
`/var/ossec/etc/ossec.conf`, sur un agent.

**Les journaux du § 1 ne sont pas collectés.** N'ajoutez pas
`/var/log/auth.log` ni `/var/log/syslog` : ce sont des copies du journal
système, déjà collecté.

### 2.2 Choisir et collecter les journaux importants

Tous les journaux du § 1 ne servent pas à détecter la même chose. Identifiez
ceux qui répondent aux questions de votre client : *qui fait quoi, depuis
quelle machine, combien, et qu'est-ce qui ne doit jamais changer ?* Justifiez
votre choix dans le rapport.

| Pour… | Où (sur l'agent concerné) | Documentation |
|---|---|---|
| collecter un journal | un bloc `<localfile>` dans `/var/ossec/etc/ossec.conf` | [Log data collection](https://documentation.wazuh.com/current/user-manual/capabilities/log-data-collection/index.html), [localfile](https://documentation.wazuh.com/current/user-manual/reference/ossec-conf/localfile.html) |
| surveiller des fichiers qui ne doivent pas changer | un bloc `<syscheck>` dans `/var/ossec/etc/ossec.conf` | [File integrity monitoring](https://documentation.wazuh.com/current/user-manual/capabilities/file-integrity/index.html), [syscheck](https://documentation.wazuh.com/current/user-manual/reference/ossec-conf/syscheck.html) |

Puis `sudo systemctl restart wazuh-agent`. Les formats utiles ici :
`log_format` `syslog` (une ligne par événement) ou `json` (une ligne JSON par
événement). Pour vérifier qu'un journal est lu :
`sudo grep "Analyzing file" /var/ossec/logs/ossec.log` sur l'agent.

### 2.3 Alerter

**Collecté ne veut pas dire alerté.** Chaque événement reçu est décodé, puis
classé par une règle qui lui donne un niveau ; seuls les niveaux 3 et plus
deviennent des alertes ([Alert management](https://documentation.wazuh.com/current/user-manual/manager/alert-management.html)).
Un accès web réussi, par exemple, est classé au niveau 0 : il n'apparaît
nulle part tant que vous n'écrivez pas de règle.

Les décodeurs qui comptent pour ce projet :

| Journal | Décodeur | Champs extraits |
|---|---|---|
| journal système : SSH, `sudo` | `sshd`, `sudo` | `srcip`, `srcuser`, `dstuser` |
| journaux d'accès nginx (`*_access.log`) | `web-accesslog` | `srcip`, `protocol` (méthode HTTP), `url`, `id` (code de retour) ; le compte n'est pas extrait |
| journal JSON (`log_format` `json`) | `json` | chaque clé devient un champ |
| journaux texte des applications | aucun | `wazuh-logtest` affiche `No decoder matched` |

**`wazuh-logtest`** (`sudo /var/ossec/bin/wazuh-logtest` sur le serveur) est
votre outil n° 1 : collez une ligne de journal, il affiche le décodeur, les
champs et la règle obtenue, avec son identifiant et son niveau
([Testing decoders and rules](https://documentation.wazuh.com/current/user-manual/ruleset/testing.html)).

**Écrire une règle** : sur le serveur, dans
`/var/ossec/etc/rules/local_rules.xml`, identifiant à partir de 100000, puis
`sudo systemctl restart wazuh-manager`
([Custom rules](https://documentation.wazuh.com/current/user-manual/ruleset/rules/custom.html) ; toutes les options :
[Rules syntax](https://documentation.wazuh.com/current/user-manual/ruleset/ruleset-xml-syntax/rules.html)).

Une règle locale part d'une règle existante (`<if_sid>`), celle qui classe
déjà l'événement visé. Son identifiant se lit dans `wazuh-logtest`, en
phase 3, même pour un niveau 0 :

```
**Phase 3: Completed filtering (rules).
	id: '5715'
	level: '3'
	description: 'sshd: authentication success.'
```

Si `wazuh-logtest` affiche `No decoder matched`, aucune règle ne classe la
ligne : votre première règle se passe de `if_sid` et porte sur le texte de la
ligne (`<match>`, voir [Rules syntax](https://documentation.wazuh.com/current/user-manual/ruleset/ruleset-xml-syntax/rules.html)) ;
les suivantes partent d'elle.

**Conseil** : pour une première règle, partez d'un événement que Wazuh sait
déjà lire, c'est-à-dire une ligne pour laquelle `wazuh-logtest` affiche un
décodeur, ou une alerte d'intégrité (550, 554). Les champs extraits rendent la
règle plus simple à écrire et à tester.

Trois exemples, testés sur Wazuh 4.14, sur l'administration de vos machines :
à vous de transposer aux journaux du client.

```xml
<group name="local,">

  <!-- QUI : une condition sur un champ.
       Connexion SSH réussie (5715) depuis une autre machine que le jump. -->
  <rule id="100010" level="10">
    <if_sid>5715</if_sid>
    <srcip negate="yes">192.168.102.10</srcip>
    <description>SSH : connexion réussie depuis une autre machine que le jump ($(srcip))</description>
  </rule>

  <!-- COMBIEN : une fréquence.
       5 commandes sudo (5402) par le même compte en une minute. -->
  <rule id="100011" level="8" frequency="5" timeframe="60">
    <if_matched_sid>5402</if_matched_sid>
    <same_srcuser />
    <description>sudo : rafale de commandes root par $(srcuser)</description>
  </rule>

  <!-- QUOI A CHANGÉ : l'intégrité.
       Fichier modifié (550) sous /etc/ssh, surveillé en temps réel par
       <directories realtime="yes" report_changes="yes">/etc/ssh</directories>. -->
  <rule id="100012" level="10">
    <if_sid>550</if_sid>
    <field name="file">^/etc/ssh/</field>
    <description>Configuration SSH modifiée : $(file)</description>
  </rule>

</group>
```
