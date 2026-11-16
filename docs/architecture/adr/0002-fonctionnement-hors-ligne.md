# ADR-0002 : Fonctionnement hors ligne

- **Statut** : accepté
- **Date** : 2026-11-16
- **Point de la todo-list** : 3

## Contexte
Le serveur est installé chez l'utilisateur, et sa connexion Internet peut tomber. Il faut décider ce qui doit continuer à marcher dans ce cas.

## Décision
**Ce qui reste disponible sans Internet** (sur le réseau local) :
- consulter les entités, leurs fiches et la carte ;
- consulter et gérer les tâches déjà prévues (calendrier).

**Ce qui fonctionne en mode dégradé** :
- la création d'entités : la saisie manuelle reste possible, seule la recherche d'informations sur Internet est indisponible.

**Ce qui est temporairement indisponible** :
- tout ce qui a besoin d'Internet : IA, météo, recherche web, envoi de messages (Telegram, WhatsApp…).

**Gestion des erreurs et des incidents** : chaque appel vers un service externe doit pouvoir échouer proprement. L'indisponibilité s'affiche clairement dans l'interface, et un journal garde la trace des incidents. Quand la connexion revient, les opérations en attente sont relancées (envois de notifications, mises à jour météo).

## Conséquences
- **Données** : toutes les données du jardin sont stockées sur le serveur local. Aucune n'est seulement dans le cloud.
- **Code** : chaque service externe (IA, météo, messagerie, recherche web) passe par une interface que l'on peut remplacer par une doublure dans les tests. Cela facilite le TDD (point 15) et permet de tester les pannes.
- **Météo** : garder en cache les dernières prévisions et indiquer leur date.
- **Notifications** : prévoir une file d'attente persistante, pour que les messages non envoyés partent quand Internet revient.
- **Plus tard** : un serveur externe de surveillance, qui alerte l'utilisateur via une messagerie si son serveur ne s'est pas manifesté depuis X temps (voir « Idées pour plus tard » dans `TODO.md`). C'est une fonctionnalité candidate au mode payant.
