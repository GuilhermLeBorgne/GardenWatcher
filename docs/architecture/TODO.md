# Todo-list des décisions d'architecture

Point de départ : un serveur central auto-hébergé chez le client (Raspberry Pi) relié à Internet. Il envoie des messages via une messagerie et expose une API utilisée par une application web.

Chaque point se clôt par un ADR dans `adr/` (modèle : `adr/0000-template.md`). L'ordre des points suit leurs dépendances.

## A. Cadrage (bloque tout le reste)

- [x] **1. Profil de déploiement** → [ADR-0001](adr/0001-profil-de-deploiement.md) : usage personnel, ou produit installé chez plusieurs clients ? Combien d'utilisateurs par serveur ? Quel niveau technique chez le client (installation, mises à jour) ?
- [ ] **2. Contraintes matérielles** : modèle de Pi (4 ou 5, RAM), stockage (carte SD fragile ou SSD USB), comportement en cas de coupure de courant ou d'Internet.
  - *Acquis* : pas de Pi pour l'instant. Le développement et les tests se font sur un ancien PC sous Ubuntu (x86, sans carte graphique). Le Pi (ARM) reste la cible à terme : les dépendances choisies doivent fonctionner sur les deux architectures. Sans GPU, un LLM local n'est pas envisageable sur ce serveur non plus.
- [x] **3. Exigences hors ligne** → [ADR-0002](adr/0002-fonctionnement-hors-ligne.md) : qu'est-ce qui doit marcher sans Internet (consultation, calendrier, notifications locales) et qu'est-ce qui peut se dégrader (IA, météo, recherche web) ?

## B. Briques externes

- [x] **4. IA / LLM** → [ADR-0003](adr/0003-usage-de-l-ia.md) : un LLM local n'est pas réaliste sur un Pi pour du raisonnement. Une API cloud (ex. Claude) pose les questions de coût, de clé API (fournie par le client ou par nous ?) et de comportement quand l'API est indisponible.
- [x] **5. Connaissances** → [ADR-0006](adr/0006-base-de-connaissances.md) : quelles sources fiables (bases botaniques, INPN, RHS…) ? Faut-il un RAG ou une base structurée (fiches espèces) ? Comment stocker la base hors ligne (SQLite + extension vectorielle ?) ? Comment l'enrichir via une recherche web (étape 3 du userflow) ?
  - *Acquis (ADR-0003)* : pas de RAG en V1. Les fiches utiles sont envoyées directement à l'IA. L'enrichissement se fait par une recherche web côté API et produit une fiche espèce au format imposé, avec ses sources.
- [x] **6. Météo** → [ADR-0005](adr/0005-meteo-et-restrictions-d-eau.md) : quel fournisseur (Open-Meteo : gratuit, sans clé ; Météo-France…) ? Quel horizon (court et moyen terme), quelle fréquence de récupération, quel cache local ?
  - *Orientation* : la météo et les alertes passent par des API open data, interrogées quelques fois par jour, **sans IA**. Les alertes sont déclenchées par des règles codées. L'IA reçoit seulement un résumé déjà calculé lors de la planification quotidienne. Point à vérifier : la licence d'Open-Meteo est gratuite uniquement pour un usage non commercial, alors que les données de Météo-France sont sous Licence Ouverte (Etalab).
  - *Restrictions d'eau* : les restrictions préfectorales doivent être prises en compte pour l'arrosage. On évite de scraper les sites des préfectures : ce serait fragile, et chacun a un format différent. Source retenue : l'API publique VigiEau ([swagger](https://api.vigieau.beta.gouv.fr/swagger), spec JSON sur `/swagger-json`).
    - `GET /api/zones?lon=&lat=&profil=particulier&zoneType=…` renvoie les zones qui s'appliquent. Pour chacune : `niveauGravite` (`vigilance` | `alerte` | `alerte_renforcee` | `crise`), `arrete` (dates de validité et PDF), et `usages[]` (`thematique`, `nom`, `description`, `concerneParticulier`).
    - Les horaires ne sont pas structurés : ils apparaissent seulement dans le texte libre `usages[].description` (ex. « Arrosage interdit de 8h à 20h. »).
    - **Principe retenu : informer, ne pas imposer** (comme un GPS qui affiche la limite de vitesse sans brider la voiture). On n'interprète pas les arrêtés. Fonctionnalité **« Restriction d'eau »** :
      1. récupération quotidienne de l'arrêté en vigueur pour l'adresse du jardin ;
      2. notification à chaque nouvel arrêté, changement de niveau ou fin de validité ;
      3. affichage du niveau, du texte des usages qui concernent les particuliers et du lien vers le PDF ;
      4. l'utilisateur choisit d'appliquer ou non une restriction. S'il l'applique, il saisit lui-même les créneaux interdits, et la planification en tient compte.
    - `zoneType` dépend de la **source d'eau** utilisée : `AEP` pour le réseau d'eau potable, `SOU` pour un puits ou un forage, `SUP` pour un prélèvement en eau de surface. C'est un réglage du jardin.
    - Une réponse 409 ne veut **pas** dire « pas de restriction » (c'est précisé dans la doc de l'API) : il faut l'afficher comme un statut inconnu.
