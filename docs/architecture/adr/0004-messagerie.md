# ADR-0004 : Messagerie

- **Statut** : accepté
- **Date** : 2026-11-16
- **Point de la todo-list** : 7

## Contexte
Le serveur doit prévenir les utilisateurs (deux pour l'instant, ADR-0001) des tâches du jour, des alertes météo et des changements de restriction d'eau. Il est installé derrière une box internet, et l'accès depuis l'extérieur n'est pas encore décidé (point 13).

## Options envisagées
1. **Telegram** : gratuit ; le serveur va chercher les messages lui-même, donc aucun accès entrant à ouvrir ; boutons d'action possibles ; bot très simple à créer.
2. **WhatsApp (API Meta)** : payant, compte entreprise et numéro dédié, modèles de messages à faire approuver, serveur obligatoirement joignable depuis Internet.
3. **Signal** : pas d'API officielle.
4. **ntfy, e-mail, notifications de l'application web** : possibles, mais moins adaptés comme premier canal.

## Décision
**Telegram**, via un bot.

V1 :
- **envoi seul** : pas d'action depuis la messagerie (pas de bouton, pas de réponse au bot) ;
- **pas de canal de secours**.

## Conséquences
- Le bot se contente d'envoyer, par l'API Bot de Telegram (requêtes sortantes) : aucun port n'est ouvert sur le serveur.
- Une interface « canal de notification » isole Telegram : on peut la remplacer par une doublure dans les tests et ajouter d'autres canaux plus tard.
- Les messages non envoyés (Internet coupé) restent dans une file d'attente persistante et partent au retour de la connexion (ADR-0002).
- Configuration : le jeton du bot et l'identifiant du **groupe Telegram commun** (les deux utilisateurs et le bot, voir ADR-0009) se trouvent dans le fichier de configuration.
- À voir plus tard : les actions depuis Telegram (valider ou reporter une tâche). Cela demandera que le bot lise les messages reçus, ce qui reste possible sans ouvrir d'accès entrant.
