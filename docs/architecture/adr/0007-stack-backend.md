# ADR-0007 : Stack technique du serveur

- **Statut** : accepté
- **Date** : 2026-11-17
- **Point de la todo-list** : 8

## Contexte
Le serveur doit tourner sur un PC Ubuntu x86 puis sur un Raspberry Pi ARM (ADR-0001). Il appelle l'API Claude avec des réponses au format imposé (ADR-0003), exécute des tâches planifiées (météo, VigiEau, planification, file de messages), et expose une API pour l'interface web. Le projet sert aussi à apprendre le TDD et le développement assisté par IA. Le porteur du projet apprécie la philosophie de Python et ne souhaite plus faire de TypeScript que pour les interfaces.

## Options envisagées
1. **Python** : très bon outillage de test, bibliothèque officielle Claude, riche pour les données et les calculs.
2. **TypeScript (Node)** : un seul langage pour le serveur et l'interface, mais le porteur ne veut plus l'utiliser côté serveur.
3. **Go** : un seul fichier exécutable, très économe, mais inconnu du porteur et plus verbeux.
4. **Rust** : performant et strict, mais difficile à apprendre, sans bibliothèque officielle Claude, et sa performance n'apporte rien ici.

## Décision
**Python 3.12+**, avec :

| Rôle | Outil |
|---|---|
| Serveur web et API | FastAPI |
| Validation, formats de données, configuration | Pydantic (et pydantic-settings) |
| Base de données | SQLite, via SQLAlchemy 2, migrations Alembic |
| Tâches planifiées | APScheduler, dans le processus du serveur |
| Appels HTTP sortants | httpx |
| Tests | pytest. respx pour simuler les appels HTTP, time-machine pour contrôler l'heure |
| Qualité du code | ruff (vérification et formatage), vérification des types (mypy ou pyright) |
| Gestion du projet | uv |

## Conséquences
- Toutes ces dépendances fonctionnent sur x86 et sur ARM.
- SQLite tient dans un seul fichier : une sauvegarde se résume à une copie (point 16). Activer le mode WAL.
- Les tâches planifiées doivent pouvoir être relancées sans effet en double (idempotentes), parce que le serveur peut redémarrer à tout moment. La file des messages en attente est une table de la base.
- Les services externes sont définis comme des interfaces (`typing.Protocol`), avec une implémentation réelle et une doublure de test pour chacun.
- Le temps est une dépendance injectée (une horloge), jamais lu directement : la planification et les alertes en dépendent, et les tests doivent pouvoir figer l'heure.
- Une fois le projet initialisé, `CLAUDE.md` doit donner les commandes de lancement, de test (y compris un test isolé) et de vérification du code.
