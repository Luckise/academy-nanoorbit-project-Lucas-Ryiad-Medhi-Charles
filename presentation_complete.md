# Présentation complète — Projet NanoOrbit

> **Modules :** ALTN82 (Développement Mobile Android) · ALTN83 (Bases de Données Réparties)
> **Équipe :** Lucas · Ryiad · Mehdi
> **Campus :** EFREI Bordeaux — Cycle ingénieur S8 — 2025/2026
> **Niveau visé :** Excellence ★ sur les deux modules + bonus AR
> **Stack globale :** Oracle 23ai · PL/SQL · Docker · Flask (API REST) · Kotlin · Jetpack Compose · MVVM · Retrofit · Room · osmdroid · CameraX · WorkManager

Ce document retrace **tout** ce qui a été fait techniquement sur le projet NanoOrbit : le déroulé chronologique, les choix d'architecture, les problèmes rencontrés et la manière dont nous les avons résolus. Il s'appuie sur les sujets, les rapports (`rapport_bdd.md`, `rapport_android.md`, `rapport_ar.md`), le code source, les `Phase1.md` à `phase4.md`, et l'historique des commits Git.

---

## 1. Contexte et objectif du projet

NanoOrbit est une **startup fictive** opérant une constellation de **CubeSats** pour la surveillance climatique : déforestation, fonte des glaces, qualité de l'air, évolution du trait de côte.

Le projet fil rouge consiste à livrer **deux applications cohérentes** entre elles :

| Module | Livrable | Rôle |
|---|---|---|
| **ALTN83 — BDD réparties** | Schéma Oracle distribué + PL/SQL + optimisation | Stocker la constellation (satellites, orbites, fenêtres de communication, missions) sur 3 centres (Paris, Houston, Singapour) |
| **ALTN82 — Android** | Application Ground Control sur smartphone | Permettre aux opérateurs de superviser la constellation en temps réel (consultation, planification, cartographie, AR) |

La consigne pédagogique fondatrice est la **synergie** entre les deux modules : mêmes entités, mêmes identifiants, mêmes règles de gestion. Une `data class Satellite` Kotlin doit refléter exactement la table `SATELLITE` Oracle.

---

## 2. Vue d'ensemble du dépôt

```
nanoorbit/
├── README.md                         ← Description générale et synergie
├── CHANGELOG.md
├── docker-compose.yml                ← Oracle 23ai + API REST Python
├── Phase1.md → phase4.md             ← Détail technique des 4 phases ALTN83
├── rapport_bdd.md                    ← Rapport ALTN83
├── rapport_android.md                ← Rapport ALTN82
├── rapport_ar.md                     ← Rapport bonus AR
│
├── altn83-bdd/                       ← Module BDD : sujets, données CSV, scripts SQL
├── altn82-android/starter/           ← Module Android : projet Kotlin / Compose
│   └── app/src/main/java/fr/efrei/nanooribt/
│
├── api/                              ← API REST Flask qui ponte Android ↔ Oracle
│   ├── app.py
│   ├── Dockerfile
│   └── requirements.txt
├── api-mock/                         ← json-server pour développement offline
├── scripts/                          ← Auto-init de la base au docker compose up
├── dump/                             ← Dump Oracle d'initialisation
├── mcd/                              ← Schéma MCD (.webp)
└── screens/                          ← Captures résultats triggers + exercices PL/SQL
```

---

## 3. Déroulé chronologique du projet

L'historique Git, mois par mois, raconte l'arc complet de la production.

### Phase initiale — Avril (2025-04-01 à 2025-04-03)
- `80e09af` Initialisation du dépôt NanoOrbit S8 2025-2026
- `2a9a099` / `c697468` Première itération Phase 1 : MCD, dictionnaire des données, MLD
- `dd59b7d` Premier squelette du front Android Compose
- `3e0d057` Implémentation Phase 2 ALTN83 : DDL, DML, triggers
- `1d4fbac` / `443070a` Réponses aux questions de réflexion (architecture distribuée, fragmentation)

### Refonte design — Mi-avril (2025-04-13)
- `a043be8` **Refonte complète du design — style SpaceX minimaliste noir/blanc**
  - Première grande décision de design produit : abandonner le look Material3 par défaut pour un thème sombre/épuré inspiré de la communication SpaceX, plus crédible pour une startup spatiale.

### Phase 3 PL/SQL & Phase 4 — (2025-04-14)
- `3e56b31` / `8b4188a` Phase 3 PL/SQL : 16 exercices + package `pkg_nanoOrbit`
- `c67e0b8` Phase 4 complète : vues, CTE, fonctions analytiques, MERGE, index, EXPLAIN PLAN
- `3b3dad9` Synthèse Android intégrant les 4 phases

