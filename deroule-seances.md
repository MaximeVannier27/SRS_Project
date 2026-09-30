# Déroulé des séances

| Séance | Infra | Incident | En sortie |
|---|---|---|---|
| **1. Concevoir** (2h) | Non | — | Schéma d'architecture attendu |
| **2. Superviser** (4h) | Oui | peut-être 😇 | Wazuh en place, matrice de flux, post-mortem ?, énumération des logs surveillés |
| **3. Durcir** (4h) | Oui | peut-être 😈 | Filtrage par machine, règles, post-mortem ?, ajustement monitoring |
| **4. Restituer** (4h) | Oui | peut-être 🔥 | Ajustements divers, avancer sur le rapport, post-mortem ?, puis soutenance|

---

## Séance 1 : concevoir (2h, sans infra)

Vous recevez votre **dossier client** et le barème. Pas encore d'accès à
l'infrastructure.

- [ ] Lire le dossier client **en entier**, récit compris.
- [ ] Repérer les machines, les personnes et les échanges : qui parle à qui,
      et pourquoi.
- [ ] Poser vos questions si nécessaire au client (l'encadrant joue ce rôle).

**En sortie** : un **schéma d'architecture** de ce que vous attendez. A insérer tel quel dans le rapport final.

> [!NOTE]
> **Un écart entre votre schéma et le réel est normal.** Vous avez conçu sur
> papier, à partir d'un client qui ne connaît pas parfaitement son SI. Ce qui
> compte, c'est de savoir **expliquer** chaque écart : flux oublié, flux
> technique, flux que le client ignorait, flux inexpliqué à creuser.

## Séance 2 : superviser (4h)

Vous recevez vos **accès** (par la machine rebond, « jump ») et le **plan
d'adressage** de votre infrastructure.

- [ ] **Priorité : déployer l'agent Wazuh** sur toutes les machines (le
      serveur Wazuh vous est livré installé). Vérifier que les journaux
      arrivent.
- [ ] Observer les **flux réels** entre les machines et les comparer à votre
      schéma.
- [ ] **Un incident ?** : une attaque pourrait viser le jump. Détectez-la, qualifiez-la, rédigez le **post-mortem**.


> [!WARNING]
> Toutes les machines doivent rester **administrables en SSH depuis le jump**.

**En sortie** : lister les sources de journaux collectées dans Wazuh, les écarts expliqués entre votre schéma d'archi et la réalité, proposer une matrice de flux et un post-mortem si vous subissez un incident.

## Séance 3 : durcir (4h)

- [ ] Écrire les **iptables de chaque machine** grâce à votre matrice de flux. Vérifier que le client fonctionne toujours notamment grâce à vos logs.
- [ ] Définir ce qui est **normal** chez votre client et écrire les **règles de
      détection** qui repèrent l'écart.
- [ ] **Un incident ?** : détectez-le et qualifiez-le, rédigez le
      **post-mortem**. Si vous n'avez rien vu, dites-le et proposez un plan
      pour mieux collecter.

**En sortie** : la matrice de flux finale, vos règles de détection avec une preuve de déclenchement, un post-mortem si vous subissez un incident.

## Séance 4 : restituer (4h)

- [ ] **Un incident ?** : détectez-le et qualifiez-le, rédigez le
      **post-mortem**. Si vous n'avez rien vu, dites-le et proposez un plan
      pour mieux collecter.
- [ ] Avancer dans la rédaction du rapport
- [ ] **Soutenance** (~10 min) : restitution au client (actions menées,
      supervision mise en place, événements détectés), puis questions-réponses.

**Après** : le **rapport final**
([`rapport-final.md`](rapport-final.md)), rendu à la date
annoncée en séance 1.

---

## Tout au long de la mission

- Un **journal de bord** : ce que vous avez fait, quand, et où l'IA vous a aidés.
- Des **preuves horodatées** (captures, extraits de journaux avec vos adresses
  et noms de machines).
- Notez les **contraintes du client qui vous ont gênés** : c'est la matière de
  votre prise de recul en fin de mission.
