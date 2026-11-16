# ADR-0009 : Moteur de planification

- **Statut** : accepté
- **Date** : 2026-11-17
- **Point de la todo-list** : 10

## Contexte
La planification est le cœur du produit. Elle doit combiner les fiches de connaissances (ADR-0006), la météo et les restrictions d'eau (ADR-0005), les modes d'arrosage, les créneaux de disponibilité, les périodes sensibles pour la faune, ainsi que les propositions de l'IA et leurs modes de validation (ADR-0003), sans jamais imposer quoi que ce soit à l'utilisateur.

## Options envisagées
1. **Planification entièrement confiée à l'IA** : coûteuse, difficile à tester, et l'outil s'arrête quand l'IA est indisponible.
2. **Règles seules** : fiables, mais elles ne voient pas les situations croisées.
3. **Règles, plus l'IA en conseillère.**

## Décision
Option 3.

### Le fonctionnement
Le moteur tourne une fois par jour, sur les 7 jours à venir, uniquement pour les entités dont la fiche est validée.
1. **Besoins** :
   - les règles traitent l'arrosage et le calendrier de la fiche ;
   - l'IA ajoute des propositions justifiées (dans la limite du budget) ;
   - l'utilisateur ajoute ses tâches manuelles.
2. **Contraintes** : pas d'arrosage si une pluie suffisante est tombée ou prévue, pas de plantation avant un gel, **avertissement** pendant les périodes sensibles pour la faune, créneaux interdits si l'utilisateur a activé une restriction d'eau.
3. **Placement** : la tâche est placée dans le meilleur créneau qui respecte à la fois les créneaux de disponibilité des utilisateurs, les horaires autorisés et la météo.
4. **Validation** :
   - selon le mode de l'entité, la tâche est `proposed` (proposée) ou `accepted` (acceptée) ;
   - en mode hybride, un « type de tâche » correspond à **action + entité** ;
   - le mode de validation se règle **par entité**, avec une valeur par défaut pour le jardin.
5. **Notification** sur Telegram.

### Les principes
- **L'IA conseille, elle ne décide pas** : si elle est indisponible ou si son budget est épuisé, les règles continuent de planifier.
- **Le moteur ne modifie jamais une tâche validée ou modifiée par l'utilisateur.** Si les conditions changent, il propose un changement et prévient.
- **Chaque tâche indique sa source** (règle, IA ou manuelle) **et sa justification.**

### Modes d'arrosage en V1
- **Simple** : un arrosage par jour, avec une quantité indicative (léger, normal ou copieux) selon la fiche. Il est sauté si une pluie suffisante est tombée ou prévue.
- **Renforcé** : la quantité, la fréquence et l'heure s'adaptent à la chaleur, au vent, à la pluie et aux besoins de l'espèce.
- **Expert** : reporté après la V1 (bilan hydrique : évaporation × coefficient propre à l'espèce, moins la pluie reçue).

### Notifications des arrosages
- **Mode simple** : une seule notification par jour regroupe tous les arrosages des entités en mode simple.
- **Mode renforcé** : une notification par entité.

### Attribution des tâches
- Les tâches ne sont pas attribuées en V1. Elles appartiennent à « nous deux » et sont placées dans les créneaux où au moins un des utilisateurs est disponible.
- Les notifications sont envoyées dans un **groupe Telegram commun** (les deux utilisateurs et le bot). Les utilisateurs se coordonnent en se répondant dans ce groupe. Le bot n'envoie que des messages et ne lit pas le groupe (ADR-0004).
- L'utilisateur marque la tâche comme faite dans l'application web.

## Conséquences
- Les règles d'arrosage, de contraintes et de placement relèvent du domaine `planning`. Ce sont des fonctions pures, développées en TDD avec une horloge figée.
- Le modèle de données (point 11) doit gérer : le statut des tâches (`proposed`, `accepted`, `done`, etc.), la source et la justification, les tâches modifiées par l'utilisateur, le mode de validation par entité, et le regroupement des arrosages simples.
- Plus tard : un bouton « Je prends » dans Telegram pour s'attribuer une tâche (lecture des réponses par le bot), et le mode expert.
