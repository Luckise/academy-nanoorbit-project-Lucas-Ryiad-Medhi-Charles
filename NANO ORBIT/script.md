# Script de présentation NanoOrbit

> Notes pour chaque slide + questions techniques à anticiper.
> Format : ce qu'on dit · ce qu'on montre · transition vers la slide suivante.
> 16 slides au total · ~18 min de présentation + 7 min de Q&A.

---

## Slide 1 — Cover (`SceneCover`) · ~30 s

**À l'écran :** "NANO·ORBIT" qui s'écrit lettre par lettre, sous-titre Ground Control System, mention équipe.

**Ce qu'on dit (accroche) :**
> "Bonjour. Pendant 18 minutes on va vous présenter NanoOrbit — une startup fictive qui exploite une constellation de CubeSats pour la surveillance climatique. Mais ce n'est pas un projet de fiction : c'est l'architecture complète, base de données distribuée et application mobile, livrée en cohérence stricte sur les modules ALTN82 et ALTN83."

**Transition :**
> "Posons d'abord le contexte."

---

## Slide 2 — Contexte (`SceneContext`) · ~45 s

**À l'écran :** "Surveiller / le climat / depuis l'orbite." + Terre stylisée + 3 satellites en orbite + chiffres clés (03 centres, 11 tables, 05 triggers, 05 écrans).

**Ce qu'on dit :**
> "NanoOrbit, c'est une constellation de CubeSats — petits satellites cubiques de 10 cm — qui passent au-dessus de zones sensibles : déforestation, fonte des glaces, qualité de l'air, érosion côtière. Pour piloter cette flotte, deux livrables : une **base Oracle distribuée sur 3 centres** (Paris, Houston, Singapour) et une **application Android** que les opérateurs Ground Control utilisent au quotidien. La consigne pédagogique fondatrice — celle qu'on ne perd jamais de vue : ces deux livrables doivent être **strictement cohérents**. Mêmes entités, mêmes identifiants, mêmes règles métier. C'est ce qu'on appelle la synergie."

**Transition :**
> "Voici comment on a structuré ça."

---

## Slide 3 — Architecture (`SceneArchitecture`) · ~60 s

**À l'écran :** trois blocs reliés — Oracle 23ai → API REST Flask → Android.

**Ce qu'on dit :**
> "Trois briques techniques. À gauche **Oracle 23ai** : 11 tables, 5 triggers, fragmentation horizontale par centre. Au milieu, **une API REST Flask en Python** qu'on a ajoutée nous-mêmes — elle n'était pas demandée mais elle est indispensable : elle ponte le SQL Oracle et le JSON consommé par l'app. À droite, **l'application Android** en Kotlin/Compose avec architecture MVVM et cache Room. Le tout orchestré par **Docker Compose** : un seul `docker compose up` lance Oracle et l'API ensemble, avec auto-init du schéma au premier démarrage."

**Point fort à appuyer :**
> "L'API mappe explicitement les valeurs Oracle accentuées vers les enums Kotlin sans accent — `Opérationnel` devient `OPERATIONNEL`. C'est petit mais sans ça, `enumValueOf` crashe."

**Transition :**
> "Côté outillage, voici les deux stacks."

---

## Slide 4 — Stack (`SceneStack`) · ~45 s

**À l'écran :** deux colonnes — ALTN83 Backend (Oracle, PL/SQL, Docker, Flask, oracledb, SQL avancé) et ALTN82 Mobile (Kotlin, Compose, MVVM/StateFlow, Retrofit/Room, osmdroid/CameraX, WorkManager).

**Ce qu'on dit :**
> "À gauche, le monde Oracle : du SQL avancé, du PL/SQL procédural, et tout l'outillage DevOps autour. À droite, la stack Android moderne : Kotlin, Jetpack Compose pour l'UI déclarative, MVVM avec StateFlow, Retrofit pour le réseau, Room pour la persistance locale, osmdroid pour la cartographie open-source, CameraX pour le bonus AR, et WorkManager pour les notifications en arrière-plan."

**Transition :**
> "On rentre dans le concret. ALTN83, Phase 1."

---

## Slide 5 — Phase 1 — Conception distribuée (`ScenePhase1`) · ~75 s

