# ADR-0003 : Usage de l'IA

- **Statut** : accepté
- **Date** : 2026-11-16
- **Point de la todo-list** : 4

## Contexte
Le serveur n'a pas de GPU (ADR-0001, point 2) : un modèle d'IA local est exclu. L'IA sert à rechercher des informations sur les entités et à en déduire des actions à proposer. Il faut maîtriser son coût, et l'outil doit continuer à fonctionner quand elle est indisponible (ADR-0002).

## Options envisagées
1. **L'IA fait tout** (météo, calculs, alertes, planification) : coûteux, peu fiable pour les calculs, difficile à tester.
2. **IA limitée à ce qu'elle seule sait faire**, tout le reste en code déterministe.

## Décision
Option 2, via une API d'IA dans le cloud (API Claude).

**Usages de l'IA en V1 :**
- **A. Enrichissement d'une entité** (à la création, sur demande) : recherche et lecture web côté API. Le résultat est une fiche espèce au format imposé (schéma validé par le code), avec ses sources, stockée localement.
- **B. Planification quotidienne** (tâche de fond, éligible au mode par lots, moins cher) : l'IA reçoit les entités, leurs fiches, un résumé météo déjà calculé et l'historique. Elle renvoie des propositions de tâches au format imposé, chacune avec sa justification.

**Hors IA (code déterministe, testé en TDD) :**
- récupération de la météo, des vigilances et des restrictions d'eau via des API open data (point 6) ;
- calculs (fenêtres d'arrosage, risque de gel, cumul de pluie) ;
- alertes, déclenchées par des règles.

**Hors périmètre V1 :** la conversation avec l'IA (messagerie ou application).

**Validation des tâches proposées**, réglable par l'utilisateur :
- **manuelle** : chaque proposition est validée par l'utilisateur ;
- **automatique** : les propositions sont acceptées directement ;
- **hybride** : la première occurrence d'un type de tâche est validée à la main, les suivantes sont acceptées automatiquement.

Reste à définir au point 10 : ce qu'est un « type de tâche » (action + entité ? action seule ?) et si le mode se règle globalement, par entité ou par type.

**Clé d'API** : renseignée dans un fichier de configuration local. Une offre SaaS (clé gérée par nous) est envisagée pour l'offre payante ou pro.

**Pas de RAG** au départ : les fiches utiles tiennent dans une seule requête.

## Conséquences
- Une interface « fournisseur d'IA » isole l'API : on peut la remplacer par une doublure dans les tests et changer de fournisseur plus tard.
- Un journal de consommation enregistre les jetons utilisés et le coût estimé. Un plafond de budget est réglable dans la configuration ; une fois atteint, l'IA est suspendue et le reste continue.
- Si l'IA est indisponible, les règles déterministes continuent de produire alertes et rappels. Seules les propositions de l'IA sont suspendues.
- Le modèle de données (point 11) doit gérer le statut « proposée » des tâches, leur justification, leurs sources et leur origine (règle, IA, manuelle).
- Coût attendu : quelques dollars par mois, l'essentiel venant de la planification quotidienne.
