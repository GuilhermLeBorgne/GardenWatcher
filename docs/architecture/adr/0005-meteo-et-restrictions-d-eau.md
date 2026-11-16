# ADR-0005 : Météo et restrictions d'eau

- **Statut** : accepté
- **Date** : 2026-11-16
- **Point de la todo-list** : 6

## Contexte
La planification (arrosage, plantation, récolte) et les alertes reposent sur la météo à court et moyen terme. L'arrosage doit aussi tenir compte des restrictions préfectorales. Ces données doivent coûter le moins possible et ne pas dépendre de l'IA (ADR-0003).

## Options envisagées
1. **Open-Meteo** : gratuit et sans clé pour un usage non commercial, avec des variables utiles au jardin (évaporation ET0, humidité et température du sol), et l'accès aux modèles Météo-France (AROME, ARPEGE).
2. **API Météo-France** : vigilances officielles, mais un compte et une clé sont nécessaires.
3. **Scraper les sites des préfectures** pour les restrictions d'eau : fragile, un format par préfecture.
4. **API VigiEau** pour les restrictions d'eau : publique, centralisée.

## Décision

### Météo : Open-Meteo
- Une couche « fournisseur météo » remplaçable isole le fournisseur.
- **Localisation** : un point GPS pour le jardin, dans la configuration.
- **Horizons** :
  - prévisions heure par heure sur 7 jours, pour les tâches et les alertes ;
  - prévisions jour par jour jusqu'à 16 jours, pour les plantations et les récoltes (fiabilité décroissante au-delà de 7 jours) ;
  - les 7 derniers jours (pluie tombée, évaporation).
- **Fréquence** : toutes les 3 heures.
- **Stockage** : la dernière prévision est conservée avec sa date (fonctionnement hors ligne, ADR-0002), ainsi qu'un résumé journalier dans l'historique du jardin.
- **Mention obligatoire** : « Données météo : Open-Meteo » (licence CC BY 4.0), affichée dans l'application.
- **Pas de vigilances Météo-France.**

### Alertes : règles codées, seuils dans la configuration
| Alerte | Seuil par défaut |
|---|---|
| Gel | minimum ≤ 0 °C |
| Canicule | maximum ≥ 35 °C |
| Vent fort | rafales ≥ 60 km/h |
| Sécheresse | N jours sans pluie significative (N à fixer) |

Plus tard, la fiche espèce pourra affiner ces seuils par plante (point 5).

### Restrictions d'eau : fonctionnalité « Restriction d'eau » avec VigiEau
- Source : l'API VigiEau (`GET /api/zones?lon=&lat=&profil=particulier&zoneType=…`), interrogée une fois par jour.
- `zoneType` est déduit de la **source d'eau** du jardin (configuration) : `AEP` pour le réseau d'eau potable, `SOU` pour un puits ou un forage, `SUP` pour de l'eau de surface.
- **Principe : informer, ne pas imposer.** L'outil n'interprète pas les arrêtés. Il :
  - affiche le niveau de gravité, le texte des usages qui concernent les particuliers et le lien vers le PDF de l'arrêté ;
  - envoie une notification à chaque nouvel arrêté, changement de niveau ou fin de validité ;
  - laisse l'utilisateur décider d'appliquer ou non une restriction. S'il l'applique, l'utilisateur saisit lui-même les créneaux interdits, que la planification respecte.
- Une réponse 409 de l'API est affichée comme un statut inconnu, pas comme une absence de restriction.

## Conséquences
- Coût nul pour l'usage personnel : une dizaine de requêtes par jour, très loin de la limite gratuite d'Open-Meteo (10 000 par jour).
- **Pour un produit commercial**, la licence gratuite d'Open-Meteo ne suffit plus. Il faudra alors prendre l'abonnement commercial ou héberger Open-Meteo soi-même (logiciel libre).
- L'API VigiEau est sur un domaine `beta.gouv.fr`, donc susceptible de changer : une couche remplaçable l'isole aussi.
- Les tests utilisent des réponses enregistrées (Open-Meteo, VigiEau) comme doublures. Les calculs et les règles d'alerte sont développés en TDD.