**À l'écran :** "Distribuer la constellation." · Paris (Maître) / Houston (Réplique) / Singapour (Lecture seule) · 11 tables 3NF · Fragmentation horizontale · Réplique R/O à Singapour.

**Ce qu'on dit :**
> "Phase 1, c'est la conception sur papier. On a fait un dictionnaire des données complet, un MCD avec deux associations porteuses — `EMBARQUEMENT` qui porte la date d'intégration et l'état de fonctionnement, et `PARTICIPATION` qui porte le rôle du satellite — et un MLD en 3NF.
>
> **La grande question de cette phase, c'est la fragmentation.** Le sujet demande comment Singapour peut continuer à planifier des fenêtres de communication si le serveur central tombe. On a tranché pour une **fragmentation horizontale** : chaque centre a sa propre table `FENETRE_COM` locale plus une réplique en lecture seule des tables globales. Ça garantit la **continuité de service** en cas de coupure réseau — c'est un choix de disponibilité plutôt que de cohérence, un classique du théorème CAP."

**Transition :**
> "Voici concrètement le MCD."

---

## Slide 6 — MCD (`SceneMCD`) · ~30 s

**À l'écran :** image du MCD complet.

**Ce qu'on dit :**
> "Voici le MCD. 11 entités, deux associations porteuses visibles ici (EMBARQUEMENT et PARTICIPATION), et la triangulation `FENETRE_COM ↔ SATELLITE ↔ STATION_SOL` qui est le cœur métier. On ne va pas le détailler entité par entité, on s'arrête sur la structure générale."

**Transition :**
> "Phase 2 : on coule le béton."

---

## Slide 7 — Phase 2 Triggers (`ScenePhase2`) · ~75 s

**À l'écran :** liste des 5 triggers T1 à T5 avec règle de gestion associée.

**Ce qu'on dit :**
> "Phase 2, c'est le DDL et la programmation des règles métier. Le DDL est straight-forward : 11 `CREATE TABLE` avec les contraintes. Le travail intéressant, ce sont les **5 triggers**.
>
> - **T1** bloque toute fenêtre de communication sur un satellite désorbité ou une station en maintenance — RG-S06 et RG-G03 — avec une `ORA-20101`.
> - **T2** détecte les chevauchements : un satellite ne peut pas parler à deux stations en même temps, et inversement — RG-F02 et RG-F03.
> - **T3** force le `volume_donnees` à NULL si la fenêtre n'est pas encore réalisée — c'est un correctif silencieux, pas une erreur.
> - **T4**, en bonus, bloque l'ajout d'un satellite à une mission terminée.
> - **T5**, aussi en bonus, journalise toutes les modifications de statut dans une table `HISTORIQUE_STATUT`.
>
> Chaque trigger a été testé avec un cas qui passe et un cas qui échoue. Les captures sont dans le dossier `screens/`."

**Transition :**
> "Sur T2, on a buté sur un classique d'Oracle."

---

## Slide 8 — ORA-04091 (`SceneORA`) · ~75 s

**À l'écran :** bloc "Avant" avec le SELECT qui plante en ORA-04091, bloc "Après" avec la solution.

**Ce qu'on dit :**
> "Le trigger T2 anti-chevauchement, première version, faisait un `SELECT * FROM FENETRE_COM` directement à l'intérieur d'un trigger row-level sur cette même table. Oracle interdit ça : c'est l'erreur **ORA-04091, table is mutating, trigger may not see it**. Logique : Oracle ne sait pas dans quel état est la table pendant qu'on est en train de la modifier.
>
> La solution, à droite : on compare `:NEW` aux **lignes déjà engagées** en excluant explicitement la ligne en cours d'insertion via `NVL(id_fenetre, -1) <> :NEW.id_fenetre`. Plus de mutation, et on détecte bien les chevauchements."

**Transition :**
> "Phase 3 : on programme dans la base."

---

## Slide 9 — Phase 3 PL/SQL (`ScenePhase3`) · ~75 s

**À l'écran :** 6 paliers à gauche (bloc anonyme, %ROWTYPE, structures de contrôle, curseurs, procédures/fonctions, package) + détail du package à droite (4 procédures, 3 fonctions, 1 type composite).

