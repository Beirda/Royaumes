# ADR-0002 - L'Event Sourcing comme mode de persistance du Village

- Statut : accepté
- Date : 2026-10-19

## Contexte

L'état d'un village évolue par des actions successives du joueur : fondation, production de ressources et améliorations de bâtiments.

La suite du projet prévoit de diffuser ces changements en temps réel (TD5) et de construire des vues de lecture spécifiques à partir de l'activité des joueurs (TD8).

Enfin, avec l'hébergement sans serveur (Vercel), plusieurs instances de l'application peuvent être exécutées en parallèle et traiter simultanément des requêtes pour un même village.

## Décision

La persistance du village repose sur l'Event Sourcing :

- L'état du village n'est pas stocké en base : il est reconstruit à la volée en rejouant l'ensemble de ses événements.
- Les événements sont enregistrés dans le journal générique de la plateforme (`platform.events`), dans un flux par village nommé `village-<joueur>`.
- Les écritures concurrentes sont détectées par la contrainte d'unicité sur le flux et la version. En cas de conflit, la commande est rejouée au plus trois fois sur l'état à jour.
- Chaque événement porte toutes les informations nécessaires à son application, comme le coût payé pour une amélioration, pour ne pas dépendre de règles de calcul futures lors du rejeu.

## Conséquences

### Positives

- Historique complet et inaltérable des actions du joueur, sans perte d'information.
- Base naturelle pour diffuser les événements en temps réel (TD5) et construire de nouvelles vues de lecture (TD8).
- Possibilité de rejouer les événements pour reproduire des bugs ou inspecter un état passé.
- Écritures simples et rapides en ajout seul (*append-only*), sans mise à jour sur place.

### Négatives

- Impossibilité de modifier ou supprimer un événement enregistré : faire évoluer la structure des événements impose de gérer la rétrocompatibilité dans le code de rejeu.
- Le coût de reconstruction de l'état augmente à mesure que le flux d'événements grandit.
- Impossible d'interroger directement l'état global des villages en SQL sans passer par des projections dédiées.

## Alternatives écartées

- **Stockage de l'état dans une table propre au module (`realm.villages`)** : simple à mettre en place, mais écrase l'état à chaque modification. Cela fait perdre l'historique et complique la diffusion temps réel ainsi que la création de vues de lecture a posteriori.
- **Verrouillage pessimiste (`SELECT ... FOR UPDATE`)** : inadapté à un hébergement sans serveur avec un pool de connexions réduit. Cela bloquerait des connexions inutilement alors que les conflits concurrents sur un même village restent rares.

## Réexamen

Cette décision devra être revue si le nombre d'événements par village devient trop élevé et ralentit la reconstruction de l'état, ce qui nécessitera la mise en place d'instantanés (*snapshots*).

## Usage de l'IA générative

Premier jet généré avec l'assistant IA, puis relecture, synthèse et ajustements manuels.