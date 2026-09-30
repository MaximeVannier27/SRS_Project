# Dossier client — Régie des Eaux *(secteur OT / eau potable)*

## 1. Lettre de mission

*De la Direction technique de la Régie des Eaux de la Vallée du Cern,*
*à l'attention de votre équipe.*

Nous produisons et distribuons l'eau potable pour une trentaine de communes.
Notre métier repose sur une **supervision permanente** de nos installations :
réservoirs, stations de pompage, traitement. Une coupure de cette supervision,
même brève, nous prive de la vue temps réel sur un service vital : c'est un
risque que nous ne prenons pas.

La réglementation nous impose désormais de **sécuriser** ce système, longtemps
resté à l'écart des préoccupations informatiques. Nous vous confions cette
mission : concevoir le cloisonnement de notre supervision, mettre en place une
supervision de sécurité capable de repérer un usage anormal, et savoir
**qualifier un incident** si l'un survient. Le tout **sans jamais interrompre**
la conduite des installations.

## 2. Présentation de l'exploitation

La Régie exploite une quinzaine de sites (captages, réservoirs, stations de
pompage). Chacun est équipé d'automates qui mesurent en continu niveaux de cuve,
débits, pression et taux de chlore, et qui actionnent pompes et vannes.

Notre parc s'est constitué au fil des marchés publics : nous avons aujourd'hui
des automates de **deux constructeurs**, qui ne parlent pas la même langue.
Les plus anciens, de marque **Hydrelec**, communiquent en Modbus/TCP ; les plus
récents, de marque **Aquatron**, en OPC UA. Plutôt que de tout remplacer, nous
avons installé la **passerelle de supervision** fournie par chaque constructeur :
un serveur par marque, qui dialogue avec ses automates et expose leurs mesures et
leurs réglages sous une forme commune. Chaque passerelle est livrée avec **un
unique compte d'accès**, que le constructeur ne sait pas démultiplier ; tout ce
qui parle à une passerelle le fait sous ce compte. Les deux passerelles
s'ignorent : chacune ne connaît que ses propres automates.

La conduite des installations se fait depuis la **salle de conduite**, où est
installé le **pupitre de conduite**. C'est une règle ancienne de la maison : on ne
touche à un réglage (consigne de chloration, démarrage d'une pompe, seuil d'un
réservoir) que depuis cette salle. Les exploitants s'y
relaient, consultent l'état des sites et interviennent quand une valeur sort des
clous. Pour comprendre une dérive, ils remontent aussi les courbes de la veille
ou de la semaine.

Ces courbes, c'est **l'historisation** qui les fournit. Un serveur dédié
interroge chaque passerelle **à intervalle régulier** pour récupérer les
dernières mesures et les archiver, puis les met à disposition du pupitre.
Cette collecte n'est le fait de personne : elle **tourne toute seule**, jour et
nuit, à cadence fixe. Elle se contente de **lire** les mesures ; elle ne pilote
aucun équipement, ne modifie aucun réglage, ne se déplace pas d'une machine à
l'autre.

Cet historique a une valeur réglementaire : c'est lui que nous produisons à
l'Agence régionale de santé en cas de contrôle ou de non-conformité de l'eau
distribuée. Il faut pouvoir dire, à la seconde près, quand une mesure a été
relevée et quand un réglage a été modifié. Toutes nos machines doivent donc être
d'accord sur l'heure. La télégestion n'ayant pas accès à Internet, c'est le
serveur d'historisation qui donne l'heure à toutes les autres machines.

## 3. Fiche technique

### Organisation & horaires

- **Salle de conduite** : occupée en permanence, les exploitants s'y relaient
  en quarts. Une dizaine de personnes se relaient au pupitre.
- **Historisation** : fonctionne **24h/24, 7j/7, sans interruption** c'est la
  contrainte non négociable.

### Postes & accès

- **Pupitre de conduite** : dans la salle de conduite. Consulte et pilote les
  installations via les deux passerelles, consulte les courbes sur le serveur
  d'historisation. C'est un simple poste de travail : il n'héberge aucun
  service.
- **Passerelles de supervision (2)** : une par constructeur (Hydrelec,
  Aquatron). Exposent en HTTPS les mesures et les réglages de leurs automates.
  Cœur du système.
- **Serveur d'historisation** : interroge les deux passerelles à cadence fixe,
  archive les mesures et les sert au pupitre. Sert aussi de référence horaire
  à la télégestion. Fonctionne sans intervention humaine régulière.
- **Comptes d'accès aux passerelles** : un par passerelle, imposé par le
  constructeur. Chaque compte est utilisé à la fois par l'historisation et par
  le pupitre. Son secret est posé une fois pour toutes dans la configuration
  de chacun.
- **Accès externe** : aucun accès direct depuis l'extérieur vers la supervision.
  En exploitation, aucune machine de la télégestion n'accède à Internet ; les
  mises à jour se font lors de fenêtres planifiées, avec l'accord de la
  direction technique.

### Services en place *(hérités, à superviser)*

- **Passerelles de supervision** : le service de chaque constructeur, qui
  expose mesures et réglages.
- **Serveur d'historisation** : collecte, archive et restitue l'historique.
- **Pupitre de conduite** : accès des exploitants à la supervision.

> [!NOTE]
> **Hors périmètre de la mission : le terrain.** Les automates, les sites et
> les liaisons Modbus/OPC UA entre passerelles et automates ne vous sont pas
> confiés et ne sont pas instanciés dans votre environnement. Votre périmètre
> commence aux passerelles de supervision.
>
> En revanche, chaque passerelle doit toujours pouvoir joindre ses automates :
> c'est par ce lien que partent les réglages décidés en salle de conduite et
> que remontent les mesures. Votre dispositif ne doit pas le couper, même si
> le terrain n'est pas présent dans votre environnement.

## 4. Contraintes & exigences

### Normes applicables

- **NIS2** : les opérateurs de services essentiels (dont l'eau potable) doivent
  élever leur niveau de sécurité et savoir détecter et notifier les incidents.
- **IEC 62443** : la sécurité des systèmes d'automatisation et de contrôle
  industriels : cloisonnement en zones et conduits, maîtrise des comptes.

### Les exigences du client

Notre direction pose deux exigences, et elle tient aux deux :

1. **La supervision ne doit jamais s'interrompre.** La collecte des mesures et
   la conduite des installations sont vitales ; toute action qui risquerait de
   les couper, même le temps d'une bascule, est exclue.
2. **Le système doit être sécurisé comme l'exige la réglementation** : les
   accès et les comptes doivent être maîtrisés, cloisonnés, et leur usage
   traçable.

### Le cadre de votre mission

Pendant toute la durée de la mission, vous travaillez avec l'existant tel
qu'il est :

- la supervision **n'est jamais interrompue** : il n'y a pas de fenêtre d'arrêt ;
- les passerelles, **leurs comptes** et leur configuration restent en l'état :
  ce sont des produits constructeurs, vous n'en modifiez ni le logiciel ni les
  secrets.

Votre dispositif de sécurité doit donc tenir **avec** ces contraintes, pas en
les contournant.

Ce cadre n'est pas gravé dans le marbre. Si vous estimez qu'une de ces
contraintes fait courir un risque à la Régie, dites-le-nous **dans votre bilan
de fin de mission**, arguments à l'appui, et proposez-nous une alternative :
nous l'étudierons pour la suite. D'ici là, nous ne la remettrons pas en cause.
