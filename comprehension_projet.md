# Compréhension du projet NanoOrbit

> Document de synthèse pour comprendre **ce qu'est le projet, ce qu'il faut produire, et ce qui a été fait**.
> Lecture autonome — pas besoin de connaître le sujet à l'avance.

---

## 1. C'est quoi NanoOrbit ?

**NanoOrbit est une startup fictive** inventée par l'EFREI pour servir de fil rouge à deux modules du semestre 8 :
- **ALTN83** — Bases de Données Réparties (Oracle)
- **ALTN82** — Développement Mobile Android (Kotlin)

L'histoire : NanoOrbit exploite une **constellation de CubeSats** (petits satellites cubiques) qui tournent en orbite basse pour faire de la surveillance climatique — déforestation, fonte des glaces, qualité de l'air, érosion côtière.

Pour piloter cette constellation, il faut deux choses :
1. **Une base de données** qui stocke les satellites, leurs orbites, les fenêtres de communication avec les stations au sol, les missions scientifiques. Cette base est **distribuée** sur 3 centres de contrôle dans le monde : Paris, Houston, Singapour.
2. **Une application mobile** que les opérateurs au sol utilisent pour superviser la constellation : voir les satellites en service, planifier les fenêtres de communication, voir les stations sur une carte.

**La consigne fondatrice** : ces deux livrables ne sont pas indépendants. Ils doivent être **strictement cohérents** entre eux. Une `data class Satellite` dans le code Kotlin doit refléter exactement la table `SATELLITE` Oracle. C'est le critère de notation principal qu'on appelle la **synergie**.

---

## 2. Les deux mondes du projet

### Le monde Oracle (ALTN83)

C'est le monde "serveur" : tables, triggers, PL/SQL, optimisation.

**Données métier modélisées :**
- **5 satellites** (`SAT-001` à `SAT-005`), dont un volontairement désorbité
- **3 orbites** différentes (2 SSO héliosynchrones + 1 LEO basse)
- **4 instruments scientifiques** embarqués sur les satellites
- **3 stations au sol** (Toulouse, Kiruna en Arctique, Singapour), dont une en maintenance
- **3 centres de contrôle** (Paris, Houston, Singapour)
- **3 missions** scientifiques, dont une terminée
- **5 fenêtres de communication** entre satellites et stations