### Connexion Android ↔ Oracle — (2025-04-14)
- `a4ba996` **Auto-init de la base Oracle au `docker compose up`** (élimine 30 min d'install manuelle)
- `4437afb` **Connexion de l'app Android à Oracle via API REST Flask**
- `8db33a2` Refactor : extraction de `API_BASE_URL` en variable BuildConfig
- `3de661d` Refactor : lecture de `API_BASE_URL` depuis `.env` (DRY entre Docker et Android)

### Finalisation et conformité — Fin avril (2025-04-14 / 2025-04-28)
- `a60bbe2` **Alignement strict du jeu de données mock Android sur la référence ALTN83** — étape clé pour respecter la consigne de synergie.
- `b3af466` Fix : bouton "+" du `PlanningScreen` rendu fonctionnel (création réelle de fenêtre via ViewModel)
- `93f5bd4` Ajout des rapports ALTN82 et ALTN83
- `05608d0` **Bonus AR Sky-Track + rapport `rapport_ar.md`**

---

## 4. ALTN83 — Bases de Données Réparties

### 4.1 Phase 1 — Conception & architecture distribuée

**Livrables :** dictionnaire des données, MCD, MLD, note d'architecture distribuée.

**Travail effectué :**

- **Dictionnaire des données** complet sur 11 tables (`ORBITE`, `SATELLITE`, `INSTRUMENT`, `EMBARQUEMENT`, `CENTRE_CONTROLE`, `STATION_SOL`, `AFFECTATION_STATION`, `MISSION`, `FENETRE_COM`, `PARTICIPATION`, `HISTORIQUE_STATUT`). Chaque attribut classé en *Structure / Contrainte / Mécanisme procédural*, avec types Oracle cibles et règles de gestion associées.
- **MCD** (`mcd/mcd.webp`) avec entités-associations porteuses :
  - `EMBARQUEMENT(date_integration, etat_fonctionnement)` — RG-S04
  - `PARTICIPATION(role_satellite)` — RG-M03
  - relation ternaire `FENETRE_COM` ↔ `SATELLITE` ↔ `STATION_SOL`
- **MLD** : passage en 3NF, identification des index candidats (FK FENETRE_COM, statut SATELLITE).

**Questions de réflexion (Q1 à Q4)** — c'est là que se joue la note d'architecture :

| Question | Décision retenue |
|---|---|
| Q1 — Tables locales | `FENETRE_COM` et `HISTORIQUE_STATUT` (planification opérationnelle propre à chaque centre) |
| Q2 — Tables globales | `ORBITE`, `SATELLITE`, `INSTRUMENT`, `EMBARQUEMENT`, `CENTRE_CONTROLE`, `STATION_SOL`, `MISSION`, `PARTICIPATION` (réplique en lecture seule sur chaque nœud) |
| Q3 — Continuité Singapour | Fragmentation horizontale + réplique locale en lecture seule des tables globales |
| Q4 — Risques de cohérence | Scénario 1 : update concurrent du statut satellite → mitigation 2PC. Scénario 2 : insert sur réplique obsolète → lecture maître obligatoire pour l'insertion |

**Problème rencontré — la fragmentation :** initialement on hésitait entre fragmentation horizontale (par centre) et verticale (par type d'attribut). On a tranché horizontale parce que le sujet décrit explicitement Singapour comme un site qui doit pouvoir continuer à planifier ses propres fenêtres en autonomie — c'est la planification qui doit être locale, pas un sous-ensemble des colonnes.

### 4.2 Phase 2 — Schéma Oracle & Triggers

**Livrables :** DDL complet, DML chargé, 5 triggers (3 Niveau 1 + 2 bonus), script de contrôle.

**DDL — 11 tables** créées dans l'ordre des dépendances FK. Points marquants :
- `FENETRE_COM.id_fenetre` et `HISTORIQUE_STATUT.id_historique` en `GENERATED ALWAYS AS IDENTITY` (pas de séquence + trigger BEFORE INSERT à la main).
- `UNIQUE (altitude, inclinaison)` sur ORBITE pour RG-O02.
- `CHECK (duree BETWEEN 1 AND 900)` sur FENETRE_COM pour RG-F04.
- PK composites sur EMBARQUEMENT et PARTICIPATION.

**Triggers — 5 au total :**

| Trigger | Niveau | Règle | Erreur émise |
|---|---|---|---|
| **T1 — `trg_valider_fenetre`** | 1 | RG-S06 (satellite Désorbité) + RG-G03 (station Maintenance) | ORA-20101 / ORA-20102 |
| **T2 — `trg_no_chevauchement`** | 1 | RG-F02 (chevauchement satellite) + RG-F03 (chevauchement station) | ORA-20201 / ORA-20202 |
| **T3 — `trg_volume_realise`** | 1 | RG-F05 : force `volume_donnees = NULL` si statut ≠ Réalisée | correctif silencieux |
| **T4 — `trg_mission_terminee`** | 2 (bonus) | RG-M04 (mission Terminée bloquée) | ORA-20301 |
| **T5 — `trg_historique_statut`** | 2 (bonus) | AFTER UPDATE → trace `(:OLD, :NEW, SYSTIMESTAMP)` | aucune |

Captures dans `screens/t1.png` à `screens/t5.png`.

**Problème rencontré — table mutating sur T2 :** la première version naïve du trigger `trg_no_chevauchement` faisait un `SELECT ... FROM FENETRE_COM` directement dans un trigger row-level sur `FENETRE_COM`, ce qui déclenche `ORA-04091: table is mutating`. **Solution :** restructurer le trigger pour comparer chaque nouvelle ligne (`:NEW`) aux lignes déjà engagées (`COMMIT`) via une requête qui n'inclut pas la ligne en cours d'insertion, en utilisant `NVL(id_fenetre, -1) <> :NEW.id_fenetre`. Cas testés : insertion d'une fenêtre qui chevauche en partie (rejet), insertion contiguë (acceptée).

**Problème rencontré — encodage des accents :** les `CHECK (statut IN ('Opérationnel','En veille','Défaillant','Désorbité'))` posent un risque si le client SQL n'envoie pas de l'UTF-8. On a forcé l'encodage `NLS_LANG=AMERICAN_AMERICA.AL32UTF8` au niveau du conteneur Docker pour garantir la cohérence des comparaisons.

### 4.3 Phase 3 — PL/SQL & Package `pkg_nanoOrbit`

**Livrables :** 16 exercices PL/SQL (Paliers 1 à 5) + package `pkg_nanoOrbit` (Palier 6 bonus).

**Paliers couverts :**

| Palier | Exercices | Notion |
|---|---|---|
| 1 — Bloc anonyme | Ex 1-2 | DBMS_OUTPUT, SELECT INTO sur SAT-001 |
| 2 — Variables & types | Ex 3-4 | `%ROWTYPE`, `NVL` sur résolution NULL d'INS-AIS-01 |
| 3 — Structures de contrôle | Ex 5-7 | IF/ELSIF, CASE (vitesse orbitale), boucle FOR |
| 4 — Curseurs | Ex 8-11 | `SQL%ROWCOUNT`, Cursor FOR Loop, OPEN/FETCH/CLOSE, curseur paramétré |
| 5 — Procédures & fonctions | Ex 12-16 | NO_DATA_FOUND, RAISE_APPLICATION_ERROR, fonction de calcul de volume |

**Package bonus `pkg_nanoOrbit` (Palier 6) :**
- **SPEC** : type public `t_stats_satellite`, constantes (`c_duree_max_fenetre = 900`, `c_seuil_revision = 50`), 4 procédures + 3 fonctions.
- **BODY** : implémentation complète orchestrant les triggers de la Phase 2.
- **Scénario validation** : bloc enchaîné qui exécute les 7 sous-programmes avec ROLLBACK final pour ne pas polluer les données de référence.

**Problème rencontré — chaîne de dépendance déclencheur ↔ procédure :** la procédure `planifier_fenetre` du package devait, par construction, déclencher T1 (validation fenêtre) avant son `INSERT`. Une première version capturait `OTHERS` trop largement et masquait les `ORA-20101` du trigger. **Solution :** capturer explicitement les `RAISE_APPLICATION_ERROR` métier (`-20100..-20999`) et ne capturer `OTHERS` que pour le logging d'erreur technique inattendue.

### 4.4 Phase 4 — Exploitation avancée & Optimisation (Niveau 2 ★)

**Livrables :** vues + vue matérialisée, CTE, fonctions analytiques, MERGE, index + EXPLAIN PLAN, rapport de pilotage intégral.

| Section | Contenu |
|---|---|
| **L4-A — Vues** | `v_satellites_operationnels`, `v_fenetres_detail`, `v_stats_missions` (`LISTAGG`), `mv_volumes_mensuels` (matérialisée, `BUILD IMMEDIATE REFRESH ON DEMAND`) |
| **L4-B — CTE & sous-requêtes** | CTE simple, multiples avec `RANK()`, **récursive** (hiérarchie centre → station → fenêtres), scalaire, corrélée, `NOT EXISTS` |
| **L4-C — Analytiques & MERGE** | `ROW_NUMBER`/`RANK`/`DENSE_RANK`, `LAG`/`LEAD`, `SUM OVER ... ROWS UNBOUNDED PRECEDING`, moyenne mobile sur 3 fenêtres, `MERGE INTO SATELLITE` (flux IoT), `MERGE INTO AFFECTATION_STATION` (sync configuration) |
| **L4-D — Index & EXPLAIN PLAN** | 6 index (FK + statut + composite + fonctionnel `TRUNC(datetime_debut, 'MM')`) ; EXPLAIN PLAN avant/après ; démonstration `ALTER INDEX ... INVISIBLE/VISIBLE` |
| **Rapport de pilotage** | Requête finale combinant 3 CTE + vue matérialisée + LAG/RANK/SUM OVER |

**Problème rencontré — performance du reporting :** la requête de reporting mensuel (4 jointures + GROUP BY) faisait `TABLE ACCESS FULL` sur FENETRE_COM. **Solution :** index fonctionnel `idx_fenetre_mois ON FENETRE_COM(TRUNC(datetime_debut, 'MM'))` qui transforme le scan en `INDEX RANGE SCAN`. Démontré dans `phase4.md` Ex 18 avec le plan d'exécution avant/après.

**Problème rencontré — choix vue vs vue matérialisée :** `v_volumes_mensuels` est appelée fréquemment par le tableau de bord et fait plusieurs agrégats coûteux. La vue simple a été conservée pour la souplesse, et une **vue matérialisée** `mv_volumes_mensuels` (refresh on demand) a été ajoutée pour les usages reporting. Le rapport documente le compromis : vue normale = données fraîches mais lent, vue matérialisée = rapide mais à rafraîchir explicitement.

### 4.5 Récapitulatif des règles de gestion couvertes

| Règle | Mécanisme implémenté |
|---|---|
| RG-S01 (id satellite unique) | PRIMARY KEY |
| RG-S04 (instrument multi-satellites) | PK composite EMBARQUEMENT |
| RG-S06 (désorbité bloqué) | Trigger T1 + T4 |
| RG-O02 (unicité altitude+inclinaison) | UNIQUE constraint |
| RG-M03 (rôle satellite-mission) | PK composite PARTICIPATION |
| RG-M04 (mission terminée bloquée) | Trigger T4 |
| RG-F02 (chevauchement satellite) | Trigger T2 |
| RG-F03 (chevauchement station) | Trigger T2 |
| RG-F04 (durée 1–900s) | CHECK + validation côté Android |
| RG-F05 (volume NULL si pas Réalisée) | Trigger T3 |
| RG-G03 (station maintenance bloquée) | Trigger T1 |

---

## 5. ALTN82 — Application Android Ground Control

### 5.1 Phase 1 — Interface & Données (Socle)

**Livrables L1-A à L1-D :**

- **`Models.kt` (L1-A)** — data classes Kotlin strictement alignées sur les tables Oracle : `Satellite`, `Orbite`, `Instrument`, `FenetreCom`, `StationSol`, `Mission`. Enums **miroir des `CHECK` Oracle** : `StatutSatellite`, `FormatCubeSat`, `StatutFenetre`, `TypeOrbite`. Champs nullables exactement aux mêmes endroits que dans le DDL (`dateLancement`, `masse`, `resolution`, `volumeDonnees`, `dateFin`, `zoneGeoCible`).
- **`MockData.kt` (L1-B)** — jeu de données aligné sur le CSV de référence ALTN83 : 5 satellites (dont SAT-005 `DESORBITE`), 3 orbites (2 SSO + 1 LEO), 4 instruments, 5 fenêtres, 3 stations.
- **`Components.kt` (L1-C)** — composants Compose réutilisables : `SatelliteCard` (pastille statut + carte grisée pour SAT-005), `StatusBadge` (chip Material3 colorée), `FenetreCard`, `InstrumentItem`. Chaque composant a son `@Preview`.
- **`DashboardScreen.kt` (L1-D)** — `LazyColumn` filtrable, barre de recherche, compteur "{X}/{N} satellites opérationnels", désactivation visuelle des satellites désorbités.

### 5.2 Phase 2 — Architecture MVVM & API REST (Socle)

**Réorganisation 3 couches : Models → ViewModel → Vue.**

**`NanoOrbitViewModel.kt` (L2-A)** expose des `StateFlow` :
- `isLoading`, `errorMessage`, `searchQuery`, `selectedStatut`
- `filteredSatellites` calculé via **`combine()`** sur les flux de filtres → testabilité côté ViewModel sans toucher au Composable
- `isOffline` pour la bannière hors-ligne
- Factory `ViewModelProvider.Factory` pour injecter le `Repository`

**`NanoOrbitApi.kt` + `NanoOrbitRepository.kt` (L2-B)** :
- Interface Retrofit (5 endpoints `/satellites`, `/satellites/{id}/instruments`, `/satellites/{id}/missions`, `/fenetres`, `/stations`)
- DTOs séparés (`SatelliteApiResponse`, etc.) pour découpler le réseau du domaine
- Repository qui convertit DTO → Entity → Domain et expose des `Flow<List<X>>` issus de Room

**Validation RG-F04 (L2-D) :** `NanoOrbitRepository.validateFenetreDuree(duree)` retourne le message d'erreur si `duree !in 1..900`. C'est l'exact miroir du `CHECK` Oracle, exécuté côté client pour donner un retour immédiat avant l'aller-retour réseau.

### 5.3 Phase 3 — Fonctionnalités avancées (Excellence ★)

**Navigation Compose (L3-A)** — `Routes.kt` + `MainActivity.kt`
- `sealed class Screen` avec 4 routes : `dashboard`, `detail/{satelliteId}`, `planning`, `map` (puis `ar` ajouté en bonus)
- `NavHost` central avec transitions `fadeIn` / `slideInVertically`
- `BottomNavigationBar` masquée sur DetailScreen via `if (currentRoute != null && !currentRoute.startsWith("detail"))`

**DetailScreen (L3-B)** — fiche satellite avec 5 sections : Statut, Télémétrie, Instruments embarqués, Missions actives, Bouton "Signaler une anomalie" → dialog.

**PlanningScreen (L3-C)** — sélecteur de station, liste triée par `datetime_debut`, validation côté client RG-F04 + miroir RG-S06 (satellite non désorbité), bouton "+" qui appelle `viewModel.addFenetre()`.

**Persistance Room — Cache-First (L3-D)** — c'est l'apport pédagogique central :
- `SatelliteEntity`, `FenetreEntity` avec `lastUpdated: Long`
- DAOs exposant `getAllSatellites(): Flow<List<SatelliteEntity>>` (Flow réactif → l'UI se met à jour automatiquement)
- `getSatellitesFlow()` lit Room en premier, `refreshSatellites()` appelle l'API et met à jour le cache
- Bannière "Mode hors-ligne" affichée quand `isOffline = true`
- **Lien explicite avec ALTN83 Q3** commenté dans `NanoOrbitRepository.kt` : c'est l'équivalent mobile de la "réplique locale en lecture seule" de la fragmentation horizontale Phase 1 ALTN83

**MapScreen — osmdroid (L3-E)**
- Initialisation osmdroid dans `MainActivity.onCreate()` via `Configuration.getInstance().load(this, PreferenceManager.getDefaultSharedPreferences(this))`
- Marqueurs colorés par état de station (Active/Maintenance/Hors service)
- Infobulle au clic, FAB "Me localiser", calcul de distance

### 5.4 Bonus L3-F — Notifications WorkManager

`FenetreWorker.kt` :
- `CoroutineWorker` qui interroge Room pour les fenêtres `PLANIFIEE` dans `[0, 15] minutes` (`ChronoUnit.MINUTES.between(now, dt) in 0..15`)
- `PeriodicWorkRequest` toutes les 15 minutes
- Canal `nanoorbit_passages` avec `IMPORTANCE_HIGH`
- Notification : titre "Passage imminent : {idSatellite}", contenu "Station {codeStation} - Début dans quelques minutes"

### 5.5 Architecture finale du code Android

```
fr.efrei.nanooribt/
├── MainActivity.kt           ← entry point, NavHost, init Retrofit/Room/WorkManager
├── Routes.kt                 ← Screen sealed class + bottomNavItems
├── Models.kt                 ← data classes + enums (domaine pur)
├── MockData.kt               ← Phase 1 mock + fallback offline
├── Components.kt             ← SatelliteCard, StatusBadge, FenetreCard, InstrumentItem
├── DashboardScreen.kt        ← liste + recherche + filtres + compteur
├── DetailScreen.kt           ← fiche satellite (5 sections + dialog anomalie)
├── PlanningScreen.kt         ← fenêtres + validation RG-F04/RG-S06
├── MapScreen.kt              ← osmdroid + géolocalisation
├── ARScreen.kt               ← BONUS — overlay caméra Sky-Track
├── NanoOrbitViewModel.kt     ← MVVM, StateFlow, combine() pour le filtrage
├── NanoOrbitRepository.kt    ← Cache-First Room + Retrofit + lien ALTN83 Q3
├── NanoOrbitApi.kt           ← interface Retrofit + DTOs
├── CelesTrakApi.kt           ← BONUS — client TLE/OMM
├── OrbitalMechanics.kt       ← BONUS — propagateur képlérien
├── AppDatabase.kt            ← Room database
├── Entities.kt               ← SatelliteEntity, FenetreEntity + mappers
├── Daos.kt                   ← SatelliteDao, FenetreDao
└── FenetreWorker.kt          ← bonus L3-F WorkManager
```

Total : ~3 000 lignes Kotlin (hors thème UI), 19 fichiers source.

### 5.6 Problèmes Android rencontrés et solutions

**Problème — bouton "+" du PlanningScreen non fonctionnel.** À l'origine le bouton ouvrait un dialog mais l'INSERT n'était pas câblé. Commit `b3af466` : ajout de `viewModel.addFenetre(fenetre)` dans le repository, puis appel de `refreshFenetres()` pour mettre à jour le Flow Room. La correction a aussi nécessité la validation côté client de RG-F04 *avant* l'aller-retour pour éviter l'erreur ORA-20101 inutile.

**Problème — désynchronisation mock vs Oracle.** Au début, le `MockData.kt` Android avait des satellites avec des id `SAT-A`, `SAT-B` ne correspondant pas au CSV de référence ALTN83 (`SAT-001` à `SAT-005`). Casse la consigne de synergie. Commit `a60bbe2` : alignement strict, désormais SAT-005 est bien `DESORBITE` côté Android comme côté Oracle, ce qui permet de tester visuellement la grisaille de la card.

**Problème — `API_BASE_URL` codée en dur.** L'URL pointait vers `10.0.2.2:5000` (loopback émulateur Android). Pour tester sur device physique, il fallait éditer le code. Commits `8db33a2` puis `3de661d` : extraction en `BuildConfig.API_BASE_URL`, alimentée depuis `.env` à la racine du projet. Même fichier `.env` consommé par `docker-compose.yml` → DRY.

**Problème — `osmdroid` ne charge pas les tuiles sur émulateur.** Le user-agent par défaut était bloqué par les serveurs OSM. **Solution :** configurer `Configuration.getInstance().userAgentValue = packageName` avant tout `MapView`. Documenté dans `MainActivity.kt`.

**Problème — `combine()` du ViewModel re-émet en boucle.** La première version exposait `filteredSatellites` directement comme `Flow` sans `stateIn`, ce qui faisait recalculer le filtre à chaque collecte. **Solution :** wrapper en `stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), emptyList())`. C'est le patron canonique Compose.

---

## 6. Backend API REST — le pont entre les deux modules

### 6.1 Pourquoi un backend dédié

Le sujet ne l'imposait pas, mais sans pont, l'app Android ne pourrait jamais lire la vraie base Oracle. On a fait le choix d'un **petit serveur Flask Python** plutôt que JDBC direct ou json-server :

- **Flask + `oracledb`** : driver Oracle officiel, gère le pool de connexions et les types Oracle natifs (DATE, NUMBER, VARCHAR2).
- **Flask-CORS** : autorise les appels depuis l'émulateur Android.
- **Mapping enum côté serveur** : le code Python convertit `"Opérationnel" → "OPERATIONNEL"` pour matcher les enums Kotlin (qui ne supportent pas les espaces ni les accents). Évite à chaque appel client de faire la translation.

### 6.2 Endpoints exposés (`api/app.py`)

```
GET /satellites                             ← Dashboard
GET /satellites/<id>                        ← DetailScreen (cas "satellite seul")
GET /satellites/<id>/instruments            ← DetailScreen section Instruments
GET /satellites/<id>/missions               ← DetailScreen section Missions
GET /fenetres                               ← PlanningScreen
GET /stations                               ← MapScreen
GET /missions                               ← Dashboard secondaire
GET /health                                 ← healthcheck Docker (sondé par WorkManager)
```

### 6.3 Orchestration Docker

`docker-compose.yml` :
- `oracle-db` (gvenzl/oracle-free:23.7-slim-faststart) — exposé sur 1521
- `api` — image Flask + oracledb, exposé sur 5000, dépend de `oracle-db`
- volume `oracle_data` pour la persistance
- mount `./scripts/01_import_schema.sh` dans `/opt/oracle/scripts/startup` pour **auto-init** au premier `docker compose up` (commit `a4ba996` — gain énorme en setup time)

### 6.4 Problèmes backend rencontrés

**Problème — Oracle pas prêt quand l'API démarre.** L'`API` `depends_on: oracle-db` ne suffit pas : Docker n'attend que le démarrage du process, pas la disponibilité TNS. **Solution :** la fonction `wait_for_db(max_retries=30, delay=5)` dans `app.py` retente la connexion 30 fois avec 5 s d'intervalle avant de lancer Flask. Plus robuste qu'un `healthcheck` shell parce que ça reflète exactement ce que l'app va tenter au runtime.

**Problème — caractères accentués dans les CHECK.** Les statuts Oracle contiennent `Opérationnel` avec accent. Le mapping Python (`STATUT_SATELLITE_MAP`) convertit en `OPERATIONNEL` SANS accent côté API → match exact des enums Kotlin. Sans ce mapping, on aurait une `IllegalArgumentException` à `enumValueOf<StatutSatellite>("Opérationnel")`.

---

## 7. Bonus AR Sky-Track

C'est le livrable bonus le plus ambitieux : une **5ᵉ vue qui pointe le téléphone vers le ciel et affiche en surimpression les satellites visibles à l'œil nu** depuis la position GPS de l'utilisateur. Branche `Bonus-AR`, commit `05608d0`.

### 7.1 Pipeline temps réel (tick 500 ms)

1. **Lecture capteurs** — `rememberDeviceOrientation()` enregistre un `SensorEventListener` sur `TYPE_ROTATION_VECTOR`. La matrice 3×3 donne l'azimut/élévation de l'axe optique caméra arrière (`-Z` du repère device).
2. **Position observateur** — `rememberObserverLocation()` lit GPS+Network sur `LocationManager` (intervalle 5 s, 10 m). Fallback Paris si pas de fix.
3. **Propagation orbitale** — pour chaque `TleElement` reçu de **CelesTrak** (groupe `visual` ~150 satellites), `OrbitalMechanics.skyPositionFromElements(...)` calcule :
   - `n = meanMotion · 2π / 86400` → demi-grand axe `a = ³√(GM / n²)`
   - `M = M₀ + n · Δt`, résolution Kepler `M = E − e sin E` par 8 itérations de Newton-Raphson
   - perifocale → ECI (rotations 3-1-3 ω, i, Ω)
   - ECI → ECEF via temps sidéral Greenwich (IAU 1982 simplifiée)
   - ECEF → topocentrique ENU → (azimut, élévation, distance)
4. **Projection écran** — pour chaque satellite : si l'écart angulaire avec l'axe caméra rentre dans HFOV 65° / VFOV 50°, on dessine un marqueur. Sinon, indicateur hors-champ sur le bord dans la direction du satellite.
5. **Rendu** — `Canvas` Compose, halo + anneau coloré + texte (NORAD ID, élévation, distance). Tap dans 70 px → `SatelliteInfoCard` Material3.

### 7.2 Architecture

| Fichier | Rôle |
|---|---|
| `ARScreen.kt` | UI Compose, permission caméra, overlay marqueurs, fiche détail |
| `CelesTrakApi.kt` | Retrofit → `https://celestrak.org/NORAD/elements/gp.php`, DTO OMM JSON |
| `OrbitalMechanics.kt` | Propagateur képlérien + transformations ECI/ECEF/ENU |
| `MainActivity.kt` | Route `Screen.AR` ajoutée au NavHost |
| `Routes.kt` | `Screen.AR` (icône `Star`) ajouté à `bottomNavItems` |
| `AndroidManifest.xml` | Permission CAMERA + uses-feature non-required |
| `libs.versions.toml`+ `build.gradle.kts` | CameraX 1.3.4 (4 artefacts) |

### 7.3 Choix de design notables

- **Groupe `visual` plutôt que `active`** : ~150 objets → overlay lisible. `active` ferait ~10 000 objets → écran noyé.
- **`TYPE_ROTATION_VECTOR` plutôt que fusion manuelle accéléromètre+magnétomètre** : Android fait déjà la fusion (gyroscope inclus si dispo), latence et jitter bien meilleurs.
- **Tick 500 ms** : un satellite LEO bouge ~0,5°/s dans le ciel ; rafraîchir à 2 Hz est imperceptible et économise la batterie.
- **HFOV/VFOV en dur 65°/50°** : valeurs typiques smartphone. Amélioration future = lire `CameraCharacteristics`.
- **Couleur stable par satellite** : hash FNV-1a du nom mappé sur palette de 6 couleurs → même couleur à travers les recompositions sans table d'état.

### 7.4 Limitations connues (assumées dans le rapport)

- **Pas de SGP4** : précision dégradée >quelques jours depuis l'époque OMM. Acceptable parce que CelesTrak met à jour les OMM toutes les 8 h et que l'app les retéléverse à chaque ouverture.
- **Pas de filtrage par éclairage solaire** : on affiche aussi les satellites "théoriquement présents mais invisibles de jour".
- **Fallback Paris** si pas de GPS — positions fausses jusqu'au fix.
- **Pas de calibration boussole** — l'utilisateur doit faire un "∞" avec son téléphone (geste Android standard).

### 7.5 Problèmes AR rencontrés

**Problème — équation de Kepler divergente pour `e ≥ 0,1`.** La résolution de `M = E − e sin E` par Newton-Raphson démarrée à `E = M` divergeait pour les orbites un peu excentriques. **Solution :** initialiser `E = M + e · sin(M)` (correction du premier ordre) et limiter à 8 itérations max. Suffisant pour `e < 0,3` ce qui couvre 100 % des satellites du groupe `visual`.

**Problème — recomposition compose lourde.** Recalculer 150 satellites toutes les 500 ms saturait le main thread. **Solution :** enchâsser la propagation dans `LaunchedEffect(tick)` qui s'exécute sur `Dispatchers.Default`, et n'exposer à Compose que la liste filtrée des marqueurs visibles via `derivedStateOf`.

**Problème — permission caméra refusée.** Premier passage : `onPermissionResult: false` → écran noir sans message. **Solution :** `PermissionRationale` Compose qui explique pourquoi la caméra est requise et propose de relancer la demande ou de revenir en arrière. La géolocalisation reste optionnelle (fallback Paris).

---

## 8. Synergie ALTN82 ↔ ALTN83 — la consigne fondatrice

C'est le critère de notation principal : les deux modules doivent être **strictement cohérents**.

| Point de cohérence | ALTN83 — Oracle | ALTN82 — Android |
|---|---|---|
| **Modèles de données** | Table `SATELLITE` avec types Oracle | `data class Satellite` Kotlin avec mêmes champs et mêmes nullabilité |
| **Enums** | `CHECK (statut IN (...))` | `enum class StatutSatellite` |
| **Règle RG-F04** | `CHECK (duree BETWEEN 1 AND 900)` | `validateFenetreDuree()` avant INSERT |
| **Règle RG-S06** | Trigger T1 (ORA-20101) | Card grisée + interaction désactivée + message UI miroir |
| **Mode hors-ligne (Q3 Phase 1)** | Fragmentation horizontale + réplique locale en lecture seule | Room Cache-First + bannière "Mode hors-ligne" — **commenté explicitement dans `NanoOrbitRepository.getSatellitesFlow()` comme étant la réponse mobile à la Q3 ALTN83** |
| **Identifiants** | `SAT-001..SAT-005`, `GS-TLS-01`, `MSN-DEF-2022` | Mêmes IDs dans `MockData.kt` et reçus via API |
| **Cas limites** | SAT-005 désorbité, GS-SGP-01 maintenance, MSN-DEF-2022 terminée | Visibles à l'œil nu dans l'app — SAT-005 grisé, GS-SGP-01 marqueur orange sur Map |

**Décision clé :** chaque table Oracle a un commentaire de correspondance dans le data class Kotlin (`// Table SATELLITE Oracle`, `// FK - VARCHAR2(20)`). Le rapport Android pointe explicitement les fichiers où la synergie est implémentée pour faciliter la correction.

---

## 9. Problèmes transverses et leçons retenues

### 9.1 Problème — synchronisation mock ↔ référence ALTN83
Les premiers commits Android utilisaient un mock fait "à la main" sans regarder le CSV ALTN83. Au moment de tester la synergie, écart partout : ids différents, statuts manquants, dates incohérentes. **Leçon :** dans un projet à deux modules, le **jeu de données de référence** doit être la source unique de vérité. Le commit `a60bbe2` a été un alignement complet ligne par ligne.

### 9.2 Problème — gestion des branches et conflits
L'historique Git montre 3 branches principales : `main`, `Luckise/feature/phase-X`, `Luckise/bmr/feature/database-link`, `Bonus-AR`. Conflits sur :
- `MainActivity.kt` (NavHost étendu sur deux branches en parallèle)
- `MockData.kt` (alignement vs nouvelles entrées)
- `build.gradle.kts` (Retrofit vs CameraX)

**Solution :** PR fréquentes, `git merge --no-ff` pour garder la trace des branches dans l'historique (visible dans `git log --graph`), et un commit dédié `25ad43c` "merge: resolve conflicts with dev, keep database-link changes" pour documenter explicitement la décision de résolution.

### 9.3 Problème — environnements hétérogènes (Windows / émulateur / device)
- Émulateur Android = `10.0.2.2` pour atteindre `localhost`
- Device physique = IP du PC sur le LAN
- Docker = `oracle-db` (DNS interne)

**Solution :** `.env` à la racine, lu à la fois par Docker (`docker-compose.yml`) et par Android (`BuildConfig.API_BASE_URL`). Un seul endroit à éditer.

### 9.4 Problème — auto-init Oracle long et fragile
Premier `docker compose up` = base vide, il fallait se connecter en sqlplus, créer le user, exécuter DDL, exécuter DML… 30 minutes par développeur. Commit `a4ba996` : `scripts/01_import_schema.sh` mounté dans `/opt/oracle/scripts/startup` qui s'exécute automatiquement au premier démarrage du conteneur. Setup divisé par 30.

### 9.5 Problème — accents et encoding
`'Opérationnel'` dans les CHECK Oracle, en Compose via `enum class StatutSatellite { OPERATIONNEL, ... }` (pas d'accent autorisé en Kotlin). Mapping côté API Python pour faire la traduction. Sans ce mapping, le client fait `enumValueOf("Opérationnel")` et crashe.

---

## 10. Synthèse — ce qui a été produit

| Module | Niveau visé | Livrables réalisés |
|---|---|---|
| **ALTN83** | Niveau 2 ★ Excellence (80 pts socle + 45 pts bonus) | 4 phases : conception distribuée, schéma Oracle complet, 5 triggers (3 + 2 bonus), package PL/SQL `pkg_nanoOrbit`, exploitation avancée (vues, CTE récursive, analytiques, MERGE, EXPLAIN PLAN) |
| **ALTN82** | Excellence ★ (Phase 1+2+3 + bonus L3-F) | App Compose MVVM 4 écrans + AR, navigation, Cache-First Room, validation RG-F04/S06 client, osmdroid, WorkManager pour notifications |
| **Backend** | Hors sujet — décision projet | API Flask Python + `oracledb`, Docker compose orchestrant Oracle + API, auto-init schéma |
| **Bonus AR** | Hors sujet | 5ᵉ écran avec CameraX + propagateur képlérien + capteur rotation + CelesTrak TLE/OMM, ~600 lignes Kotlin |

**Volume de code produit :**
- ~3 000 lignes Kotlin (19 fichiers source)
- ~600 lignes SQL/PL/SQL (DDL + DML + 5 triggers + 16 exercices + package + Phase 4)
- ~300 lignes Python (API Flask)
- ~150 lignes YAML/Bash (Docker compose + scripts d'init)
- 4 rapports Markdown détaillés (`Phase1.md` à `phase4.md`) + 3 rapports de synthèse (`rapport_bdd.md`, `rapport_android.md`, `rapport_ar.md`)

---

## 11. Plan de présentation orale suggéré (15-20 min)

1. **Contexte (1 min)** — startup CubeSats, 2 modules, consigne synergie.
2. **Démo live ALTN83 (3 min)** — DDL + insert d'une fenêtre sur SAT-005 désorbité → ORA-20101. Vue `v_fenetres_detail`. Procédure `pkg_nanoOrbit.planifier_fenetre`.
3. **Démo live Android (5 min)** — Dashboard, recherche, DetailScreen, PlanningScreen avec validation RG-F04, MapScreen osmdroid, mode hors-ligne (couper le Wi-Fi pendant la démo).
4. **Démo live AR (3 min)** — pointer le téléphone vers le ciel, montrer un marqueur ISS, tap → fiche détail avec NORAD ID. Croisement avec Stellarium si possible.
5. **Architecture & synergie (3 min)** — schéma Oracle ↔ Kotlin ↔ API Flask. Pointer les commentaires "Lien ALTN83 Q3" dans `NanoOrbitRepository.kt`.
6. **Difficultés & solutions (3 min)** — table mutating sur T2, désynchronisation mock, équation de Kepler divergente, auto-init Docker.
7. **Bilan (1 min)** — niveaux atteints, ouvertures (SGP4, calibration FOV).

---

## 12. Annexes — pointeurs utiles

- **Captures d'écran des triggers et exercices :** `screens/t1.png` à `screens/t5.png`, `screens/q1_phase3.png` à `screens/q16_phase3.png`
- **MCD :** `mcd/mcd.webp`
- **Sujets PDF :** `altn83-bdd/sujets/` et `altn82-android/sujets/`
- **CSV de référence :** `altn83-bdd/donnees/ALTN83_NanoOrbit_*.csv`
- **Code source Android :** `altn82-android/starter/app/src/main/java/fr/efrei/nanooribt/`
- **API REST :** `api/app.py`
- **Orchestration :** `docker-compose.yml` à la racine
- **Branche AR :** `Bonus-AR`
