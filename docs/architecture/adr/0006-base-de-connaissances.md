# ADR-0006 : Base de connaissances

- **Statut** : accepté
- **Date** : 2026-11-17
- **Point de la todo-list** : 5

## Contexte
La planification a besoin de connaître chaque plante : besoins en eau, rusticité, calendrier, entretien, et impact sur la faune et la flore locales. Ces informations doivent être fiables, disponibles hors ligne (ADR-0002), et provenir d'une recherche web faite par l'IA (ADR-0003, usage A).

## Options envisagées
1. **Toutes les informations portées par chaque entité** : doublons, et incohérences entre plusieurs plantes de la même espèce.
2. **Une fiche de connaissances partagée, plus des ajustements par entité.**
3. **Recherche web sans restriction** : large, mais de fiabilité inégale.
4. **Recherche web limitée à une liste de sites de confiance.**

## Décision
Options 2 et 4.

### Deux niveaux d'information
- **La fiche de connaissances**, partagée par toutes les entités de la même espèce ou variété. L'ensemble des fiches constitue la **base de connaissances hors ligne**, stockée sur le serveur.
- **L'entité** : les données propres à un individu (âge, emplacement, état, mode d'arrosage). Elle peut **modifier** n'importe quelle valeur de sa fiche (ex. un pot abrité est moins rustique).

### Contenu d'une fiche
| Bloc | Contenu |
|---|---|
| Identification | Nom scientifique (identifiant unique), noms courants, famille, variété |
| Rusticité | Température minimale supportée (seuil de gel propre à l'espèce, voir ADR-0005) |
| Eau | Besoin, sensibilité à l'excès, type d'eau |
| Exposition et sol | Ensoleillement, type de sol, pH |
| Calendrier | Semis, plantation, floraison, récolte, taille, repos |
| Entretien | Taille, apport d'engrais, rempotage |
| Santé | Maladies et ravageurs courants |
| Faune et flore locales | Espèce locale ou exotique, envahissante, intérêt pour les pollinisateurs, **accueil de la faune** (nidification, refuge, nourriture) |
| Traçabilité | Sources (adresse et date de consultation), date de mise à jour, champs modifiés à la main |

### Sources
- **Identification de l'espèce, sans IA** : le référentiel national des espèces TAXREF (INPN, Muséum national d'Histoire naturelle). Son API publique a été testée le 2026-11-17 ([doc](https://taxref.mnhn.fr/taxref-web/api/doc)) :
  - Base `https://taxref.mnhn.fr/api/`, sans clé d'API. L'en-tête **`Accept: application/hal+json;version=1` est obligatoire** (sinon erreur 406).
  - `GET taxa/fuzzyMatch?term=<nom scientifique>` renvoie des taxons avec `id` (cd_nom), `referenceId` (cd_ref, le nom de référence), `scientificName`, `referenceName`, `frenchVernacularName`, `familyName`, `rankId`, et un statut biogéographique par territoire : `fr` pour la France métropolitaine. La recherche par nom courant (« Citronnier ») ne renvoie rien : l'utilisateur ou l'IA doit fournir le nom scientifique.
  - Le champ `fr` répond directement à la question « locale ou pas » : `P` présent (indigène), `I` introduit, **`J` introduit envahissant**, `M` domestique ou cultivé, `A` absent… Testé : *Robinia pseudoacacia* donne `J`, *Citrus limon* donne `M`. La signification exacte des codes reste à confirmer dans la doc.
  - `GET taxa/{referenceId}/status/lines` renvoie les statuts détaillés : listes rouges mondiale, nationale et **régionales**, statut biogéographique, réglementations sur les espèces envahissantes. Les statuts régionaux permettent d'adapter les informations à la région du jardin.
  - Les cultivars sont peu couverts (pour TAXREF, l'exhaustivité sur les plantes cultivées n'est pas une priorité). La variété horticole reste une donnée de la fiche, enrichie par la recherche web.
  - Licence : Licence Ouverte (Etalab), avec citation obligatoire de TAXREF et de sa version (le champ `taxrefVersion` est fourni). Une nouvelle version paraît chaque année.
- **Recherche web de l'IA limitée aux sites d'une liste de confiance** (filtrage par domaine côté API). La liste est modifiable dans la configuration. Liste initiale : INPN, Tela Botanica, Société nationale d'horticulture de France, RHS, centre de ressources sur les espèces exotiques envahissantes.

### Validation
- Une fiche générée par l'IA reste en **brouillon** jusqu'à sa relecture et sa validation par l'utilisateur.
- **Une entité n'entre dans la planification qu'une fois sa fiche validée.** Avant cela, elle reste au statut « En cours de création ».

### Protection de la faune
La planification tient compte des périodes sensibles. Par exemple, elle ne propose pas de taille de haie ou d'arbre pendant la nidification, et prévient si l'utilisateur en programme une. Comme pour les restrictions d'eau, l'outil **informe et ne bloque pas** (ADR-0005). La période par défaut doit être confirmée à partir des recommandations de l'Office français de la biodiversité (OFB). Elle est réglable dans la configuration.

## Conséquences
- Le modèle de données (point 11) doit gérer :
  - les fiches et leurs états (brouillon, validée) ;
  - le lien entre entité et fiche ;
  - les valeurs modifiées par entité ;
  - les sources.
- L'enrichissement par l'IA produit une fiche au format imposé et validé par le code, avec les sources obligatoires.
- Le seuil de gel d'une alerte est choisi dans cet ordre : la valeur modifiée sur l'entité, puis celle de la fiche, puis le seuil global de la configuration.
- Parcours d'identification :
  1. L'utilisateur saisit un nom, courant ou scientifique.
  2. Si besoin, l'IA propose le nom scientifique.
  3. TAXREF (`fuzzyMatch`) confirme et normalise ce nom.
  4. L'utilisateur choisit parmi les candidats.
  5. La fiche est rattachée au `referenceId` TAXREF, et les synonymes pointent vers la même fiche.
- La mention « Source : TAXREF vXX, MNHN » apparaît sur les fiches concernées.
- Les doublures de test couvrent l'IA et TAXREF (réponses enregistrées).
