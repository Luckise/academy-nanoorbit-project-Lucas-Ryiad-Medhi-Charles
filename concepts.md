# Tous les concepts du projet — expliqués simplement

> Glossaire pour comprendre **chaque mot technique** utilisé dans le projet NanoOrbit.
> Explications courtes, en français de tous les jours, avec analogies quand c'est utile.

---

## 1. Concepts métier (ce que représente le projet)

### CubeSat
Petit satellite cubique standardisé. L'unité de base s'appelle **1U** (10×10×10 cm, ~1 kg). On peut empiler les unités → **3U** (3 cubes), **6U**, **12U**. C'est moins cher qu'un gros satellite et ça suffit pour de la surveillance climatique.

### Orbite (SSO, LEO, MEO, GEO)
Trajectoire que suit le satellite autour de la Terre.
- **LEO** (Low Earth Orbit) — orbite basse, 200-2000 km. Vitesse rapide (~90 min par tour).
- **SSO** (Sun-Synchronous Orbit) — cas particulier de LEO, le satellite passe au-dessus de chaque point à la même heure solaire locale. Idéal pour la surveillance environnementale (mêmes conditions de lumière).
- **MEO / GEO** — orbites moyennes / géostationnaires, plus hautes.

### Fenêtre de communication
Créneau pendant lequel un satellite peut échanger des données avec une station au sol — typiquement 5 à 10 minutes pendant qu'il survole la station. Hors de ce créneau, il est trop loin ou caché par l'horizon.

### Station au sol
Antenne sur Terre qui parle aux satellites quand ils passent au-dessus. Dans le projet : Toulouse, Kiruna (Arctique), Singapour.

### Centre de contrôle
Bâtiment où des opérateurs supervisent la constellation. Dans le projet : Paris, Houston, Singapour.

### NORAD ID
Numéro d'identification unique attribué par l'armée américaine à chaque objet en orbite. L'ISS = NORAD 25544.

---

## 2. Bases de données (ALTN83)

### SGBD / SGBDR
**Système de Gestion de Base de Données** (Relationnel). Logiciel qui stocke des tables et garantit que les données restent cohérentes. Oracle, MySQL, PostgreSQL en sont. "Relationnel" = données organisées en tables liées entre elles.

### Base de données distribuée (ou répartie)
Une seule base "logique" mais dont les données sont **physiquement éparpillées sur plusieurs serveurs**. Dans NanoOrbit : un serveur à Paris, un à Houston, un à Singapour. L'utilisateur final ne voit qu'une seule base.

### Fragmentation
Découper une table sur plusieurs serveurs.
- **Horizontale** = on découpe en lignes. Paris garde ses propres fenêtres, Houston les siennes. Comme une copie de classeur où chaque centre a ses propres factures.
- **Verticale** = on découpe en colonnes. Un serveur a les noms, un autre a les données techniques. (Pas utilisé dans NanoOrbit.)

### Réplication
Copier les mêmes données sur plusieurs serveurs. **Réplique en lecture seule** = un serveur a une copie qu'il peut lire mais pas modifier (la copie de référence est ailleurs). Sert à survivre aux coupures réseau.

### 2PC (Two-Phase Commit)
Protocole pour faire une mise à jour cohérente sur plusieurs serveurs en même temps. Étape 1 : on demande à tous les serveurs "tu peux le faire ?". Étape 2 : si tous disent oui, on valide partout ; sinon on annule partout. Évite qu'un serveur valide pendant qu'un autre échoue.

### Théorème CAP
Dans un système distribué, on ne peut avoir que **2 sur 3** parmi : **C**onsistency (toutes les copies sont identiques), **A**vailability (le système répond toujours), **P**artition tolerance (le système survit aux coupures réseau). NanoOrbit choisit AP : on accepte une légère incohérence temporaire pour rester disponible.