**Ce qu'on dit :**
> "Phase 3, c'est du **PL/SQL** : 16 exercices répartis sur 5 paliers progressifs — bloc anonyme, `%ROWTYPE`, if/case/for, curseurs, procédures et fonctions.
>
> Le **palier 6 en bonus**, c'est le package `pkg_nanoOrbit` : 4 procédures (planifier, clôturer, affecter, mettre en révision), 3 fonctions (volume théorique, statut constellation, stats par satellite), un type composite et des constantes métier centralisées.
>
> L'élégance du package, c'est qu'il **orchestre les triggers de la Phase 2**. Quand on appelle `pkg_nanoOrbit.planifier_fenetre`, l'INSERT déclenche T1, qui peut lever ORA-20101 si le satellite est désorbité. Donc la même règle métier est garantie quel que soit le chemin d'écriture."

**Transition :**
> "Phase 4 : on optimise pour la production."

---

## Slide 10 — Phase 4 Optimisation (`ScenePhase4`) · ~90 s

**À l'écran :** grille 6 cases — VUES, CTE, ANALYTIQUES, MERGE, INDEX, PLAN.

**Ce qu'on dit :**
> "Phase 4, c'est de l'Oracle avancé. On a livré :
>
> - **3 vues plus une vue matérialisée** `mv_volumes_mensuels` en `BUILD IMMEDIATE REFRESH ON DEMAND` pour le reporting lent.
> - **Une CTE récursive** qui parcourt la hiérarchie centre → station → fenêtre avec indentation visuelle.
> - **Des fonctions analytiques** : `LAG`/`LEAD` pour comparer deux fenêtres successives, `RANK` pour le classement, et `SUM OVER ROWS UNBOUNDED PRECEDING` pour les cumuls glissants.
> - **Un MERGE** qui synchronise un flux IoT vers la table SATELLITE — update si le satellite existe, insert sinon.
> - **6 index stratégiques** dont un **index fonctionnel** sur `TRUNC(datetime_debut, 'MM')` pour le reporting mensuel.
> - **Un EXPLAIN PLAN avant/après** qui montre comment on est passé de `TABLE ACCESS FULL` à `INDEX RANGE SCAN` sur la requête de reporting."

**Transition :**
> "On bascule côté Android."

---

## Slide 11 — MVVM (`SceneMVVM`) · ~60 s

**À l'écran :** trois couches Model / ViewModel / View avec leurs caractéristiques.

**Ce qu'on dit :**
> "Côté Android, architecture MVVM en trois couches.
>
> - **Models** : data classes Kotlin et enums, **strictement alignés sur les tables Oracle**. Les champs nullables sont aux mêmes endroits que dans le DDL.
> - **ViewModel** : expose des `StateFlow`. Le filtrage est calculé via `combine()` qui fusionne le flux de la query texte et celui du statut sélectionné. La validation client de RG-F04 — durée entre 1 et 900 secondes — est faite ici, avant l'aller-retour réseau.
> - **Vue** : Jetpack Compose pur. Aucune logique métier dans les Composables, juste de l'observation d'état.
>
> Le repository utilise une stratégie **Cache-First** : Room en lecture, Retrofit en refresh."

**Transition :**
> "Plutôt que de dérouler des slides d'écrans, on passe à la démo."

---

## Slide 12 — Demo (`SceneDemo`) · ~5 min de démo

**À l'écran :** mot "DEMO." centré, mention "Dashboard · Detail · Planning · Map · AR".

**Ce qu'on dit (et ce qu'on montre en live) :**

1. **Dashboard** — recherche en temps réel, filtres par statut, compteur "{X}/{N} opérationnels". Mettre en avant SAT-005 grisé (désorbité).
2. **Detail** sur SAT-001 — cinq sections : statut, télémétrie, instruments, missions, bouton Signaler une anomalie.
3. **Planning** — sélecteur de station, liste triée, **création d'une fenêtre avec une durée de 1000s** → message d'erreur RG-F04 affiché côté client. Création valide à 300s → fenêtre apparaît.
4. **Map** osmdroid — marqueurs colorés des stations, tap sur GS-SGP-01 → infobulle "En maintenance".
5. **AR** — pointer le téléphone (ou écran preview) vers le ciel, marqueurs satellites apparaissent.
6. **Test du mode hors-ligne** — couper le Wi-Fi, l'app reste utilisable, bannière "Mode hors-ligne" en haut.

**Transition :**
> "C'est précisément ce mode hors-ligne qui matérialise la synergie ALTN82/ALTN83."

