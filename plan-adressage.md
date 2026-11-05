# Plan d'adressage : Régie des Eaux de la Vallée du Cern, télégestion

*Remis au groupe 2 en début de séance 2, avec les identifiants (à part).*

## Accès

| Accès | Comment |
|---|---|
| Machine rebond (jump) | `ssh -p 2202 presta@<adresse communiquée>`, mot de passe remis à part |
| Autres machines | depuis le jump : `ssh <adresse>` (sans mot de passe) |
| Interface Wazuh | `https://<adresse communiquée>:4402`, compte `admin`, mot de passe remis à part |

À la première connexion, SSH vous demande de confirmer l'empreinte du jump.
Comparez-la avant de répondre `yes` : `SHA256:<empreinte à compléter>`.

Le compte `presta` a les droits `sudo` sur toutes les machines. Toutes doivent
rester administrables en SSH depuis le jump pendant toute la mission.

> Vous êtes sur l'infrastructure de production du client : un grand pouvoir
> implique de grandes responsabilités.
>
> Les applications du client, leurs fichiers (programmes, configuration,
> données) et leurs journaux lui appartiennent : ne les modifiez pas. Vous
> supervisez et durcissez le système autour de ces applications.

> **Confidentialité des accès.** Le client vous confie son système de
> production. Son adresse publique, les ports d'accès et vos identifiants sont
> des secrets : ils restent entre vous et le client. Il serait intolérable
> qu'un prestataire de sécurité donne à un outil d'IA un accès aux systèmes
> métier de son client : aucune IA ne doit se connecter aux machines, ni y
> agir à travers vous. L'infrastructure est supervisée : un comportement
> d'IA détecté entraînera une sanction sur la note du groupe.

## Réseau `192.168.102.0/24`

| Machine | Adresse | Fonction |
|---|---|---|
| `jump` | 192.168.102.10 | machine rebond, seul point d'entrée depuis l'extérieur |
| `scada-a` | 192.168.102.20 | passerelle de supervision Hydrelec |
| `historian` | 192.168.102.21 | serveur d'historisation |
| `pupitre` | 192.168.102.22 | pupitre de conduite (salle de conduite) |
| `scada-b` | 192.168.102.24 | passerelle de supervision Aquatron |
| `wazuh` | 192.168.102.30 | serveur de supervision Wazuh |
| (passerelle) | 192.168.102.1 | passerelle vers l'extérieur, gérée par l'hébergeur : hors périmètre |

Les automates de terrain ne sont pas dans votre environnement (voir le dossier
client).

## Où trouver les journaux

> Étudiez les journaux **à partir du jour de la remise des accès** : ce qui
> précède correspond à la mise en service de l'infrastructure.

**Sur toutes les machines** : le journal système (`journalctl`), et ses copies
texte dans `/var/log/` (`syslog`, `auth.log`, `kern.log`) ; les installations
de paquets (`/var/log/dpkg.log`, `/var/log/apt/`).

Les applications du client tiennent aussi leurs propres journaux : à vous de
les localiser.

Pour repérer ce qui tourne sur une machine, et où cela écrit :
`systemctl list-units --type=service --state=running` (services),
`systemctl list-timers` (tâches planifiées), `systemctl cat <unité>`
(programme lancé, compte, répertoire de travail), `sudo ss -tulpn` (ports en
écoute), `journalctl -u <service>` (journal d'un service).