### MCD / MLD
- **MCD** (Modèle Conceptuel des Données) = schéma sur papier qui montre les **entités** et les **associations** entre elles. Indépendant de la techno. C'est le plan d'architecte avant de couler le béton.
- **MLD** (Modèle Logique des Données) = traduction du MCD en tables relationnelles avec PK/FK. Plus proche du code.

### Forme normale (3NF)
Règles de bonne hygiène pour éviter les redondances dans les tables. La **3NF** (3ème forme normale) dit : pas de doublons, chaque colonne dépend uniquement de la clé primaire, pas de dépendance entre colonnes non-clés. En pratique : si on doit changer une info, on ne la change qu'à un seul endroit.

### Clé primaire (PK)
Identifiant unique d'une ligne dans une table. Comme un numéro de Sécurité sociale : un seul par personne.

### Clé étrangère (FK)
Colonne qui pointe vers la clé primaire d'une autre table. Crée le lien entre tables. `SATELLITE.id_orbite` est une FK qui pointe vers `ORBITE.id_orbite`.

### Clé composite
Une PK ou une FK faite de plusieurs colonnes. Dans `EMBARQUEMENT`, la PK c'est `(id_satellite, ref_instrument)` ensemble : un même instrument peut être embarqué sur plusieurs satellites, ce qui fait l'unicité c'est le couple.