---

## Slide 13 — Cache-First (`SceneCacheFirst`) · ~75 s

**À l'écran :** diagramme UI → ROOM → API + 3 colonnes (Flow, Bannière, Synergie).

**Ce qu'on dit :**
> "Le **Cache-First avec Room**, c'est le morceau pédagogique central de la partie Android.
>
> Le flow : **l'UI lit toujours Room en priorité** — c'est un `Flow<List<SatelliteEntity>>` réactif. **En parallèle**, le repository déclenche un refresh API en arrière-plan qui met à jour Room. Si l'API est indisponible, on capture l'exception, on bascule `isOffline = true`, et la bannière "Mode hors-ligne" apparaît. L'app reste utilisable.
>
> **Et c'est ici que la synergie ALTN82/ALTN83 prend tout son sens** : ce Cache-First côté mobile est l'**équivalent direct** de la fragmentation horizontale plus réplique locale en lecture seule, qu'on avait théorisée à la Phase 1 ALTN83 pour Singapour. La même réponse architecturale, déclinée dans deux mondes différents. Et c'est commenté explicitement dans `NanoOrbitRepository.getSatellitesFlow()` du code."

**Transition :**
> "Pour le bonus, on est allés plus loin que le sujet."

---

## Slide 14 — AR Sky-Track (`SceneAR`) · ~90 s

**À l'écran :** colonne gauche — CelesTrak TLE/OMM, ~150 satellites visuels. Colonne droite — viewport AR avec marqueurs satellite et coordonnées AZ/EL.

**Ce qu'on dit :**
> "Le bonus AR Sky-Track, c'est un **5ᵉ écran ajouté hors sujet**. L'idée : pointer le téléphone vers le ciel et voir les satellites visibles à l'œil nu en surimpression sur le flux caméra.
>
> Les données viennent de **CelesTrak**, qui publie en JSON les éléments orbitaux (TLE/OMM) de ~150 satellites visibles, mis à jour toutes les 8 heures. Pour chaque satellite, on **propage l'orbite**, on **lit l'orientation du téléphone** via le capteur `TYPE_ROTATION_VECTOR`, on **lit la position GPS** de l'observateur, et on projette dans le champ de vision de la caméra — HFOV 65° / VFOV 50°. Tick de 500 ms.
>
> Si l'écart angulaire entre le satellite et l'axe optique caméra rentre dans le FOV, on dessine un marqueur. Sinon on dessine un indicateur sur le bord de l'écran dans la direction du satellite, pour aider l'utilisateur à le retrouver."

**Transition :**
> "Trois frictions transverses méritent d'être mentionnées."

---

## Slide 15 — Problèmes transverses (`SceneProblems`) · ~75 s

**À l'écran :** trois cards — Auto-init Oracle, Accents Oracle/Kotlin, .env partagé.

**Ce qu'on dit :**
> "Trois décisions DevOps qui ont tout changé.
>
> **Un**, l'**auto-init Oracle**. Au début, premier `docker compose up` = base vide, il fallait se connecter en sqlplus, créer le user, exécuter DDL puis DML — 30 minutes par développeur. On a monté `scripts/01_import_schema.sh` dans `/opt/oracle/scripts/startup` pour qu'il s'exécute automatiquement au premier démarrage du conteneur. Setup divisé par 30.
>
> **Deux**, le **mapping des accents**. Les CHECK Oracle contiennent `'Opérationnel'` avec accent. Les enums Kotlin n'autorisent ni accents ni espaces. On fait la translation côté API Python, dans un dictionnaire dédié. Sans ça, `enumValueOf('Opérationnel')` crashe à la désérialisation.
>
> **Trois**, le **.env partagé** entre Docker et Android. L'URL `API_BASE_URL` est lue à la fois par `docker-compose.yml` et par `BuildConfig` côté Android. Une seule source de vérité, pas de désynchronisation possible entre les environnements."

**Transition :**
> "Récapitulons."

---

## Slide 16 — Questions (`SceneQuestions`) · transition Q&A

**À l'écran :** "QUESTIONS." en très grand + équipe en pied.

**Ce qu'on dit :**
> "Voilà. NanoOrbit, c'est ~3 000 lignes de Kotlin, ~600 lignes de SQL/PL/SQL, ~300 lignes de Python. **Excellence sur les deux modules** plus deux livrables hors sujet : le backend Flask et le bonus AR. Merci pour votre attention. Nous sommes prêts pour vos questions."