**Règles métier clés** (qu'on appelle RG = règles de gestion) :
- Un satellite **désorbité** ne peut plus participer à rien (RG-S06)
- Une station **en maintenance** ne peut pas recevoir de fenêtre de communication (RG-G03)
- Une fenêtre de communication dure entre **1 et 900 secondes** (RG-F04)
- Pas de **chevauchement** : un satellite ne peut pas parler à deux stations en même temps, et une station ne peut pas parler à deux satellites en même temps (RG-F02, RG-F03)
- Une mission **terminée** ne peut plus accueillir de nouveau satellite (RG-M04)

### Le monde Android (ALTN82)

C'est le monde "client" : écrans, navigation, données locales, capteurs.

**Application mobile pour opérateur Ground Control** avec 4 écrans principaux + 1 bonus :
- **Dashboard** — liste de tous les satellites avec recherche et filtres
- **Detail** — fiche complète d'un satellite (instruments, missions, télémétrie)
- **Planning** — fenêtres de communication par station, avec création de nouvelles fenêtres
- **Map** — carte avec les stations au sol géolocalisées
- **AR** (bonus) — pointer le téléphone vers le ciel, voir les satellites en surimpression

### Le pont entre les deux

Pour que l'app Android lise vraiment la base Oracle, on a ajouté **une petite API REST Flask** en Python qui fait le lien. Tout est orchestré par **Docker Compose** : un seul `docker compose up` lance Oracle + l'API ensemble.

```
┌─────────────────┐         ┌──────────────┐         ┌──────────────────┐
│  Android App    │ ──HTTP─→│  API Flask   │ ──SQL──→│  Oracle 23ai     │
│  (Kotlin)       │         │  (Python)    │         │  (Docker)        │
└─────────────────┘         └──────────────┘         └──────────────────┘
       Room cache               oracledb                 11 tables + 5 triggers
       (offline)                Flask-CORS               + package PL/SQL
```

---

## 3. Ce qui était demandé (les sujets)

### ALTN83 — 4 phases progressives

| Phase | Ce qu'il faut produire | Pourquoi |
|---|---|---|
| **Phase 1** | Dictionnaire des données + MCD + MLD + note d'architecture distribuée | Concevoir le schéma sur papier avant de coder, et réfléchir à la **fragmentation** des données entre les 3 centres |
| **Phase 2** | Script DDL Oracle + DML d'insertion + triggers PL/SQL | Implémenter le schéma et les **règles métier qui ne tiennent pas dans un simple CHECK** |
| **Phase 3** | Exercices PL/SQL sur 5 paliers + package `pkg_nanoOrbit` | Apprendre la programmation procédurale Oracle : variables, curseurs, procédures, fonctions |
| **Phase 4** | Vues + CTE + fonctions analytiques + index + EXPLAIN PLAN | Optimisation et reporting de niveau "production" |

### ALTN82 — 3 phases + bonus

| Phase | Ce qu'il faut produire | Pourquoi |
|---|---|---|
| **Phase 1** | Modèles Kotlin + jeu de données mock + composants Compose + Dashboard | Maquette fonctionnelle avec données en dur |
| **Phase 2** | ViewModel MVVM + StateFlow + Retrofit + DTOs | Architecture propre et appel à l'API |
| **Phase 3** | Navigation + Detail + Planning + Map + Cache-First Room | Application complète multi-écrans avec mode hors-ligne |
| **Bonus L3-F** | Notifications WorkManager | Alerter l'opérateur d'un passage imminent |

### Niveau visé sur les deux modules

**Excellence ★** — c'est-à-dire socle + tous les bonus + lien explicite entre les deux modules.

---

## 4. Comment on a abordé chaque module

### ALTN83 — la base de données

**Phase 1 — Conception**

On a fait le dictionnaire des 11 tables (`ORBITE`, `SATELLITE`, `INSTRUMENT`, `EMBARQUEMENT`, `CENTRE_CONTROLE`, `STATION_SOL`, `AFFECTATION_STATION`, `MISSION`, `FENETRE_COM`, `PARTICIPATION`, `HISTORIQUE_STATUT`). Chaque attribut classé en *Structure / Contrainte / Mécanisme procédural*.

Le MCD (`mcd/mcd.webp`) montre les associations porteuses :
- `EMBARQUEMENT(date_integration, etat_fonctionnement)` — un instrument peut être monté sur plusieurs satellites
- `PARTICIPATION(role_satellite)` — un satellite peut participer à plusieurs missions

**La grande question de la Phase 1 : la fragmentation.** Le sujet demande comment Singapour peut continuer à planifier des fenêtres si le serveur central tombe. **Notre choix : fragmentation horizontale.** Chaque centre a sa propre table `FENETRE_COM` locale + une réplique en lecture seule des tables globales. Singapour reste autonome en cas de coupure réseau.

**Phase 2 — Schéma + triggers**

Le DDL est straight-forward : 11 `CREATE TABLE` avec FK, CHECK, UNIQUE, PK simples et composites.

Le travail intéressant ce sont les **5 triggers** :

| Trigger | Ce qu'il fait |
|---|---|
| **T1** | Bloque l'insertion d'une fenêtre si le satellite est désorbité ou la station en maintenance |
| **T2** | Bloque les chevauchements de fenêtres (sur le même satellite ou la même station) |
| **T3** | Force `volume_donnees = NULL` si la fenêtre n'est pas encore réalisée |
| **T4** | Bloque l'ajout d'un satellite à une mission terminée |
| **T5** | Trace toutes les modifications de statut de satellite dans une table d'historique |

**Phase 3 — PL/SQL**

16 exercices répartis sur 5 paliers progressifs : bloc anonyme → variables/types → if/case/for → curseurs → procédures et fonctions.

En bonus on a packagé tout ça dans `pkg_nanoOrbit` avec :
- 4 procédures (planifier une fenêtre, clôturer une fenêtre, affecter un satellite à une mission, mettre en révision)
- 3 fonctions (calculer un volume théorique, état de la constellation, stats par satellite)
- 1 type composite `t_stats_satellite`
- 3 constantes métier

L'élégance du package, c'est qu'il **orchestre les triggers de la Phase 2**. Quand on appelle `pkg_nanoOrbit.planifier_fenetre(...)`, l'INSERT déclenche T1 qui peut lever ORA-20101 si le satellite est désorbité — donc la même règle métier est garantie quel que soit le chemin d'écriture.

**Phase 4 — Optimisation**

Là c'est de l'Oracle "avancé" :
- 3 vues + 1 vue **matérialisée** (`mv_volumes_mensuels` rafraîchie à la demande)
- CTE simples, multiples, **récursive** (hiérarchie centre → station → fenêtre avec indentation)
- Sous-requêtes scalaires et corrélées
- Fonctions analytiques : `RANK`, `LAG`/`LEAD`, `SUM OVER ROWS UNBOUNDED PRECEDING`, moyennes mobiles
- `MERGE INTO` pour la synchronisation flux IoT et fichier de configuration
- 6 index stratégiques dont un **fonctionnel** sur `TRUNC(datetime_debut, 'MM')` pour le reporting mensuel
- `EXPLAIN PLAN` avant/après pour démontrer l'effet des index

### ALTN82 — l'application Android

**Phase 1 — UI avec données en dur**

On a créé les `data class` Kotlin **strictement alignées** sur les tables Oracle (mêmes noms de champs, mêmes nullabilités). Les enums Kotlin (`StatutSatellite`, `FormatCubeSat`, `StatutFenetre`, `TypeOrbite`) reflètent les `CHECK` Oracle.

Les composants Compose sont reusables : `SatelliteCard` avec pastille colorée selon le statut, `StatusBadge` chip Material3, `FenetreCard`, `InstrumentItem`. Chacun a son `@Preview`.

Le `DashboardScreen` est un `LazyColumn` avec recherche en temps réel et compteur "{X}/{N} satellites opérationnels".

**Phase 2 — MVVM + API**

L'application passe de "mock en dur" à **architecture propre** : Models → ViewModel → Vue.

Le `NanoOrbitViewModel` expose des `StateFlow` que la Vue collecte avec `collectAsStateWithLifecycle()`. Le filtrage est fait avec `combine()` sur les flux de filtres ce qui rend le ViewModel testable sans toucher au Composable.

`NanoOrbitApi` est une interface Retrofit avec 5 endpoints. Les DTOs sont séparés du domaine (on ne réutilise pas les `data class` métier sur le réseau) — c'est plus de code mais ça découple proprement.

**Phase 3 — App complète**

Navigation Compose avec `sealed class Screen` et 4 routes. Bottom bar masquée sur Detail (UX classique).

**Le morceau pédagogique central**, c'est le **Cache-First avec Room** :
- L'app lit toujours Room en premier (`Flow<List<SatelliteEntity>>` réactif)
- En parallèle, refresh API en arrière-plan qui met à jour Room
- Si l'API tombe : bannière "Mode hors-ligne" affichée mais l'app reste utilisable

**C'est ici que la synergie ALTN82/ALTN83 prend tout son sens** : le Cache-First côté mobile est l'**équivalent** de la fragmentation horizontale + réplique locale en lecture seule discutée à la Phase 1 ALTN83. Le commentaire dans `NanoOrbitRepository.kt` le dit explicitement.

`MapScreen` utilise **osmdroid** (OpenStreetMap) avec marqueurs colorés par état de station et géolocalisation.

**Bonus L3-F — Notifications**

`FenetreWorker` est un `CoroutineWorker` enregistré comme `PeriodicWorkRequest` toutes les 15 minutes. Il interroge Room pour trouver les fenêtres `PLANIFIEE` dans les 0 à 15 minutes à venir et envoie une notification système.

### Bonus AR Sky-Track

C'est un **5ᵉ écran** ajouté hors sujet : pointer le téléphone vers le ciel et voir les satellites visibles en surimpression sur le flux caméra.

**Comment ça marche, en bref :**

1. **CelesTrak** publie en JSON les éléments orbitaux (TLE/OMM) de ~150 satellites visibles à l'œil nu
2. Pour chacun, on **propage** l'orbite : équation de Kepler `M = E − e·sin(E)` résolue par Newton-Raphson, transformations ECI → ECEF → topocentrique → (azimut, élévation, distance)
3. On lit l'**orientation du téléphone** via le capteur `TYPE_ROTATION_VECTOR` (Android fait la fusion accéléromètre+magnétomètre+gyroscope)
4. On lit la **position GPS** de l'observateur
5. Pour chaque satellite, si l'écart angulaire avec l'axe optique caméra rentre dans le champ de vision (HFOV 65° / VFOV 50°), on dessine un marqueur sur le `Canvas` Compose superposé au `PreviewView` CameraX
6. Tap sur un marqueur → fiche détail avec NORAD ID, période, altitude, inclinaison

C'est un propagateur **képlérien à deux corps** (pas SGP4) — moins précis mais largement suffisant pour de l'AR éducative tant qu'on rafraîchit les TLE à chaque ouverture (CelesTrak les met à jour toutes les 8 h).

---

## 5. La synergie en pratique

C'est le critère de notation principal. Voici **point par point** ce qui rend les deux modules cohérents :

| Élément | ALTN83 (Oracle) | ALTN82 (Android) |
|---|---|---|
| **Modèle de données** | Table `SATELLITE` avec types Oracle | `data class Satellite` avec mêmes champs et nullabilités |
| **Énumérations** | `CHECK (statut IN ('Opérationnel', 'En veille', 'Défaillant', 'Désorbité'))` | `enum class StatutSatellite { OPERATIONNEL, EN_VEILLE, DEFAILLANT, DESORBITE }` |
| **RG-F04 (durée)** | `CHECK (duree BETWEEN 1 AND 900)` côté serveur | `validateFenetreDuree()` côté client, **avant** l'aller-retour réseau |
| **RG-S06 (désorbité)** | Trigger T1 lève `ORA-20101` | Card grisée + interaction désactivée + message UI miroir |
| **Mode hors-ligne** | Fragmentation horizontale + réplique locale (Q3 Phase 1) | Room Cache-First + bannière "Mode hors-ligne" |
| **Identifiants** | `SAT-001`..`SAT-005`, `GS-TLS-01`, `MSN-DEF-2022` | Mêmes IDs dans le mock Android et dans les réponses API |
| **Cas limites** | SAT-005 désorbité, GS-SGP-01 maintenance, MSN-DEF-2022 terminée | SAT-005 visiblement grisé, GS-SGP-01 marqueur orange sur Map |

Chaque `data class` Kotlin a un commentaire de correspondance vers la table Oracle dans le code. Le rapport Android pointe explicitement les fichiers où la synergie est implémentée.

---

## 6. Les problèmes vraiment intéressants qu'on a résolus

### Côté Oracle

**Table mutating (ORA-04091).** Le trigger T2 anti-chevauchement, première version, faisait un `SELECT ... FROM FENETRE_COM` dans un trigger row-level sur `FENETRE_COM`. Oracle interdit. **Solution :** comparer `:NEW` aux lignes déjà engagées via une requête qui exclut la ligne en cours d'insertion (`NVL(id_fenetre, -1) <> :NEW.id_fenetre`).

**Performance reporting.** La requête mensuelle (4 jointures + GROUP BY) faisait `TABLE ACCESS FULL`. **Solution :** index fonctionnel `idx_fenetre_mois ON FENETRE_COM(TRUNC(datetime_debut, 'MM'))` qui transforme le full scan en `INDEX RANGE SCAN`.

**Encodage des accents.** Les CHECK Oracle contiennent `'Opérationnel'` avec accent. Si le client SQL n'envoie pas du UTF-8 strict, le CHECK échoue silencieusement. **Solution :** `NLS_LANG=AMERICAN_AMERICA.AL32UTF8` sur le conteneur Docker.

### Côté Android

**Bouton "+" du Planning non câblé.** Le dialog s'ouvrait mais l'INSERT n'était pas appelé. **Solution :** `viewModel.addFenetre(fenetre)` + `refreshFenetres()` après succès, plus validation côté client de RG-F04 *avant* l'aller-retour pour éviter l'erreur ORA-20101 inutile.

**Désynchronisation mock vs Oracle.** Au début, le `MockData.kt` Android utilisait des IDs `SAT-A`/`SAT-B` qui ne matchaient pas le CSV de référence ALTN83. **Solution :** alignement strict ligne par ligne — désormais SAT-005 est bien `DESORBITE` côté Android comme côté Oracle.

**`combine()` qui re-émet en boucle.** Le ViewModel exposait `filteredSatellites` directement comme `Flow` sans `stateIn`, ce qui faisait recalculer le filtre à chaque collecte. **Solution :** `stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), emptyList())`.

**API_BASE_URL en dur.** L'URL pointait vers `10.0.2.2:5000` (loopback émulateur). Sur device physique, il fallait éditer le code à chaque test. **Solution :** extraction en `BuildConfig.API_BASE_URL` alimentée depuis un `.env` à la racine, partagé avec Docker.

### Côté backend / DevOps

**Oracle pas prêt quand l'API démarre.** `depends_on` n'attend que le démarrage du process, pas la disponibilité TNS. **Solution :** boucle `wait_for_db(max_retries=30, delay=5)` dans `app.py` qui retente la connexion 30 fois avant de lancer Flask.

**Setup Oracle de 30 minutes par dev.** Premier `docker compose up` = base vide, il fallait se connecter en sqlplus et exécuter manuellement DDL puis DML. **Solution :** `scripts/01_import_schema.sh` mounté dans `/opt/oracle/scripts/startup` qui s'exécute automatiquement au premier démarrage du conteneur.

**Mapping des accents Oracle ↔ Kotlin.** Les enums Kotlin n'autorisent ni accents ni espaces. **Solution :** mapping côté API Python (`STATUT_SATELLITE_MAP = {"Opérationnel": "OPERATIONNEL", ...}`) qui fait la traduction au moment du JSON.

### Côté AR

**Équation de Kepler divergente pour `e ≥ 0,1`.** Newton-Raphson démarré à `E = M` divergeait sur les orbites un peu excentriques. **Solution :** initialisation `E = M + e·sin(M)` (correction du premier ordre), maximum 8 itérations.

**Recomposition Compose lourde.** 150 satellites × propagation orbitale toutes les 500 ms saturait le main thread. **Solution :** `LaunchedEffect(tick)` sur `Dispatchers.Default` + `derivedStateOf` pour ne re-pousser à Compose que la liste des marqueurs visibles.

---

## 7. Ce qui a été produit au final

| Module | Niveau atteint | Détail |
|---|---|---|
| **ALTN83** | Niveau 2 ★ Excellence | 80 pts socle + 45 pts bonus. 4 phases complètes : conception distribuée, schéma + 5 triggers, PL/SQL + package `pkg_nanoOrbit`, optimisation avancée |
| **ALTN82** | Excellence ★ | Phase 1 + 2 + 3 + bonus L3-F notifications. App Compose 4 écrans + AR, MVVM, Cache-First Room, validation client/serveur miroir |
| **Backend** | Hors sujet | API Flask + `oracledb` + Docker compose orchestrant Oracle + API + auto-init schéma |
| **Bonus AR** | Hors sujet | 5ᵉ écran AR Sky-Track avec CameraX, propagateur képlérien, capteur rotation, CelesTrak TLE/OMM (~600 lignes Kotlin) |

**Volume de code :**
- ~3 000 lignes Kotlin (19 fichiers source)
- ~600 lignes SQL/PL/SQL
- ~300 lignes Python
- ~150 lignes YAML/Bash
- 4 rapports `Phase1.md` à `phase4.md` + 3 rapports de synthèse

**Documents de référence dans le dépôt :**
- `README.md` — description générale, structure, démarrage rapide
- `Phase1.md` à `phase4.md` — détail technique complet de chaque phase ALTN83
- `rapport_bdd.md` — synthèse ALTN83
- `rapport_android.md` — synthèse ALTN82
- `rapport_ar.md` — synthèse bonus AR
- `presentation_complete.md` — déroulé chronologique exhaustif
- `comprehension_projet.md` — ce document

---

## 8. Si tu ne dois retenir que 3 choses

1. **Le cœur du projet, c'est la synergie.** Deux livrables techniquement différents (Oracle vs Android) mais qui partagent les mêmes entités, les mêmes règles métier, les mêmes identifiants. La règle RG-F04 est validée à la fois par un `CHECK` Oracle et par une fonction Kotlin. Le mode hors-ligne mobile (Cache-First Room) est l'équivalent direct de la fragmentation horizontale ALTN83.

2. **Le projet va du concept à la production.** On part d'un MCD sur papier (Phase 1), on construit le schéma Oracle (Phase 2), on programme dedans (Phase 3), on l'optimise (Phase 4). En parallèle on construit une app mobile complète qui consomme cette base. Le bonus AR montre qu'on est allés plus loin que demandé.

3. **Les difficultés ont été techniques, pas conceptuelles.** Mutation de table Oracle, désynchronisation des mocks, équation de Kepler qui diverge, accents qui cassent les enums Kotlin, Oracle pas prêt avant Flask. Chaque problème a une solution documentée dans le code et dans les rapports.