### Contraintes
Règles automatiquement appliquées par Oracle.
- **NOT NULL** = la colonne ne peut pas être vide.
- **UNIQUE** = pas deux lignes avec la même valeur.
- **CHECK** = condition à respecter (ex: `duree BETWEEN 1 AND 900`).
- **PK** = clé primaire (= NOT NULL + UNIQUE).
- **FK** = clé étrangère (= la valeur doit exister dans l'autre table).

### DDL / DML
Langages Oracle :
- **DDL** (Data Definition Language) = créer/modifier des tables. Mots-clés : `CREATE TABLE`, `ALTER`, `DROP`.
- **DML** (Data Manipulation Language) = manipuler les données. Mots-clés : `INSERT`, `UPDATE`, `DELETE`, `SELECT`.

### Trigger
Bout de code qui s'exécute automatiquement quand un événement se produit sur une table. Comme un capteur de mouvement : "quelqu'un essaie d'insérer une fenêtre → je vérifie d'abord que le satellite n'est pas désorbité".

- **BEFORE** = avant l'événement (peut bloquer)
- **AFTER** = après (pour logger, par exemple)
- **ROW** = ligne par ligne
- **STATEMENT** = une seule fois par requête

### `:OLD` / `:NEW`
Dans un trigger : `:OLD` = la ligne avant modification, `:NEW` = la ligne après. Utile pour comparer (ex: "si le statut a changé, logger l'ancien et le nouveau").

### Mutation de table (ORA-04091)
Erreur Oracle classique : un trigger row-level sur une table essaie de lire la même table → interdit parce qu'Oracle ne sait pas dans quel état elle est pendant la modification. Solution : restructurer le trigger pour ne pas relire la table en cours, ou passer en statement-level.

### `RAISE_APPLICATION_ERROR`
Procédure PL/SQL pour lever une erreur métier personnalisée. `RAISE_APPLICATION_ERROR(-20101, 'Satellite désorbité')` = renvoie l'erreur ORA-20101 avec le message. Les codes -20000 à -20999 sont réservés aux erreurs custom.

### PL/SQL
Langage de programmation procédural d'Oracle = **SQL + structures de programme** (variables, if, boucles, fonctions). On l'utilise pour les triggers, procédures, fonctions, packages.

### Bloc anonyme
Programme PL/SQL qui n'est pas sauvegardé dans la base, juste exécuté une fois. Structure : `DECLARE ... BEGIN ... END;`.

### `%ROWTYPE`
Raccourci PL/SQL pour déclarer une variable qui a la même structure qu'une ligne de table. `v_sat SATELLITE%ROWTYPE` = variable avec un champ pour chaque colonne de SATELLITE. Si la table change, le code suit automatiquement.

### Curseur
Pointeur qui parcourt les résultats d'un `SELECT` ligne par ligne.
- **Implicite** = créé automatiquement par Oracle pour un `SELECT INTO`.
- **Explicite** = on le déclare, on l'ouvre, on fait des FETCH, on le ferme.
- **Paramétré** = curseur explicite qui prend des arguments comme une fonction.
- **Cursor FOR Loop** = forme compacte qui ouvre/parcourt/ferme automatiquement.

### Procédure / Fonction
- **Procédure** = bout de code réutilisable qui fait une action (ne renvoie rien, ou des paramètres OUT).
- **Fonction** = procédure qui renvoie une valeur. S'utilise comme dans une expression : `SELECT calculer_volume(...) FROM ...`.

### Package
Regroupement nommé de procédures, fonctions, types, constantes. Composé de **SPEC** (la déclaration publique = le sommaire visible) et **BODY** (l'implémentation cachée). Comme un module Python.

### `NVL` / `COALESCE`
Gérer les valeurs `NULL`. `NVL(resolution, 'N/A')` = renvoie `resolution` ou `'N/A'` si NULL. `COALESCE` = comme NVL mais avec plus d'arguments.

### Vue
Requête SELECT sauvegardée et nommée, qu'on utilise comme une table en lecture. Permet de cacher la complexité (jointures, calculs) derrière un nom simple.

### Vue matérialisée
Vue dont le résultat est **stocké physiquement** sur disque. Plus rapide à lire mais à rafraîchir manuellement (`REFRESH ON DEMAND`) ou automatiquement. Utile pour les rapports lents qu'on consulte souvent.

### CTE (Common Table Expression)
Sous-requête nommée qu'on définit avec `WITH ... AS (...)` au début d'une requête, et qu'on réutilise plusieurs fois. Plus lisible qu'une grosse sous-requête imbriquée.

### CTE récursive
CTE qui s'appelle elle-même pour parcourir une hiérarchie. Permet d'afficher un arbre (centre → station → fenêtre) en une seule requête.

### Sous-requête scalaire / corrélée
- **Scalaire** = sous-requête qui renvoie une seule valeur, utilisable dans un `SELECT` ou `WHERE`. Ex : `WHERE prix > (SELECT AVG(prix) FROM produits)`.
- **Corrélée** = sous-requête qui dépend de la requête extérieure (référence une colonne de l'extérieur). Recalculée pour chaque ligne. Plus lente mais plus expressive.

### Fonctions analytiques (window functions)
Fonctions qui calculent sur **un sous-ensemble glissant** de lignes au lieu de regrouper. Diffèrent de `GROUP BY` parce qu'elles **gardent toutes les lignes**.
- **`ROW_NUMBER()`** = numérote les lignes 1, 2, 3...
- **`RANK()`** = classement avec ex aequo (1, 2, 2, 4)
- **`DENSE_RANK()`** = classement sans trou (1, 2, 2, 3)
- **`LAG()` / `LEAD()`** = valeur de la ligne précédente / suivante
- **`SUM() OVER (...)`** = somme cumulée sur une fenêtre

### `PARTITION BY`
Clause des fonctions analytiques qui dit "calcule séparément pour chaque groupe". `RANK() OVER (PARTITION BY type_orbite ORDER BY altitude)` = classement de l'altitude **dans chaque type d'orbite**.

### `MERGE INTO` (UPSERT)
Faire `INSERT` **ou** `UPDATE` selon que la ligne existe déjà. Utile pour synchroniser deux sources : "si le satellite existe, mets à jour son statut, sinon crée-le".

### Index
Structure auxiliaire (souvent un **B-tree**) qui accélère la recherche. Comme l'index alphabétique d'un livre : au lieu de tourner toutes les pages, on va directement à la bonne. **Coût** : plus lent à écrire, plus de stockage.

- **Index B-tree** = arbre équilibré, le standard.
- **Index composite** = sur plusieurs colonnes (`statut, id_orbite`).
- **Index fonctionnel** = sur le résultat d'une fonction (`TRUNC(date, 'MM')`).

### `EXPLAIN PLAN`
Demande à Oracle "comment vas-tu exécuter cette requête ?". Affiche le **plan d'exécution** : ordre des jointures, accès aux tables, utilisation des index.

### TABLE ACCESS FULL vs INDEX RANGE SCAN
Deux façons de lire une table.
- **TABLE ACCESS FULL** = scanner toute la table de la première à la dernière ligne. Lent sur grosse table.
- **INDEX RANGE SCAN** = consulter un index pour ne lire que les lignes pertinentes. Beaucoup plus rapide si l'index est bien choisi.

### `GENERATED ALWAYS AS IDENTITY`
Colonne qui s'auto-incrémente à chaque INSERT. Remplace l'ancien combo "séquence + trigger BEFORE INSERT".

### `LISTAGG`
Fonction d'agrégation qui concatène des chaînes en une seule. `LISTAGG(type, ', ')` sur 3 lignes = `'SSO, SSO, LEO'`. Utile pour le reporting.

---

## 3. Android / Kotlin / Compose (ALTN82)

### Kotlin
Langage moderne créé par JetBrains, devenu **le langage officiel d'Android**. Plus concis et plus sûr que Java (pas de NullPointerException si on respecte la syntaxe). Compile vers la JVM.

### `data class`
Classe Kotlin spéciale pour stocker des données. Génère automatiquement `equals`, `hashCode`, `toString`, `copy`. Une ligne suffit : `data class Satellite(val id: String, val nom: String)`.

### `enum class`
Type énuméré : un ensemble fini de valeurs nommées. `enum class StatutSatellite { OPERATIONNEL, EN_VEILLE, ... }`. Plus sûr qu'une `String` parce que le compilateur vérifie qu'on n'utilise que des valeurs valides.

### `sealed class`
Classe dont **toutes les sous-classes sont connues à la compilation**. Permet au compilateur de vérifier qu'on traite tous les cas dans un `when`. Utilisée pour `Screen` dans la navigation.

### Jetpack Compose
Framework UI déclaratif d'Android (sortie 2021), remplace l'ancien système XML. On écrit `@Composable fun Card() { Text("Hello") }` et l'UI se construit automatiquement. Inspiré de React/SwiftUI.

### `@Composable`
Annotation qui marque une fonction comme un "morceau d'UI". Une fonction `@Composable` peut appeler d'autres `@Composable`. C'est l'unité de base de Compose.

### Recomposition
Quand un état change, Compose **réexécute** automatiquement les fonctions `@Composable` qui dépendent de cet état. C'est le mécanisme magique qui synchronise UI et données.

### `@Preview`
Annotation qui permet de voir le rendu d'un Composable directement dans Android Studio sans lancer l'app. Hyper utile pour itérer sur le design.

### `State` / `StateFlow` / `Flow`
- **`Flow`** = flux asynchrone de valeurs (zéro, une ou plusieurs au cours du temps). Comme un tuyau d'eau.
- **`StateFlow`** = `Flow` qui a toujours une valeur courante (un état). On peut le lire avec `.value` et le collecter.
- **`State<T>`** dans Compose = valeur observable qui déclenche la recomposition quand elle change.

### `collectAsStateWithLifecycle()`
Méthode qui transforme un `StateFlow` en `State<T>` Compose, **en respectant le cycle de vie** : si l'écran n'est plus visible, la collecte s'arrête (économie de batterie).

### `LazyColumn`
Liste verticale qui **n'affiche que les éléments visibles** à l'écran (et un peu de marge). Équivalent du `RecyclerView` de l'ancien monde. Sans ça, afficher 1000 satellites planterait l'app.

### Material3
Système de design de Google (boutons, cards, chips, couleurs, typo). Compose fournit des composants `Material3` prêts à l'emploi.

### MVVM (Model-View-ViewModel)
Architecture en 3 couches.
- **Model** = données pures (`data class Satellite`).
- **ViewModel** = logique de présentation, expose des états observables (`StateFlow`).
- **View** = l'UI Compose qui observe le ViewModel et affiche.

Avantage : on peut tester le ViewModel sans toucher à l'UI.

### `ViewModel`
Classe Android qui **survit aux rotations d'écran** et qui héberge la logique de présentation. Se construit avec une `Factory` quand il a besoin de dépendances.

### Repository pattern
Couche entre le ViewModel et les sources de données (API, base locale). Le ViewModel ne sait pas si les données viennent d'Internet ou du cache. Permet de changer la source sans toucher au ViewModel.

### DTO (Data Transfer Object)
Objet utilisé pour le **transport** des données (réseau, JSON) — séparé du modèle métier. `SatelliteApiResponse` est un DTO ; on le convertit en `Satellite` (modèle) à l'arrivée. Évite que les changements d'API contaminent le code métier.

### Retrofit
Librairie qui transforme une **interface Kotlin** en appels HTTP. On déclare `@GET("/satellites") suspend fun getSatellites(): List<SatelliteDto>` et Retrofit génère le code réseau.

### Room (DAO, Entity, Database)
ORM (Object-Relational Mapping) officiel d'Android pour SQLite.
- **Entity** = classe annotée `@Entity` qui devient une table.
- **DAO** (Data Access Object) = interface annotée `@Dao` avec les requêtes (`@Query`, `@Insert`).
- **Database** = classe annotée `@Database` qui rassemble entités et DAOs.

Room expose des `Flow<List<X>>` qui réagissent automatiquement aux changements de la base.

### Cache-First (stratégie)
On lit **toujours d'abord le cache local**, puis on rafraîchit en arrière-plan. Avantage : l'UI s'affiche instantanément (pas d'écran de chargement) et l'app marche même hors-ligne. Si le rafraîchissement échoue, on reste sur les données cachées.

### WorkManager / `CoroutineWorker`
Système Android pour lancer des **tâches en arrière-plan** garanties (s'exécutent même si l'app est fermée). Idéal pour : envoi de données, vérifications périodiques.
- **`CoroutineWorker`** = `Worker` qui utilise des coroutines (code asynchrone moderne).
- **`PeriodicWorkRequest`** = "exécute toutes les 15 minutes minimum".

### Coroutines / `suspend` / `viewModelScope`
- **Coroutine** = fil d'exécution léger pour code asynchrone. Une app peut en avoir des milliers, vs ~1 par thread classique.
- **`suspend`** = mot-clé qui marque une fonction comme "peut prendre du temps, peut s'interrompre". Ne peut être appelée que d'une autre fonction `suspend` ou d'une coroutine.
- **`viewModelScope`** = scope de coroutine fourni par AndroidX, automatiquement annulé quand le ViewModel est détruit.

### `combine()` / `stateIn` / `derivedStateOf`
- **`combine(flow1, flow2) { ... }`** = combine plusieurs `Flow` en un seul, recalcule à chaque émission. Utilisé pour le filtrage : "filtre = combinaison de la query texte ET du statut sélectionné".
- **`stateIn(scope, started, initial)`** = transforme un `Flow` cold en `StateFlow` partageable. Évite de recalculer à chaque collecte.
- **`derivedStateOf { ... }`** = état Compose dérivé d'autres états, recalculé seulement quand la dépendance change vraiment.

### Navigation Compose / `NavHost` / `Screen`
Système de navigation entre écrans en Compose.
- **`NavHost`** = conteneur central qui décide quel écran afficher selon la route active.
- **`sealed class Screen`** = définit toutes les routes possibles de manière typée (`Screen.Dashboard`, `Screen.Detail("SAT-001")`).

### osmdroid
Librairie Android qui affiche des cartes **OpenStreetMap** (alternative gratuite à Google Maps). Pas besoin de clé API, pas de quotas.

### CameraX / `PreviewView`
API officielle Android pour la caméra, plus simple que l'ancienne Camera2.
- **`PreviewView`** = vue qui affiche le flux caméra en direct.
- Architecture en use-cases : Preview, ImageAnalysis, ImageCapture.

### `SensorManager` / `TYPE_ROTATION_VECTOR`
- **`SensorManager`** = service Android qui donne accès aux capteurs (accéléromètre, gyroscope, magnétomètre…).
- **`TYPE_ROTATION_VECTOR`** = capteur **virtuel** qui combine accéléromètre + gyroscope + magnétomètre pour donner directement l'orientation 3D du téléphone (matrice de rotation). Plus précis que les capteurs bruts.

### `LocationManager`
Service Android pour la géolocalisation. Donne accès à GPS, Network (Wi-Fi/cellulaire), Passive. Demande des permissions runtime.

### `BuildConfig`
Classe générée par Gradle à la compilation, contenant les constantes de build (version, URL d'API, etc.). On y met `API_BASE_URL` au lieu de hardcoder dans le code.

### `AndroidManifest.xml`
Fichier descriptif de l'app : permissions demandées (`CAMERA`, `INTERNET`), composants (Activities), features matérielles requises. C'est la "carte d'identité" de l'app.

### Permission runtime
Depuis Android 6, certaines permissions (caméra, GPS, micro) doivent être demandées **à l'utilisateur au moment de l'usage**, pas seulement à l'install. Si l'utilisateur refuse, l'app doit dégrader gracieusement.

---

## 4. Backend / Web / DevOps

### API REST
**A**pplication **P**rogramming **I**nterface **RE**presentational **S**tate **T**ransfer. Façon standardisée pour qu'une app cliente parle à un serveur via HTTP.

### Endpoint / Route
URL exposée par l'API qui fait une chose précise. `GET /satellites` = "donne-moi tous les satellites". Une API a typiquement une dizaine d'endpoints.

### Verbes HTTP
- **GET** = lire (idempotent, sans effet de bord).
- **POST** = créer.
- **PUT** = mettre à jour entièrement.
- **PATCH** = mettre à jour partiellement.
- **DELETE** = supprimer.

### JSON
**JavaScript Object Notation**. Format texte pour échanger des données structurées. `{"id": "SAT-001", "nom": "Alpha"}`. Lu nativement par JavaScript, toutes les autres langues ont des parsers.

### CORS (Cross-Origin Resource Sharing)
Sécurité navigateur : par défaut, une page web sur `domaineA.com` ne peut pas appeler une API sur `domaineB.com`. Le serveur doit explicitement l'autoriser via des headers HTTP. Géré par `flask-cors` côté serveur.

### Flask
Micro-framework web Python. Écrire une API en Flask = quelques lignes : on importe, on crée une `Flask(__name__)`, on décore une fonction avec `@app.route("/satellites")`, et c'est servi.

### Docker
Outil qui empaquette une application + ses dépendances dans un **conteneur**, isolé du système hôte. "Ça marche chez moi" → "ça marche partout". Image = recette ; conteneur = instance qui tourne.

### Image / Conteneur
- **Image** = fichier figé qui contient l'app + ses dépendances (Linux + Python + Flask + code). Comme une ISO de Live CD.
- **Conteneur** = exécution d'une image. On peut en lancer 10 du même image en parallèle.

### `docker-compose`
Outil qui orchestre plusieurs conteneurs ensemble. On décrit dans `docker-compose.yml` : "je veux un conteneur Oracle, un conteneur Flask, ils doivent se voir". Un seul `docker compose up` lance tout.

### Volume Docker
Répertoire persistant attaché à un conteneur. Quand le conteneur est supprimé, les données du volume restent. Indispensable pour la base de données — sinon on perd tout à chaque redémarrage.

### `depends_on`
Directive `docker-compose` qui dit "ne lance pas Flask avant qu'Oracle soit démarré". **Attention** : ça attend juste le démarrage du process, pas la disponibilité réelle. Pour ça il faut un healthcheck ou une boucle de retry côté code.

### Healthcheck
Vérification périodique que le service répond bien. Côté Docker = commande qui doit renvoyer 0. Côté HTTP = endpoint `/health` qui répond 200 OK si tout va bien.

---

## 5. Mécanique orbitale (bonus AR)

### TLE / OMM
Formats publics qui décrivent l'orbite d'un satellite à un instant donné.
- **TLE** (Two-Line Element) = format historique, deux lignes de chiffres ASCII.
- **OMM** (Orbit Mean-elements Message) = format moderne, JSON ou XML, plus lisible.

CelesTrak publie ces deux formats pour ~25 000 satellites, mis à jour toutes les 8 h.

### Élément orbital
Paramètre qui décrit l'orbite. Les **6 éléments képlériens** classiques :
1. **Demi-grand axe (a)** = "taille" de l'orbite.
2. **Excentricité (e)** = à quel point l'orbite est ovale (0 = cercle, proche de 1 = très allongée).
3. **Inclinaison (i)** = angle entre l'orbite et l'équateur.
4. **RAAN (Ω)** = orientation du plan orbital (longitude du nœud ascendant).
5. **Argument du périgée (ω)** = orientation de l'ellipse dans son plan.
6. **Anomalie moyenne (M)** = position du satellite sur l'orbite à un instant donné.

### Anomalie (moyenne / vraie / excentrique)
Trois façons de mesurer la position du satellite sur son orbite à un instant `t`.
- **Moyenne (M)** = "comme s'il allait à vitesse constante" (fictive, mais facile à calculer : `M = M₀ + n·t`).
- **Excentrique (E)** = position projetée sur le cercle circonscrit à l'ellipse.
- **Vraie (ν)** = vraie position angulaire vue depuis le foyer (la Terre).

Conversion : `M → E` via Kepler, puis `E → ν` par formule trigonométrique.

### Équation de Kepler
`M = E − e · sin(E)`. Donne `E` si on connaît `M` et `e`. **Pas de solution analytique** → résolution numérique (Newton-Raphson).

### Newton-Raphson
Méthode numérique pour trouver les zéros d'une fonction. On part d'une estimation et on itère : `x_{n+1} = x_n - f(x_n)/f'(x_n)`. Converge rapidement si on part près de la solution.

### Repères / référentiels
- **ECI** (Earth-Centered Inertial) = origine au centre de la Terre, axes **fixes par rapport aux étoiles**. Idéal pour les calculs orbitaux.
- **ECEF** (Earth-Centered Earth-Fixed) = origine au centre de la Terre, axes **qui tournent avec la Terre**. La latitude/longitude est dans ECEF.
- **ENU** (East-North-Up) = repère **local à l'observateur**. East = vers l'est, North = vers le nord, Up = vers le ciel. C'est dans ce repère qu'on calcule azimut et élévation.

### Azimut / Élévation
- **Azimut** = angle horizontal mesuré depuis le nord, dans le sens horaire (0° = nord, 90° = est, 180° = sud, 270° = ouest).
- **Élévation** = angle au-dessus de l'horizon (0° = horizon, 90° = zénith, négatif = sous l'horizon).

Tout objet visible dans le ciel a un couple (azimut, élévation) à un instant donné.

### Champ de vision (FOV)
Ouverture angulaire de la caméra.
- **HFOV** = horizontal field of view (~65° pour un smartphone)
- **VFOV** = vertical field of view (~50°)

Si l'écart entre la direction de pointage et la direction d'un satellite tient dans le FOV, on peut le dessiner sur l'écran.

### Propagateur (képlérien vs SGP4)
Algorithme qui calcule où sera un satellite à un instant futur, en partant des éléments orbitaux.
- **Képlérien** = orbite à 2 corps, parfait pour expliquer mais ignore les perturbations (aplatissement Terre, traînée atmosphérique, Lune, Soleil). Précis sur quelques heures.
- **SGP4** (Simplified General Perturbations 4) = standard NORAD, prend en compte les principales perturbations. Précis sur plusieurs jours. Beaucoup plus complexe à implémenter.

### Constantes physiques
- **R_terre = 6371 km** — rayon moyen de la Terre.
- **GM = 398 600,4418 km³/s²** — constante gravitationnelle × masse de la Terre.
- **ω_terre = 7,2921 × 10⁻⁵ rad/s** — vitesse de rotation de la Terre (1 tour par 23h56m).

---

## 6. Quelques abréviations qui reviennent

| Sigle | Signification |
|---|---|
| **DDL** | Data Definition Language (CREATE, ALTER) |
| **DML** | Data Manipulation Language (INSERT, UPDATE, DELETE, SELECT) |
| **PK / FK** | Primary Key / Foreign Key |
| **3NF** | 3ème Forme Normale |
| **CTE** | Common Table Expression |
| **MVVM** | Model-View-ViewModel |
| **DTO** | Data Transfer Object |
| **API** | Application Programming Interface |
| **REST** | Representational State Transfer |
| **JSON** | JavaScript Object Notation |
| **CORS** | Cross-Origin Resource Sharing |
| **DAO** | Data Access Object |
| **ORM** | Object-Relational Mapping |
| **SDK** | Software Development Kit |
| **FOV** | Field Of View |
| **TLE** | Two-Line Element |
| **OMM** | Orbit Mean-elements Message |
| **ECI / ECEF / ENU** | Earth-Centered Inertial / Earth-Fixed / East-North-Up |
| **LEO / SSO / GEO** | Low Earth / Sun-Synchronous / Geostationary Orbit |
| **CubeSat** | Satellite cubique standardisé (1U = 10×10×10 cm) |
| **SGBD(R)** | Système de Gestion de Base de Données (Relationnelle) |
| **2PC** | Two-Phase Commit |
| **CAP** | Consistency / Availability / Partition tolerance |
| **RG** | Règle de Gestion |

---

## 7. Si tu n'as pas le temps de tout lire

**Bases de données — les 5 concepts à connaître absolument** :
1. **Trigger** = code qui s'exécute automatiquement sur INSERT/UPDATE/DELETE.
2. **PL/SQL** = langage Oracle pour les procédures, triggers, fonctions.
3. **Vue / vue matérialisée** = SELECT sauvegardé sous un nom (matérialisée = stockée).
4. **Index** = structure qui accélère les recherches au prix de l'écriture.
5. **Fragmentation horizontale** = découper une table en lignes sur plusieurs serveurs.

**Android — les 5 concepts à connaître absolument** :
1. **Compose `@Composable`** = fonction qui définit un morceau d'UI déclaratif.
2. **`StateFlow`** = état observable qui déclenche la recomposition.
3. **MVVM** = séparer les données (Model), la logique (ViewModel) et l'UI (View).
4. **Cache-First avec Room** = lire local d'abord, rafraîchir en arrière-plan.
5. **Retrofit** = transformer une interface Kotlin en appels HTTP.

**Backend / DevOps — les 3 concepts à connaître** :
1. **API REST** = serveur qui répond à des URLs en JSON.
2. **Docker / docker-compose** = empaqueter et orchestrer plusieurs services.
3. **CORS** = sécurité navigateur qui filtre les requêtes cross-domain.

**AR / orbite — les 3 concepts à connaître** :
1. **Éléments orbitaux + équation de Kepler** = on sait calculer où est un satellite à un instant `t`.
2. **ECI → ECEF → ENU** = transformations pour passer du repère "étoiles" au repère "horizon de l'observateur".
3. **`TYPE_ROTATION_VECTOR`** = capteur virtuel Android qui donne directement l'orientation 3D du téléphone.
