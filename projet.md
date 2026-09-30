# Projet SRS — Supervision d'un SI client

*Pierre Dubreuil*

Durée : 14 heures encadrées (4 séances : 2h + 3 × 4h)

---

## Le cadre

Vous êtes une **équipe prestataire en cybersécurité**, mandatée par un client pour prendre en
charge son système d'information. Selon le secteur qui vous est attribué
(établissement de santé, opérateur d'eau, laboratoire d'IA), vous devrez :

1. **Comprendre le métier** du client et traduire ses besoins en architecture.
2. **Superviser** l'infrastructure : collecter les journaux, écrire des règles
   de détection.
3. **Durcir l'existant** : un filtrage adapté sur chaque machine, sans casser
   les usages du client.
4. **Qualifier un incident** lorsqu'une activité malveillante survient.
5. **Proposer des évolutions architecturelles** pour améliorer le niveau de maturité du client.

Chaque groupe reçoit un **dossier client** différent. Lisez-le attentivement :
il contient, dans la description du fonctionnement de l'établissement, tout ce
dont vous avez besoin pour définir ce qui est *normal* et donc pour repérer ce
qui ne l'est pas.

## La démarche

Vous commencez par découvrir le SI de votre client, en deux temps.

1. **Ce que le client en dit**. Vous partez de ses explications,
   c'est-à-dire du dossier client, pour comprendre comment son SI fonctionne :
   qui fait quoi, depuis quel poste, à quel moment. Vous en tirez un schéma de
   l'architecture telle que vous la comprenez.
2. **Ce que l'infrastructure montre**. Une fois vos accès reçus,
   vous constatez sur les machines ce qu'il en est réellement : services en
   écoute, flux observés, journaux produits.

Entre les deux, il y aura des **écarts**, et c'est normal. Le client décrit son
métier avec ses mots, l'expert raisonne en machines, en ports et en flux :
aucun des deux ne parle tout à fait le langage de l'autre, et un client ne
connaît jamais son SI dans le moindre détail. Relevez ces écarts et
expliquez-les plutôt que de les corriger en silence : ils font partie de votre
analyse.

Vient ensuite le travail de sécurisation, qui se construit par itérations.

3. **Mettre en place la supervision.** Le serveur Wazuh vous est livré
   installé sur la machine prévue à cet effet. À vous de déployer un agent
   Wazuh sur chacune des autres machines et de vérifier que leurs journaux
   arrivent bien. Commencez par quelques sources de journaux, en priorité
   celles du jump, seule machine exposée, puis élargissez pas à pas : journaux
   système, applicatifs, de pare-feu. À chaque source ajoutée, demandez-vous
   ce qu'elle vous permet de voir de plus, et ajustez vos règles pour que
   l'alerte utile ne soit pas noyée dans le bruit.
4. **Durcir.** À partir des flux légitimes que vous avez identifiés, vous
   mettez en place un pare-feu sur chaque machine : n'autoriser que ce dont le
   métier a besoin, refuser le reste et journaliser ce qui est refusé. Ces
   refus deviennent une nouvelle source pour Wazuh. Vérifiez après chaque
   changement que les usages du client fonctionnent toujours, sans oublier
   ceux de votre propre dispositif (remontée des agents Wazuh, administration
   par le jump) : un durcissement qui casse le service n'est pas une réussite.
5. **Détecter et qualifier.** Plus votre supervision s'étend, plus vous êtes
   en mesure de repérer des activités anormales, y compris des incidents que
   vous n'auriez pas vus au début. Chaque incident détecté est qualifié et
   donne lieu à un post-mortem.

Ces étapes ne se font pas une fois pour toutes : un nouveau journal peut
révéler un flux oublié, qui amène à revoir une règle de pare-feu, dont les
refus journalisés enrichissent à leur tour la supervision, et ainsi de suite.

Si tout cela est en place, ne vous arrêtez pas là : **creusez d'autres pistes
de durcissement** (comptes et droits, services inutiles, configuration SSH,
détection réseau avec Suricata, etc.). Ce que vous ne pouvez pas mettre en
œuvre dans le cadre de la mission nourrira vos recommandations au client.

Vous remarquerez que votre client vous soumet des **exigences qui ne
peuvent pas être toutes satisfaites en même temps**. Il n'existe pas de réponse
unique : la façon dont vous durcissez le SI (ce que vous ouvrez, ce que vous
fermez, ce que vous journalisez) détermine ce que vous serez capables de
détecter ensuite. **Prenez du recul là-dessus au fil du projet** : c'est la
matière de votre « ouverture » en soutenance et dans le rapport.

## Le déroulé des séances

Le [déroulé des séances](deroule-seances.md) donne, pour chaque séance,
l'objectif, les étapes et ce que vous devez avoir produit en sortie. Servez-vous
en pour suivre votre avancement. Il pourra **évoluer au fil du projet** pour
s'adapter à la progression des groupes : reportez-vous toujours à sa dernière
version.

## Les incidents

Des activités malveillantes peuvent survenir **à partir de la séance 2**, sans
préavis : vous ne saurez ni quand, ni lesquelles. Votre **machine rebond**
(« jump ») est la seule exposée sur l'extérieur : c'est une cible évidente,
collectez ses journaux au plus tôt.
Votre dispositif de supervision sera jugé sur sa capacité à les **détecter et à
les qualifier**, pas seulement à les bloquer. Chaque incident subi donne lieu à
un **post-mortem** (voir le modèle de rapport).

## L'IA comme outil, pas comme auteur

L'usage d'assistants IA est autorisé comme **outil d'aide** (comprendre une
norme, explorer une piste, faciliter la compréhension). Il ne peut pas produire le
travail à votre place : l'évaluation repose sur *votre* infrastructure (vos IP,
vos logs, vos captures) et sur la présentation cliente en fin de projet.
Un rapport générique, non ancré dans ce que vous avez réellement fait, ne
rapporte pas de points.

Vous tiendrez un **journal de bord** où vous noterez notamment où l'IA vous a
aidés et ce que vous avez vérifié derrière. Ce n'est pas un piège : savoir
utiliser puis contrôler un outil est une compétence attendue.

**⚠️ Warning: la détection d'une IA qui viendrait agir directement sur l'infrastructure mise à disposition durant le TP sera pénalisée.**

## Livrables et notation

Le projet est noté sur 20 points, en deux parties.

- **Rapport final : 12 points.** Il suit le modèle proposé dans
  [`rapport-final.md`](rapport-final.md) et est rendu après la soutenance, au
  plus tard à la fin de la semaine de la dernière séance.
- **Soutenance : 8 points.** Elle prend la forme d'une **restitution au
  client** (environ 10 minutes, en séance 4) : les actions que vous avez
  menées, la supervision mise en place et les événements que vous avez
  détectés. Elle se termine par un temps de questions-réponses.

## Outils

- **Virtualisation** : Proxmox (infrastructure fournie et administrée par
  l'encadrant).
- **Accès** : une **machine rebond** (« jump ») SSH par groupe (point d'entrée unique vers votre
  périmètre). Toutes les machines de votre périmètre s'administrent en SSH, en
  rebondissant par le jump ; elles doivent le rester pendant toute la mission,
  quel que soit le durcissement que vous leur appliquez.
- **Supervision** : Wazuh (imposé). Suricata en extension possible pour les
  groupes qui avancent vite.
- **Filtrage** : iptables / nftables.
