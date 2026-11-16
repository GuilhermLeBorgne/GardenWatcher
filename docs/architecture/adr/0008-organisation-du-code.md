# ADR-0008 : Organisation interne du serveur

- **Statut** : accepté
- **Date** : 2026-11-17
- **Point de la todo-list** : 9

## Contexte
Le serveur (ADR-0007) sert deux utilisateurs sur un PC ou un Raspberry Pi. Il doit rester simple à exploiter, facile à tester en TDD, et pouvoir évoluer vers un produit.

## Options envisagées
1. **Services séparés** (plusieurs programmes qui communiquent) : trop complexe pour ce contexte.
2. **Un seul programme découpé en modules**, chacun organisé en couches.

## Décision

### Un seul processus
L'API et les tâches planifiées tournent dans un seul processus, avec une seule instance du serveur web pour éviter que les tâches planifiées s'exécutent en double. Le code reste organisé de façon à pouvoir sortir les tâches planifiées dans un second processus si besoin.

### Modules
| Module | Rôle |
|---|---|
| `garden` | Entités, relations, statuts, position sur la carte |
| `knowledge` | Fiches de connaissances, TAXREF, enrichissement par l'IA |
| `tasks` | Calendrier, propositions, modes de validation |
| `planning` | Moteur de planification : règles, IA, créneaux, périodes sensibles pour la faune |
| `weather` | Prévisions, alertes |
| `water` | Restrictions d'eau (VigiEau) |
| `notifications` | File des messages en attente, Telegram |
| `users` | Comptes, créneaux de disponibilité |
| `ai` | Connexion au fournisseur d'IA, budget, journal de consommation |
| `core` | Configuration, horloge, base de données, journal des incidents |

### Couches dans chaque module
- `domain/` : règles métier pures, sans accès réseau, disque ni base.
- `services/` : cas d'usage, qui orchestrent le domaine et les accès.
- `adapters/` : base de données, clients HTTP.
- `api/` : routes FastAPI.

**Règles de dépendance :**
- le domaine ne dépend d'aucune autre couche ;
- un module n'accède jamais aux tables d'un autre module : il passe par les services de ce module ;
- pas de système d'événements interne en V1 : ce sont les tâches planifiées qui enchaînent les étapes.

### Langues
- **Code en anglais** (noms de modules, classes, fonctions, variables, tables), conformément à l'usage de l'écosystème. Le vocabulaire métier suit le glossaire de `CLAUDE.md`.
- **Français** pour l'interface, les messages Telegram, les contenus produits par l'IA et la documentation.
- **Internationalisation plus tard**, si le projet devient un produit. Dès la V1, les textes affichés à l'utilisateur sont **regroupés dans des fichiers dédiés** et non éparpillés dans le code, pour que la traduction soit simple le jour venu.

## Conséquences
- La plupart des tests portent sur `domain/` : ils sont rapides et ne simulent rien. Les services sont testés avec des doublures, et les adaptateurs avec des réponses enregistrées.
- Le glossaire français → anglais de `CLAUDE.md` doit être tenu à jour à chaque nouveau concept métier.
- Les consignes envoyées à l'IA demandent explicitement des réponses en français.