---

# Questions techniques anticipées

> Questions probables d'un jury technique, classées par thème, avec réponses prêtes.

## Côté ALTN83 — Bases de données

### Q1 — "Pourquoi avoir choisi la fragmentation horizontale plutôt que verticale ?"
> Parce que le sujet demande explicitement que **Singapour puisse continuer à planifier ses propres fenêtres** en autonomie en cas de coupure. C'est la planification — donc une partie des **lignes** de `FENETRE_COM` — qui doit être locale. La fragmentation verticale aurait coupé la table par colonnes, ce qui n'a pas de sens fonctionnel : on aurait toujours besoin d'aller chercher les autres colonnes ailleurs pour faire un INSERT complet. Horizontale = chaque centre a un fragment autonome, et les tables globales (satellites, stations) sont répliquées en lecture seule.

### Q2 — "Pourquoi `FENETRE_COM` en local et pas en global ?"
> Parce que c'est la table qui change le plus souvent — c'est la table opérationnelle. Si elle était globale, chaque INSERT à Singapour devrait remonter au maître. En cas de coupure, plus de planification possible. En local, Singapour reste autonome ; les fragments seront synchronisés au retour de la connectivité.

### Q3 — "Comment vous gérez la cohérence si Paris et Houston modifient le même satellite en même temps ?"
> Sur les **tables globales**, on est en réplication **maître-esclaves** : seules les écritures sur le nœud maître sont autorisées, les autres centres lisent une réplique. Ça élimine le conflit. Si on voulait du multi-maître, il faudrait un protocole **2PC (Two-Phase Commit)**, qu'on a documenté dans la note d'architecture comme "scénario 1" mais pas implémenté — c'est une décision de simplicité.

### Q4 — "Pourquoi 5 triggers et pas une procédure unique ?"
> Parce qu'un trigger garantit la règle métier **quel que soit le chemin d'écriture** : `INSERT` direct depuis sqlplus, depuis le package, depuis l'API Flask, depuis n'importe quel client SQL. Une procédure ne s'applique que si on l'appelle explicitement. T1 protégera toujours contre l'INSERT d'une fenêtre sur SAT-005, même si quelqu'un fait un `INSERT` à la main.

### Q5 — "Vous avez parlé d'ORA-04091 — vous pouvez expliquer ?"
> Oracle empêche un trigger row-level d'interroger la table sur laquelle il est posé, parce que pendant l'INSERT la table est dans un état transitoire incohérent. Notre solution : on compare le `:NEW` aux lignes déjà engagées via `NVL(id_fenetre, -1) <> :NEW.id_fenetre`. Une autre solution aurait été un trigger compound ou un trigger statement-level avec collection — plus complexe pour le même résultat.

### Q6 — "Pourquoi un index fonctionnel `TRUNC(datetime_debut, 'MM')` plutôt qu'un index simple sur `datetime_debut` ?"
> Parce que la requête de reporting fait `GROUP BY TRUNC(datetime_debut, 'MM')`. Un index simple sur `datetime_debut` aurait été ignoré par l'optimiseur, parce que la fonction `TRUNC` casse l'ordre. Avec un index fonctionnel, Oracle peut utiliser directement les valeurs tronquées pré-calculées. EXPLAIN PLAN passe de `TABLE ACCESS FULL` à `INDEX RANGE SCAN`.

### Q7 — "C'est quoi la différence entre une vue et une vue matérialisée ?"
> Une vue est une requête SELECT sauvegardée — elle est **recalculée à chaque appel**. Une vue matérialisée stocke physiquement le résultat sur disque ; elle est **lue instantanément** mais doit être rafraîchie (`REFRESH ON DEMAND` ou `ON COMMIT`). On a choisi `REFRESH ON DEMAND` pour `mv_volumes_mensuels` parce que le reporting est consulté plus souvent qu'il n'est mis à jour, et qu'on accepte une latence de fraîcheur.

### Q8 — "Votre package `pkg_nanoOrbit`, vous gérez les exceptions comment ?"
> On capture explicitement les codes ORA-20100 à -20999 — qui sont nos erreurs métier custom — et on les relance pour que l'appelant les voie. Pour `OTHERS`, on log et on relance. Au début on capturait `OTHERS` trop largement, ce qui masquait les erreurs métier des triggers. La leçon : ne jamais avaler `OTHERS` sans relancer.