- [x] **7. Messagerie** → [ADR-0004](adr/0004-messagerie.md) : comparer
  - Telegram : Bot API gratuite. Le long polling évite d'ouvrir un port entrant sur le Pi.
  - WhatsApp : Business API Meta, payante, demande une vérification et une URL de webhook publique.
  - Signal, ntfy, e-mail.

  Préciser aussi le sens des échanges : notifications seules, ou échange bidirectionnel (valider une tâche ou interroger l'agent depuis la messagerie) ?

## C. Architecture du serveur

- [x] **8. Stack backend** → [ADR-0007](adr/0007-stack-backend.md) : quel langage ? Python/FastAPI colle bien à l'écosystème IA, Go ou Node consomment moins de ressources. Base de données (SQLite probablement suffisante), planificateur de tâches (cron interne, APScheduler…).
- [ ] **9. Découpage interne** : monolithe modulaire ou services séparés ? Modules pressentis : entités, tâches/calendrier, météo, agent IA, notifications, connaissances.
- [ ] **10. Moteur de planification** : règles déterministes (saisons, seuils météo), décisions de l'IA, ou approche hybride (règles + IA qui propose, utilisateur qui valide) ?
  - *Exigence* : si l'utilisateur a choisi d'appliquer une restriction d'eau, les arrosages sont placés dans les créneaux qu'il a déclarés autorisés. On choisit le meilleur grâce à la météo (le plus frais, sans vent, sans pluie prévue juste après). Ce calcul est fait par du code, sans IA. S'il n'existe aucun créneau autorisé, la tâche est reportée et l'utilisateur est prévenu.
  - *Exigence* : **modes d'arrosage, réglés par entité** :
    - **Simple** : un arrosage par jour, avec une quantité approximative par plante.
    - **Renforcé** : tient compte des horaires, de la météo (chaleur, vent, pluie) et des besoins de l'espèce.
    - **Expert** : optimisation maximale (plusieurs passages par jour, quantités calculées, données du sol, évaporation).
  - *Exigence* : **périodes sensibles pour la faune** (ADR-0006). La planification ne propose pas de taille de haie ou d'arbre pendant la nidification, et prévient l'utilisateur s'il en programme une, sans bloquer. La période par défaut est à confirmer d'après les recommandations de l'OFB, et réglable dans la configuration.
  - *Exigence* : une entité n'entre dans la planification qu'une fois sa fiche de connaissances validée (ADR-0006).
  - *Exigence* : **créneaux de disponibilité par utilisateur** (ex. « le soir en semaine, le matin le week-end »). Les tâches sont placées dans ces créneaux. Le choix du meilleur créneau combine la disponibilité, les besoins de l'entité, la météo et les éventuelles restrictions d'eau appliquées par l'utilisateur.
- [ ] **11. Modèle de données** : entités, relations, statuts (liste complète), tâches, journal/historique des interventions, géolocalisation pour la carte.
  - À intégrer : le mode d'arrosage de chaque entité, les créneaux de disponibilité de chaque utilisateur, la source d'eau du jardin, les restrictions d'eau appliquées (créneaux interdits saisis par l'utilisateur) et le mode de validation des tâches (ADR-0003).
- [ ] **12. API** : REST + OpenAPI, ou autre ? Authentification (mono-utilisateur ou comptes), versionnement.

## D. Accès, client web, exploitation

- [ ] **13. Accès distant** : comment l'application web atteint le Pi hors du réseau local (Tailscale, Cloudflare Tunnel, ouverture de port) ? HTTPS, sécurité.
- [ ] **14. Application web** : framework, application servie par le Pi ou hébergée ailleurs, PWA (mobile, hors ligne), carte (Leaflet sur fond satellite, ou plan du jardin dessiné).
  - *Acquis* : **TypeScript sans framework**, avec Tailwind pour le CSS et Leaflet pour la carte. Le but est d'éviter de dépendre d'un framework front qui change sans cesse. Il reste à décider : l'hébergement de l'application (servie par le serveur ou ailleurs), l'outil de compilation, la PWA et le fond de carte.
- [ ] **15. Qualité et tests** : frameworks de test, doublures pour la météo, le LLM et la messagerie (pour appliquer le TDD).
- [ ] **16. Déploiement et exploitation** : packaging (Docker Compose ou image Pi), mises à jour, sauvegardes des données, gestion des secrets, logs et supervision.

## Idées pour plus tard

- **Actions depuis Telegram** : valider, reporter ou marquer une tâche comme faite directement depuis la messagerie. Ajouter aussi un canal de secours pour les alertes importantes (voir ADR-0004).
- **Suivi de soins** : suivi rapproché d'une plante en difficulté (journal d'évolution, photos, traitements, rappels fréquents). Probablement déclenché par le statut « Malade ». C'est une fonctionnalité à part entière, au-delà de l'arrosage.
- **Conversation avec l'IA** : poser des questions sur le jardin, via la messagerie ou l'application (exclue de la V1, voir ADR-0003).
- **Offre SaaS** : clé d'API d'IA gérée par nous, dans une offre payante ou pro.
- **Surveillance externe du serveur** : le serveur de l'utilisateur contacte régulièrement un serveur externe. Si ce dernier n'a plus de nouvelles depuis X temps, il prévient l'utilisateur via une messagerie. Candidat au « mode payant ».

## Couverture de la spec (README)

| Fonctionnalité | Points |
|---|---|
| Carte des entités et statuts | 11, 14 |
| Météo (arrosage, plantation, récolte) | 6, 10 |
| Base de connaissances hors ligne | 3, 5 |
| Recherche d'infos sur Internet | 4, 5 |
| Notifications (tâches, alertes météo) | 7, 10 |
| Calendrier des tâches (CRUD, validation) | 11, 12, 14 |
| Statuts des entités | 11 |
