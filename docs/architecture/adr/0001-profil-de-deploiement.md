# ADR-0001 : Profil de déploiement

- **Statut** : accepté
- **Date** : 2026-11-16
- **Point de la todo-list** : 1

## Contexte
Il faut savoir pour qui on construit avant de choisir quoi que ce soit d'autre : nombre d'utilisateurs, mode d'installation, niveau d'exigence.

## Options envisagées
1. **Usage personnel uniquement** : simple, mais on risque de devoir tout reprendre si le projet devient un produit.
2. **Produit multi-clients dès le départ** : trop lourd pour un premier jet sans aucun utilisateur.
3. **Usage personnel d'abord, produit possible ensuite.**

## Décision
Option 3.
- Les premiers utilisateurs sont le porteur du projet et sa compagne : **un serveur, un jardin partagé, deux utilisateurs**.
- On garde la porte ouverte à un produit installé chez d'autres clients, avec à terme un éventuel « mode payant » aux fonctionnalités avancées.
- Le projet sert aussi à apprendre à développer un produit sérieux : TDD obligatoire (voir `CLAUDE.md`) et développement assisté par IA.

## Conséquences
- **Utilisateurs (point 12)** : prévoir des comptes utilisateurs dès le départ (au moins deux), avec un jardin commun.
- **Installation (point 16)** : pas besoin d'un installateur grand public tout de suite, mais l'installation et la configuration doivent rester scriptables et reproductibles.
- **Réglages** : rien de spécifique au jardin du porteur ne doit être codé en dur (localisation, clés d'API, etc.). Tout passe par la configuration.
- **Choix techniques** : privilégier des technologies bien documentées et bien testables, cohérentes avec l'objectif d'apprentissage TDD et IA.