---

## Côté ALTN82 — Android

### Q9 — "Pourquoi MVVM et pas MVC ou MVP ?"
> MVVM est l'architecture **recommandée par Google** pour Android moderne, particulièrement avec Compose. Le ViewModel survit aux rotations d'écran sans qu'on ait à gérer manuellement, et `StateFlow` s'intègre nativement à Compose via `collectAsStateWithLifecycle`. MVC aurait été plus difficile à tester et MVP plus verbeux.

### Q10 — "Pourquoi Cache-First plutôt que Network-First ?"
> Cache-First donne une **UI immédiate** : pas d'écran de chargement, l'app est utilisable même hors-ligne. Network-First afficherait un spinner à chaque ouverture, ce qui dégrade l'expérience. Le compromis, c'est qu'on peut afficher des données légèrement périmées pendant quelques secondes avant le refresh — acceptable pour un Ground Control où la fréquence de mise à jour n'est pas critique.

### Q11 — "Vous avez fait la validation RG-F04 côté client ET côté serveur — pourquoi pas un seul ?"
> Parce que ce sont deux usages différents. **Côté client**, c'est de l'UX — on évite un aller-retour réseau pour une erreur évidente. **Côté serveur** (CHECK Oracle), c'est de la sécurité — on ne fait pas confiance au client. **Le serveur reste l'autorité**. Si on avait juste la validation client, n'importe qui pourrait insérer une fenêtre de 10 000 secondes via curl. Si on avait juste la validation serveur, l'UX serait dégradée par les allers-retours.

### Q12 — "Comment Compose sait quand recomposer ?"
> Compose suit les **lectures d'état** au moment de la composition. Quand un `State<T>` change, Compose **invalide les Composables qui l'ont lu** et les réexécute. C'est de l'observabilité fine — un Composable qui ne lit pas une variable n'est pas recomposé même si elle change. C'est la raison pour laquelle on utilise `derivedStateOf` quand on veut éviter les recompositions inutiles.

### Q13 — "Pourquoi Room et pas SharedPreferences ou un fichier JSON ?"
> Pour trois raisons : **typage** (entités fortement typées avec migrations), **requêtes** (DAO avec `@Query`, support des jointures et transactions), et surtout **réactivité** — Room expose des `Flow` qui se déclenchent automatiquement à chaque INSERT/UPDATE. Avec SharedPreferences, il faudrait gérer manuellement les notifications de changement.

### Q14 — "Pourquoi osmdroid plutôt que Google Maps ?"
> Pas de clé API, pas de quota, pas de dépendance Google Play Services. Et `osmdroid` est totalement open-source. Pour un projet pédagogique sans contrainte de tile-style premium, c'est largement suffisant. La seule subtilité : il faut configurer un user-agent custom pour ne pas se faire bloquer par les serveurs OSM.

### Q15 — "Vos DTOs séparés du domaine, c'est pas du code en double ?"
> Oui c'est plus de code, et c'est volontaire. Si demain l'API change un nom de champ, on ajuste le DTO sans impacter le ViewModel ni la Vue. Si on avait directement déserialisé dans la `data class` métier, un changement d'API casserait toute l'app. C'est un coût d'écriture pour un gain de **résilience aux changements externes**.

### Q16 — "Comment vous testeriez ce ViewModel ?"
> Le ViewModel n'a aucune dépendance Android — il dépend juste du Repository qu'on injecte par interface. Donc en test unitaire JVM : on injecte un faux Repository qui renvoie des Flows contrôlés, on appelle `viewModel.onSearchQueryChange("alpha")`, et on vérifie que `filteredSatellites.value` ne contient que les satellites avec "alpha" dans le nom. Pas besoin d'émulateur.

---

## Côté backend / API / DevOps

### Q17 — "Pourquoi Flask et pas Spring Boot ou Node ?"
> Flask, c'est ~30 lignes pour exposer 8 endpoints. Le driver `oracledb` officiel en Python est mature. Pour un pont CRUD sans logique métier, c'est le moins de code possible. Spring Boot aurait été lourd pour un service de cette taille, Node nous aurait obligés à mélanger un troisième langage dans le projet.

### Q18 — "Comment vous gérez le démarrage Oracle qui est lent ?"
> `depends_on` Docker n'attend que le démarrage du process, pas la disponibilité TNS. On a une boucle `wait_for_db(max_retries=30, delay=5)` dans `app.py` qui retente la connexion 30 fois avec 5 secondes d'intervalle avant de lancer Flask. Plus robuste qu'un healthcheck shell parce que ça reflète exactement ce que l'app fera au runtime.

### Q19 — "Pas trop risqué d'exposer la base via Flask sans authentification ?"
> Pour le projet pédagogique, oui c'est ouvert. En production, on ajouterait une authentification JWT côté API et un user Oracle en lecture seule. Le code est structuré pour que l'ajout d'un middleware d'auth soit indolore — décorateur sur chaque route.

### Q20 — "Comment vous synchronisez l'auto-init Oracle entre les machines de l'équipe ?"
> Les scripts d'init sont versionnés dans `scripts/01_import_schema.sh` qui est mounté en volume. Tant que tout le monde fait `git pull`, le script est aligné. Le volume `oracle_data` est local à chaque machine — si un dev veut repartir d'un état propre, `docker compose down -v` détruit le volume et le prochain `up` rejoue le script.

---

## Côté bonus AR

### Q21 — "Pourquoi un propagateur képlérien et pas SGP4 ?"
> SGP4 prend en compte les perturbations — aplatissement de la Terre J2, traînée atmosphérique, gravitation de la Lune et du Soleil. C'est la référence pour la précision longue durée (jours, semaines). Mais c'est ~500 lignes à implémenter correctement. Le propagateur képlérien fait ~50 lignes, et il est précis à quelques degrés près sur quelques heures. Vu qu'on rafraîchit les TLE à chaque ouverture de la vue AR et que CelesTrak met à jour ses OMM toutes les 8 h, l'erreur résiduelle reste inférieure à la taille angulaire d'un marqueur.

### Q22 — "Comment vous résolvez l'équation de Kepler `M = E − e sin E` ?"
> Newton-Raphson. On part d'une estimation `E = M + e·sin(M)` — correction du premier ordre — et on itère 8 fois. La convergence est quadratique, donc 8 itérations suffisent largement pour `e < 0,3`. Au début on initialisait à `E = M` et ça divergeait pour `e ≥ 0,1`.

### Q23 — "Pourquoi le capteur `TYPE_ROTATION_VECTOR` plutôt qu'accéléromètre + magnétomètre directement ?"
> Parce que `TYPE_ROTATION_VECTOR` est un **capteur virtuel** : Android fait déjà la fusion accéléromètre + gyroscope + magnétomètre avec un filtre de Kalman ou équivalent. Latence et jitter bien meilleurs que ce qu'on coderait à la main. Et ça nous donne directement une matrice de rotation 3D, pas deux vecteurs à intégrer.

### Q24 — "Comment vous gérez les permissions caméra ?"
> Avec `rememberLauncherForActivityResult(ActivityResultContracts.RequestPermission())`. Si l'utilisateur refuse, on affiche un `PermissionRationale` qui explique pourquoi la caméra est requise et propose de relancer la demande. La géolocalisation est optionnelle — fallback sur Paris si pas de fix GPS.

### Q25 — "C'est précis à combien votre AR ?"
> Quelques degrés. Les sources d'erreur : magnétomètre non calibré (le user doit faire un ∞ avec le téléphone), HFOV/VFOV en dur à 65°/50° au lieu de lire `CameraCharacteristics`, et propagation képlérienne. Largement suffisant pour identifier visuellement un satellite. On a vérifié en croisant avec Stellarium sur l'ISS — coïncidence à 2-3° près.

---

## Côté synergie / projet global

### Q26 — "Concrètement, qu'est-ce qui prouve que les deux modules sont synchronisés ?"
> Trois choses. Un, les **identifiants** sont identiques : `SAT-001` à `SAT-005`, `GS-TLS-01`, `MSN-DEF-2022` apparaissent à la fois dans le CSV ALTN83 et dans le `MockData.kt` Android. Deux, **SAT-005 désorbité** est grisé dans l'UI et bloqué par T1 dans Oracle — on peut le démontrer en live. Trois, **RG-F04** est validée des deux côtés avec le même message d'erreur : on peut tester en saisissant 1000s dans Planning et l'erreur s'affiche sans aller-retour.

### Q27 — "Vous avez utilisé l'IA pour le projet ?"
> Pour la documentation et la rédaction des rapports, oui. Pour le code, on relit et on comprend chaque ligne — sinon on ne peut pas répondre à vos questions techniques. Les choix d'architecture (fragmentation horizontale, Cache-First, propagateur képlérien) sont des **décisions justifiées** qu'on peut défendre, pas des outputs aveugles.

### Q28 — "S'il fallait passer à 1 million de satellites, qu'est-ce qui casse en premier ?"
> Plusieurs choses. Côté Oracle : la table `FENETRE_COM` exploserait — il faudrait du **partitionnement par mois**. Côté API : un endpoint `GET /satellites` qui renvoie tout en une fois deviendrait infaisable — il faudrait de la **pagination**. Côté Android : la `LazyColumn` tient déjà mais le filtrage côté client deviendrait impraticable — il faudrait migrer vers de la recherche serveur. Côté AR : le rendu de 1M de marqueurs noierait l'écran — il faudrait clusteriser les satellites proches.

### Q29 — "Si le serveur central tombe en pleine présentation, qu'est-ce qui se passe ?"
> L'app continue de fonctionner. C'est exactement ce que démontre le Cache-First — on peut couper le Wi-Fi en pleine démo (et on le fera). La bannière "Mode hors-ligne" apparaît, la recherche et la consultation continuent, seule la création de fenêtre échouerait au moment de l'INSERT serveur (mais elle serait queueable si on voulait pousser plus loin).

### Q30 — "Qu'est-ce qui vous a pris le plus de temps ?"
> Honnêtement, **l'alignement du mock Android sur le CSV ALTN83**. C'était bête mais critique — on avait commencé avec des IDs `SAT-A`, `SAT-B`, et au moment de tester la synergie il a fallu tout réaligner ligne par ligne. Le commit `a60bbe2` est entièrement dédié à ça. La leçon retenue : dans un projet à plusieurs livrables couplés, **le jeu de données de référence doit être la source unique de vérité dès le jour 1**.

---

## Pièges à éviter pendant la présentation

- **Ne pas dire "c'est simple" ou "c'est facile"** : pour un évaluateur, ça signale qu'on n'a pas mesuré la complexité. Préférer "c'est une décision claire" ou "c'est un compromis bien défini".
- **Ne pas sur-vendre l'IA** : si quelqu'un demande, dire honnêtement qu'on l'a utilisée pour la doc et qu'on relit le code. Ne pas prétendre l'avoir tout codé à la main.
- **Ne pas s'embarquer sur SGP4** si la question vient — répondre la limitation assumée et arrêter là. Ne pas faire d'envolée scientifique sur la perturbation J2.
- **Ne pas lire les slides** — elles sont volontairement minimalistes, le contenu vit dans la voix.
- **Si une démo plante** : "C'est exactement le scénario qu'on a optimisé — regardez, l'app reste utilisable grâce au cache Room." Toujours retomber sur ses pattes.
- **Si on ne sait pas répondre** : "C'est une bonne question, on n'a pas creusé ce point — voici ce qu'on ferait pour y répondre." Mille fois mieux que d'inventer.

---

## Timing récapitulatif

| Slide | Durée | Cumul |
|---|---|---|
| 1. Cover | 30 s | 0:30 |
| 2. Contexte | 45 s | 1:15 |
| 3. Architecture | 60 s | 2:15 |
| 4. Stack | 45 s | 3:00 |
| 5. Phase 1 | 75 s | 4:15 |
| 6. MCD | 30 s | 4:45 |
| 7. Phase 2 Triggers | 75 s | 6:00 |
| 8. ORA-04091 | 75 s | 7:15 |
| 9. Phase 3 PL/SQL | 75 s | 8:30 |
| 10. Phase 4 Optim | 90 s | 10:00 |
| 11. MVVM | 60 s | 11:00 |
| 12. Demo | 5 min | 16:00 |
| 13. Cache-First | 75 s | 17:15 |
| 14. AR | 90 s | 18:45 |
| 15. Problèmes | 75 s | 20:00 |
| 16. Questions | — | Q&A |

**Cible : 18-20 minutes**, garder une marge pour la démo qui peut déborder.
