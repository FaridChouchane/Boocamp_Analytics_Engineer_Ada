# ❄️ Snowflake — Cours complet sur les Tasks
> **Guide débutant · Projet `health_app`** · Optimisé Notion · Réutilisable sur d'autres projets
---

## 📋 Table des matières

1. [Introduction aux Tasks](#1--introduction-aux-tasks)
2. [Exécuter les tasks d'un graph](#2--exécuter-les-tasks-dun-graph)
3. [Comportement des tasks en cas de bug](#3--comportement-des-tasks-en-cas-de-bug)
4. [Logging & qualité de la data](#4--logging--qualité-de-la-data)
5. [Gestion des exceptions](#5--gestion-des-exceptions)
6. [Introduction aux Streams](#6--introduction-aux-streams)
7. [Créer des tasks "Event Driven"](#7--créer-des-tasks-event-driven)
8. [La task Finalizer](#8--la-task-finalizer)
9. [Architecture du projet health_app](#9--architecture-du-projet-health_app)
10. [Glossaire complet](#10--glossaire-complet)
11. [Checklist de déploiement](#11--checklist-de-déploiement)
12. [Patterns réutilisables](#12--patterns-réutilisables)

---

## 1 · Introduction aux Tasks

### Objectifs de cette section
- Savoir créer une task
- Définir les options de configuration les plus importantes
- Définir l'intervalle d'exécution
- Définir le graphe de dépendance
- Suspendre et re-démarrer une task

---

### Qu'est-ce qu'une Task ?

Une **Task** dans Snowflake est une tâche SQL automatisée. Elle permet d'exécuter du code SQL (requêtes, procédures stockées…) automatiquement selon :
- un **planning** (ex : toutes les heures)
- ou un **déclencheur** (ex : quand une autre tâche se termine, ou quand un stream a des données)

### Créer une Task — syntaxe de base

```sql
CREATE OR ALTER TASK mon_schema.ma_task
    WAREHOUSE = COMPUTE_WH      -- moteur de calcul utilisé (consomme des crédits)
    SCHEDULE  = '1 HOURS'       -- fréquence d'exécution automatique
AS
    SELECT 1;                   -- code SQL exécuté à chaque déclenchement
```

### Options de configuration importantes

| Option | Description | Exemple |
|---|---|---|
| `WAREHOUSE` | Entrepôt de calcul qui exécute la task | `COMPUTE_WH` |
| `SCHEDULE` | Fréquence (root task uniquement) | `'1 HOURS'`, `'5 MINUTES'`, `'USING CRON 0 * * * * UTC'` |
| `AFTER` | Dépendance vers une autre task | `AFTER mon_schema.ma_root_task` |
| `WHEN` | Condition supplémentaire pour s'exécuter | `WHEN SYSTEM$STREAMHASDATA('mon_stream')` |
| `USER_TASK_TIMEOUT_MS` | Timeout en millisecondes | `3600000` (= 1h) |
| `SUSPEND_TASK_AFTER_NUM_FAILURES` | Suspension auto après N échecs | `3` |

### Intervalle d'exécution — 2 syntaxes

```sql
-- Syntaxe simple (minutes ou heures)
SCHEDULE = '30 MINUTES'
SCHEDULE = '1 HOURS'

-- Syntaxe CRON (plus flexible — format Unix standard)
-- CRON : minute heure jour_mois mois jour_semaine timezone
SCHEDULE = 'USING CRON 0 * * * * UTC'    -- toutes les heures pile
SCHEDULE = 'USING CRON 0 8 * * MON UTC'  -- tous les lundis à 8h UTC
SCHEDULE = 'USING CRON 0 0 1 * * UTC'    -- le 1er de chaque mois à minuit
```

### Définir un graphe de dépendance (DAG)

```sql
-- Root task : a un SCHEDULE, démarre tout le graphe
CREATE OR ALTER TASK raw.data_quality_task
    WAREHOUSE = COMPUTE_WH
    SCHEDULE  = '1 HOURS'
AS
    CALL raw.data_quality();

-- Child tasks : ont un AFTER, pas de SCHEDULE
CREATE OR ALTER TASK raw.hih_listener_manager
    WAREHOUSE = COMPUTE_WH
    AFTER raw.data_quality_task       -- ← dépendance
AS
    CALL raw.enrich_data('hih_listener_manager', 'HiH_ListenerManager');
```

> 💡 **Règle** : dans un DAG, **seule la root task** a un `SCHEDULE`. Les enfants ont uniquement `AFTER`. Une task ne peut pas avoir les deux.

### Cycle de vie d'une Task

```
[Création]
    │
    ▼
SUSPENDED ←──────────────────────────────────┐
    │                                         │
    │  ALTER TASK ... RESUME                  │  ALTER TASK ... SUSPEND
    ▼                                         │
RESUMED ─────────────────────────────────────┘
    │
    │  (selon SCHEDULE ou AFTER)
    ▼
RUNNING → SUCCEEDED / FAILED
```

### Suspendre et re-démarrer

```sql
-- Suspendre (nécessaire avant toute modification !)
ALTER TASK raw.data_quality_task SUSPEND;

-- Modifier la task
CREATE OR ALTER TASK raw.data_quality_task
    WAREHOUSE = COMPUTE_WH
    SCHEDULE  = '2 HOURS'   -- ← modification
AS ...;

-- Re-démarrer (enfants EN PREMIER, racine EN DERNIER)
ALTER TASK raw.hih_listener_manager  RESUME;
ALTER TASK raw.step_lsc              RESUME;
-- ... tous les enfants ...
ALTER TASK raw.data_quality_task     RESUME;  -- ← EN DERNIER
```

> ⚠️ **Règle absolue** : toujours `SUSPEND` avant de modifier une task. Si tu modifies une task active, Snowflake retourne une erreur.

---

## 2 · Exécuter les tasks d'un graph

### Objectifs de cette section
- Savoir exécuter des tasks
- Savoir débugger des erreurs dans les tasks
- Utiliser le task history pour retrouver les exécutions d'un task graph

---

### Exécution manuelle

```sql
-- Force le déclenchement immédiat sans attendre le planning
-- Snowflake déclenche la root task, puis toutes les enfants en cascade
EXECUTE TASK raw.data_quality_task;
```

> 💡 Très utile pour tester un nouveau DAG sans attendre la prochaine heure planifiée.

### Consulter l'historique d'exécution

```sql
-- Historique des tasks de la dernière heure
SELECT *
FROM TABLE(INFORMATION_SCHEMA.TASK_HISTORY(
    SCHEDULED_TIME_RANGE_START => DATEADD('hour', -1, current_timestamp())
))
WHERE schema_name = 'RAW'
ORDER BY SCHEDULED_TIME DESC;
```

**Colonnes clés à surveiller :**

| Colonne | Ce qu'elle indique |
|---|---|
| `NAME` | Nom de la task |
| `STATE` | `SUCCEEDED` / `FAILED` / `RUNNING` / `SKIPPED` |
| `SCHEDULED_TIME` | Quand la task était censée s'exécuter |
| `QUERY_START_TIME` | Quand elle a réellement démarré |
| `COMPLETED_TIME` | Quand elle a terminé |
| `ERROR_MESSAGE` | Détail de l'erreur si `STATE = FAILED` |
| `GRAPH_RUN_GROUP_ID` | ID unique du run complet du DAG |

### Débugger une erreur dans une task

```sql
-- Étape 1 : trouver les tasks en échec
SELECT name, state, error_message, scheduled_time
FROM TABLE(INFORMATION_SCHEMA.TASK_HISTORY(
    SCHEDULED_TIME_RANGE_START => DATEADD('hour', -1, current_timestamp())
))
WHERE state = 'FAILED'
  AND schema_name = 'RAW';

-- Étape 2 : vérifier les logs de la table raw.logging
SELECT *
FROM raw.logging
WHERE error_message IS NOT NULL
ORDER BY created_at DESC;

-- Étape 3 : lister toutes les tasks et leur statut actuel
SHOW TASKS IN SCHEMA HEALTH_APP.RAW;
```

---

## 3 · Comportement des tasks en cas de bug

### Objectifs de cette section
- Comprendre le comportement par défaut en cas d'erreur
- Savoir quand utiliser l'option `RETRY LAST`
- Comprendre l'intérêt de découpler la définition des tasks de la logique de transformation

---

### Comportement par défaut

Quand une task **enfant** échoue dans un DAG :
- Elle est marquée `FAILED` dans `TASK_HISTORY`
- Les autres tasks enfants (parallèles) **continuent quand même**
- La root task du prochain run repart **depuis le début**

```
Run #1 : data_quality_task → OK
         ├── hih_listener_manager  → ✅ OK
         ├── step_lsc              → ❌ FAILED
         └── step_screenutil       → ✅ OK  (continue malgré l'échec de step_lsc)

Run #2 (automatique) : repart depuis data_quality_task (tout recommence)
```

### L'option `RETRY LAST`

Permet de **relancer uniquement les tasks qui ont échoué** lors du dernier run, sans tout relancer depuis le début.

```sql
-- Relancer uniquement les tasks FAILED du dernier run
EXECUTE TASK raw.data_quality_task RETRY LAST;
```

**Quand l'utiliser ?**
- Quand le problème était temporaire (réseau, warehouse suspendu…)
- Quand les tasks qui ont réussi ne doivent pas être re-exécutées (ex : éviter les doublons)
- Quand le DAG est long et que relancer tout depuis le début est coûteux

### Découpler la définition des tasks de la logique

**Mauvaise pratique ❌ — logique dans la task directement**

```sql
CREATE OR ALTER TASK raw.step_lsc
    WAREHOUSE = COMPUTE_WH
    AFTER raw.data_quality_task
AS
    -- toute la logique est ici : difficile à tester, à réutiliser, à débugger
    INSERT INTO staging.step_lsc (event_timestamp, process_id, log_trigger, message)
    SELECT re.event_timestamp, re.process_id, ...
    FROM raw.raw_events re
    LEFT JOIN raw.data_anomalies da ON re.event_id = da.event_id
    WHERE re.process_name = 'Step_LSC'
      AND da.event_id IS NULL;
```

**Bonne pratique ✅ — logique dans une procédure, task = planificateur**

```sql
-- La procédure contient la logique (testable avec CALL indépendamment)
CREATE OR REPLACE PROCEDURE raw.enrich_data(table_name STRING, process_name STRING, run_id STRING)
    ...

-- La task ne fait QUE déclencher la procédure
CREATE OR ALTER TASK raw.step_lsc
    WAREHOUSE = COMPUTE_WH
    AFTER raw.data_quality_task
AS
DECLARE
    run_id STRING := SYSTEM$TASK_RUNTIME_INFO('CURRENT_TASK_GRAPH_RUN_GROUP_ID');
BEGIN
    CALL raw.enrich_data('step_lsc', 'Step_LSC', :run_id);
END;
```

**Avantages du découplage :**
- On peut tester la procédure avec `CALL` sans déclencher toute la task
- La logique est réutilisable dans d'autres contextes
- Plus facile à débugger (on isole le problème)
- Plus facile à faire évoluer (modifier la procédure sans toucher à la task)

---

## 4 · Logging & qualité de la data

### Objectifs de cette section
- Mettre en place du logging dans les tasks
- Définir une procédure pour tester la qualité de la data reçue

---

### La table de logging

```sql
CREATE OR ALTER TABLE raw.logging (
    created_at          TIMESTAMP DEFAULT CURRENT_TIMESTAMP(),
    graph_run_group_id  STRING,              -- ID unique du run complet du DAG
    table_name          STRING,              -- quelle table a été enrichie
    n_rows              NUMBER,              -- nb de lignes insérées (NULL si erreur)
    error_message       STRING DEFAULT NULL  -- NULL si succès, message si échec
);
```

**Comment lire les logs :**

| `n_rows` | `error_message` | Signification |
|---|---|---|
| `5842` | `NULL` | ✅ Succès — 5842 lignes insérées |
| `NULL` | `"Table not found..."` | ❌ Échec — voir le message d'erreur |
| `0` | `NULL` | ⚠️ Aucune donnée nouvelle ce run (normal) |

### La procédure de logging

```sql
CREATE OR REPLACE PROCEDURE raw.log_results(
    graph_run_group_id  STRING,
    table_name          STRING,
    n_rows              NUMBER,
    error_message       STRING
)
RETURNS STRING
LANGUAGE SQL
EXECUTE AS CALLER
AS $$
    INSERT INTO raw.logging (graph_run_group_id, table_name, n_rows, error_message)
    VALUES (:graph_run_group_id, :table_name, :n_rows, :error_message);
$$;
```

### La procédure de data quality

```sql
CREATE OR REPLACE PROCEDURE raw.data_quality(graph_run_group_id STRING)
RETURNS STRING
LANGUAGE SQL
EXECUTE AS CALLER
AS $$
BEGIN
    Let nbre_lignes_incorrectes INT := 0;

    -- Détecte les anomalies : timestamp invalide OU process_name invalide
    INSERT INTO raw.data_anomalies (event_id, is_correct_timestamp, is_correct_process_name, graph_run_group_id)
    WITH source AS (
        SELECT
            event_id,
            raw.check_correct_timestamp(event_timestamp)   AS is_correct_timestamp,
            raw.check_correct_process_name(process_name)   AS is_correct_process_name
        FROM raw.raw_events
    )
    SELECT *, :graph_run_group_id AS graph_run_group_id
    FROM source
    WHERE is_correct_timestamp = FALSE
       OR is_correct_process_name = FALSE;

    nbre_lignes_incorrectes := SQLROWCOUNT;
    RETURN :nbre_lignes_incorrectes;
END;
$$;
```

### Le `graph_run_group_id` — concept clé

```
Run du DAG #1 (10h00)
├── data_quality_task     → graph_run_group_id = "abc-123"
├── hih_listener_manager  → graph_run_group_id = "abc-123"  ← même ID !
└── step_lsc              → graph_run_group_id = "abc-123"  ← même ID !

Run du DAG #2 (11h00)
├── data_quality_task     → graph_run_group_id = "xyz-456"  ← nouvel ID
├── hih_listener_manager  → graph_run_group_id = "xyz-456"
└── step_lsc              → graph_run_group_id = "xyz-456"
```

Cela permet de retrouver **tout ce qui s'est passé lors d'un run** avec un simple filtre :

```sql
SELECT * FROM raw.logging WHERE graph_run_group_id = 'abc-123';
```

---

## 5 · Gestion des exceptions

### Objectifs de cette section
- Savoir attraper les exceptions dans Snowflake avec du SQL
- Définir et lancer des exceptions personnalisées
- Utiliser la variable `SQLERRM` pour comprendre la cause d'une exception

---

### Structure d'un bloc avec exceptions

```sql
DECLARE
    -- Déclarer une exception personnalisée
    -- Syntaxe : EXCEPTION (code_erreur, 'message')
    -- Le code négatif (-9999) évite les conflits avec les codes système Snowflake
    mon_exception EXCEPTION (-9999, 'Description de mon erreur');

BEGIN
    -- Code qui peut potentiellement échouer
    INSERT INTO ma_table ...;

EXCEPTION
    -- WHEN OTHER THEN = attrape TOUTES les exceptions non gérées explicitement
    WHEN OTHER THEN
        -- SQLERRM : variable système contenant le MESSAGE TEXTE de l'erreur
        -- Exemple : "Table 'STAGING.STEP_LSC' does not exist"
        LET message STRING := SQLERRM;

        -- Faire quelque chose avec l'erreur (ex: logger)
        INSERT INTO raw.logging (error_message) VALUES (:message);

        -- RAISE : re-propage l'exception vers l'appelant
        -- Sans RAISE, l'erreur est "avalée" et l'appelant croit que tout s'est bien passé
        RAISE mon_exception;

END;
```

### Les 3 variables système à connaître

| Variable | Type | Contient | Disponible |
|---|---|---|---|
| `SQLROWCOUNT` | Automatique | Nb de lignes affectées par le dernier DML | Après chaque INSERT/UPDATE/DELETE |
| `SQLERRM` | Contexte d'erreur | Message texte de l'erreur | Dans le bloc `EXCEPTION` uniquement |
| `SQLCODE` | Contexte d'erreur | Code numérique de l'erreur | Dans le bloc `EXCEPTION` uniquement |

### Exemple complet — `enrich_data` avec exceptions

```sql
CREATE OR REPLACE PROCEDURE raw.enrich_data(
    table_name          STRING,
    process_name        STRING,
    graph_run_group_id  STRING
)
RETURNS STRING
LANGUAGE SQL
EXECUTE AS CALLER
AS $$
DECLARE
    full_table_name  STRING := CONCAT('staging.', :table_name);
    insert_exception EXCEPTION (-9999, 'Exception in data loading into staging tables');

BEGIN
    Let n_rows INT := 0;

    INSERT INTO IDENTIFIER(:full_table_name) (event_timestamp, process_id, log_trigger, message)
    WITH source AS (
        SELECT
            re.event_timestamp,
            re.process_id,
            raw.extract_log_trigger(re.message) AS log_trigger,
            raw.extract_log_message(re.message) AS message
        FROM raw.raw_events re
        LEFT JOIN raw.data_anomalies da ON re.event_id = da.event_id
        WHERE re.process_name = :process_name
          AND da.event_id IS NULL
    )
    SELECT event_timestamp, process_id, log_trigger, message
    FROM source;

    n_rows := SQLROWCOUNT;

    -- Log du SUCCÈS
    CALL raw.log_results(:graph_run_group_id, :table_name, :n_rows, NULL);
    RETURN :n_rows;

EXCEPTION
    WHEN OTHER THEN
        -- 1. Capture le message d'erreur (SQLERRM)
        -- 2. Log l'ÉCHEC avant de propager
        -- 3. RAISE re-propage → la Task est marquée FAILED dans TASK_HISTORY
        CALL raw.log_results(:graph_run_group_id, :table_name, NULL, :SQLERRM);
        RAISE insert_exception;

END;
$$;
```

### Flux de gestion d'erreur visualisé

```
INSERT échoue
      │
      ▼
WHEN OTHER THEN
      │
      ├─→ SQLERRM  →  "Table STAGING.STEP_LSC does not exist"
      │
      ├─→ log_results(NULL, SQLERRM)
      │          ↓
      │     raw.logging:
      │     n_rows        = NULL
      │     error_message = "Table STAGING.STEP_LSC does not exist"
      │
      └─→ RAISE insert_exception
                 ↓
           Task Snowflake → STATE = FAILED
                 ↓
           Visible dans TASK_HISTORY avec error_message
```

---

## 6 · Introduction aux Streams

### Objectifs de cette section
- Comprendre le concept de Stream
- Savoir définir un stream
- Comprendre les propriétés des streams

---

### Qu'est-ce qu'un Stream ?

Un **Stream** est un objet Snowflake qui **capture les changements** (INSERT, UPDATE, DELETE) survenus sur une table depuis la dernière fois qu'il a été lu. C'est un mécanisme de **Change Data Capture (CDC)**.

```
Table raw_events
│
│  nouvelles lignes insérées →  [ ligne A ]
│                                [ ligne B ]
│
▼
Stream raw_events_stream
→ contient uniquement les lignes NOUVELLES ou MODIFIÉES depuis la dernière lecture
```

### Créer un Stream

```sql
-- Stream sur une table (capture les INSERTs, UPDATEs, DELETEs)
CREATE OR REPLACE STREAM raw.raw_events_stream
    ON TABLE raw.raw_events;

-- Stream append-only (capture uniquement les INSERTs — plus léger)
CREATE OR REPLACE STREAM raw.raw_events_stream
    ON TABLE raw.raw_events
    APPEND_ONLY = TRUE;   -- recommandé si la table ne reçoit que des INSERT
```

### Propriétés et colonnes spéciales d'un Stream

Quand tu lis un stream, des colonnes supplémentaires sont disponibles :

| Colonne | Type | Contient |
|---|---|---|
| `METADATA$ACTION` | STRING | `INSERT` ou `DELETE` |
| `METADATA$ISUPDATE` | BOOLEAN | `TRUE` si c'est la partie UPDATE d'un changement |
| `METADATA$ROW_ID` | STRING | Identifiant unique de la ligne dans le stream |

```sql
-- Lire les nouvelles données d'un stream
SELECT
    event_id,
    event_timestamp,
    process_name,
    METADATA$ACTION    AS action,         -- 'INSERT' ou 'DELETE'
    METADATA$ISUPDATE  AS is_update
FROM raw.raw_events_stream;
```

### Comportement important à connaître

> ⚠️ **Un stream est consommé quand on le lit dans une transaction DML (INSERT, UPDATE, DELETE, MERGE).** Un simple `SELECT` ne consomme PAS le stream.

```sql
-- ❌ Ne consomme PAS le stream
SELECT * FROM raw.raw_events_stream;

-- ✅ Consomme le stream (les données lues disparaissent du stream)
INSERT INTO staging.raw_events_copy
SELECT * FROM raw.raw_events_stream;
```

### Vérifier si un stream a des données

```sql
-- Retourne TRUE si le stream contient des données non consommées
SELECT SYSTEM$STREAMHASDATA('raw.raw_events_stream');
```

---

## 7 · Créer des tasks "Event Driven"

### Objectifs de cette section
- Comprendre l'intérêt d'une architecture réactive
- Utiliser la fonction `SYSTEM$STREAMHASDATA`
- Débugger des tasks event-driven

---

### Architecture planifiée vs réactive

**Architecture planifiée (Schedule) :**
```
Toutes les heures →  Task s'exécute
                      Si données : traite
                      Si pas de données : inutile mais consomme quand même le warehouse
```

**Architecture réactive (Event Driven) :**
```
Nouvelles données dans le stream →  Task se déclenche automatiquement
                                      Traite uniquement s'il y a quelque chose à faire
                                      Pas d'exécution inutile → économie de crédits
```

### Créer une task déclenchée par un stream

```sql
-- Root task avec SCHEDULE + condition WHEN
CREATE OR ALTER TASK raw.data_quality_task
    WAREHOUSE = COMPUTE_WH
    SCHEDULE  = '1 MINUTES'   -- vérifie toutes les minutes s'il y a des données
    WHEN SYSTEM$STREAMHASDATA('raw.raw_events_stream')  -- ← ne s'exécute QUE s'il y a des données
AS
DECLARE
    run_id STRING := SYSTEM$TASK_RUNTIME_INFO('CURRENT_TASK_GRAPH_RUN_GROUP_ID');
BEGIN
    CALL raw.data_quality(:run_id);
END;
```

> 💡 **Astuce** : Le `SCHEDULE` définit la fréquence de vérification, pas la fréquence d'exécution. Si `SYSTEM$STREAMHASDATA` retourne `FALSE`, la task est `SKIPPED` sans consommer de crédits warehouse.

### `SYSTEM$STREAMHASDATA` expliqué

```sql
-- Retourne TRUE si le stream a des données non consommées
SELECT SYSTEM$STREAMHASDATA('raw.raw_events_stream');
-- → TRUE  = il y a des nouvelles données à traiter
-- → FALSE = rien de nouveau, la task sera SKIPPED

-- Utilisation dans une task
WHEN SYSTEM$STREAMHASDATA('raw.raw_events_stream')
```

### Débugger une task event-driven

```sql
-- Vérifier si les tasks sont SKIPPED (stream vide) ou FAILED
SELECT name, state, error_message, scheduled_time
FROM TABLE(INFORMATION_SCHEMA.TASK_HISTORY(
    SCHEDULED_TIME_RANGE_START => DATEADD('hour', -1, current_timestamp())
))
WHERE schema_name = 'RAW'
ORDER BY scheduled_time DESC;

-- STATE = 'SKIPPED' → normal, le stream était vide à ce moment-là
-- STATE = 'SUCCEEDED' → données traitées
-- STATE = 'FAILED' → erreur, voir error_message

-- Vérifier manuellement si le stream a des données
SELECT SYSTEM$STREAMHASDATA('raw.raw_events_stream');

-- Voir les données actuelles dans le stream
SELECT COUNT(*) FROM raw.raw_events_stream;
```

---

## 8 · La task Finalizer

### Objectifs de cette section
- Définir une task pour faire le ménage à la fin d'un DAG
- Comprendre le rôle et la syntaxe du Finalizer
- Débugger les tasks Finalizer

---

### Qu'est-ce qu'un Finalizer ?

Le **Finalizer** est une task spéciale qui s'exécute **toujours en dernier**, après que **toutes les tasks du DAG ont terminé**, qu'elles aient réussi ou échoué.

```
identify_new_data_task
        │
        ├── hih_listener_manager  ✅
        ├── step_lsc              ❌ (échec)
        └── step_screenutil       ✅
                │
                │ (toutes terminées, succès ou échec)
                ▼
        [finalize_transformation]  ← s'exécute TOUJOURS
```

**Usages typiques du Finalizer dans ce projet :**
- Compter les erreurs du run depuis `raw.logging`
- Enregistrer le statut global (`SUCCEEDED` / `FAILED`) dans `raw.transformation_pipeline_status`
- Nettoyer la table de staging intermédiaire `raw.data_to_process` si tout s'est bien passé
- Propager une exception si des erreurs ont été détectées

---

### Code complet — Le Finalizer du projet `health_app`

#### Étape 1 — Suspendre la root task

```sql
-- Obligatoire avant toute modification du DAG
ALTER TASK raw.identify_new_data_task SUSPEND;
```

#### Étape 2 — Table de suivi du pipeline

```sql
-- Stocke le statut global de chaque run du DAG
-- Un enregistrement par run : démarré à, terminé à, statut final
CREATE OR ALTER TABLE raw.transformation_pipeline_status (
    graph_run_group_id  STRING,     -- ID unique du run
    started_at          TIMESTAMP,  -- heure de démarrage (depuis la root task)
    finished_at         TIMESTAMP,  -- heure de fin (enregistrée par le Finalizer)
    status              STRING      -- 'SUCCEEDED' ou 'FAILED'
);
```

#### Étape 3 — La procédure `finalize_transformation`

```sql
CREATE OR REPLACE PROCEDURE raw.finalize_transformation(
    graph_run_group_id  STRING,    -- ID du run, passé depuis la task
    started_at          TIMESTAMP  -- timestamp de démarrage de la root task
)
RETURNS STRING
LANGUAGE SQL
EXECUTE AS CALLER
AS
$$
DECLARE
    -- Exception custom déclenchée si des erreurs sont détectées dans le run
    pipeline_exception EXCEPTION (-20002, 'Exception in the transformation pipeline');

BEGIN
    -- Initialise le compteur d'erreurs à 0
    LET n_errors INT := 0;

    -- Étape A : Compter les erreurs du run courant dans raw.logging
    -- error_message IS NOT NULL = une task a planté lors de ce run
    SELECT COUNT(*) INTO n_errors
    FROM raw.logging
    WHERE graph_run_group_id = :graph_run_group_id
      AND error_message IS NOT NULL;

    -- Étape B : Enregistrer le statut global du run dans transformation_pipeline_status
    -- IFF(condition, valeur_si_vrai, valeur_si_faux)
    -- → si n_errors > 0 : statut = 'FAILED'
    -- → sinon           : statut = 'SUCCEEDED'
    INSERT INTO raw.transformation_pipeline_status (graph_run_group_id, started_at, finished_at, status)
    SELECT
        :graph_run_group_id  AS graph_run_group_id,
        :started_at          AS started_at,
        CURRENT_TIMESTAMP()  AS finished_at,
        IFF(:n_errors > 0, 'FAILED', 'SUCCEEDED');

    -- Étape C : Nettoyage conditionnel
    -- Si aucune erreur → on peut vider la table intermédiaire en toute sécurité
    -- Si erreurs → on garde les données pour investigation, et on lève une exception
    IF (n_errors = 0) THEN
        TRUNCATE TABLE raw.data_to_process;  -- ← nettoyage de la table intermédiaire
    ELSE
        RAISE pipeline_exception;            -- ← propage l'erreur → Finalizer marqué FAILED
    END IF;

END;
$$;
```

#### Étape 4 — La task Finalizer

```sql
CREATE OR ALTER TASK raw.finalize_transformation
    WAREHOUSE = COMPUTE_WH
    FINALIZE  = 'raw.identify_new_data_task'  -- ← lié à la root task du DAG
AS
DECLARE
    -- Récupère l'ID du run courant (partagé par toutes les tasks du DAG)
    graph_run_group_id STRING    := SYSTEM$TASK_RUNTIME_INFO('CURRENT_TASK_GRAPH_RUN_GROUP_ID');

    -- Récupère le timestamp de démarrage prévu de la root task
    -- Utile pour calculer la durée totale du run
    started_at         TIMESTAMP := SYSTEM$TASK_RUNTIME_INFO('CURRENT_TASK_GRAPH_ORIGINAL_SCHEDULED_TIMESTAMP');
BEGIN
    CALL raw.finalize_transformation(:graph_run_group_id, :started_at);
END;
```

#### Étape 5 — Activation

```sql
-- Finalizer d'abord, root task EN DERNIER
ALTER TASK raw.finalize_transformation   RESUME;
ALTER TASK raw.identify_new_data_task    RESUME;  -- ← EN DERNIER
```

#### Étape 6 — Données de test + vérification

```sql
-- Nettoyer les tables pour repartir à zéro (test propre)
TRUNCATE TABLE raw.raw_events;
TRUNCATE TABLE staging.step_lsc;

-- Insérer une ligne de test dans raw_events
INSERT INTO raw.raw_events (event_timestamp, process_name, process_id, message)
VALUES ('2018-12-23 22:15:29.606'::TIMESTAMP, 'Step_LSC', 30002312, 'onStandStepChanged 3579');

-- Vérifier l'historique des tasks (dernière heure)
SELECT *
FROM TABLE(INFORMATION_SCHEMA.TASK_HISTORY(
    SCHEDULED_TIME_RANGE_START => DATEADD('hour', -1, current_timestamp())
))
WHERE schema_name = 'RAW';

-- Vérifier les logs d'enrichissement
SELECT *
FROM raw.logging
ORDER BY created_at DESC;
```

---

### Flux complet du Finalizer — visualisé

```
Toutes les tasks enfants ont terminé
              │
              ▼
  raw.finalize_transformation (task)
              │
              ▼
  raw.finalize_transformation (procédure)
              │
              ├─ COUNT erreurs dans raw.logging
              │         │
              │    n_errors = 0 ?
              │         │
              │    ┌────┴────┐
              │   OUI       NON
              │    │         │
              │    ▼         ▼
              │  TRUNCATE   RAISE
              │  data_to_   pipeline_
              │  process    exception
              │    │         │
              └────┴─────────┘
                    │
                    ▼
  raw.transformation_pipeline_status
  ┌─────────────────────────────────────────────────────┐
  │ graph_run_group_id │ started_at │ finished_at │ status │
  │ "abc-123"          │ 10:00:00   │ 10:02:34    │ SUCCEEDED │
  │ "xyz-456"          │ 11:00:00   │ 11:03:01    │ FAILED    │
  └─────────────────────────────────────────────────────┘
```

---

### La fonction `SYSTEM$TASK_RUNTIME_INFO` — les paramètres utiles

| Paramètre | Type retourné | Contient |
|---|---|---|
| `'CURRENT_TASK_GRAPH_RUN_GROUP_ID'` | STRING | ID unique du run complet du DAG |
| `'CURRENT_TASK_GRAPH_ORIGINAL_SCHEDULED_TIMESTAMP'` | TIMESTAMP | Timestamp prévu de démarrage de la root task |
| `'CURRENT_TASK_NAME'` | STRING | Nom complet de la task en cours d'exécution |

> 💡 `CURRENT_TASK_GRAPH_ORIGINAL_SCHEDULED_TIMESTAMP` est particulièrement utile dans le Finalizer pour calculer la **durée totale du run** : `finished_at - started_at`.

---

### Différence `AFTER` vs `FINALIZE`

| `AFTER` | `FINALIZE` |
|---|---|
| S'exécute après UNE task spécifique | S'exécute après TOUTES les tasks du DAG |
| Ne s'exécute pas si la task parente a échoué | S'exécute TOUJOURS (succès ou échec) |
| Plusieurs tasks peuvent avoir le même `AFTER` | Un seul Finalizer par DAG |
| Task enfant normale | Task spéciale de clôture |

### Débugger le Finalizer

```sql
-- Vérifier que le Finalizer s'est exécuté en dernier
SELECT name, state, scheduled_time, completed_time, error_message
FROM TABLE(INFORMATION_SCHEMA.TASK_HISTORY(
    SCHEDULED_TIME_RANGE_START => DATEADD('hour', -1, current_timestamp())
))
WHERE schema_name = 'RAW'
ORDER BY completed_time DESC;
-- Le Finalizer doit apparaître avec le COMPLETED_TIME le plus tardif

-- Vérifier le statut global enregistré par le Finalizer
SELECT *
FROM raw.transformation_pipeline_status
ORDER BY finished_at DESC;
```

---

## 9 · Architecture du projet health_app

### Vue d'ensemble du pipeline complet

```
raw.raw_events (données brutes)
        │
        ▼
[data_quality_task] ← toutes les heures (ou event-driven)
        │  Appelle raw.data_quality()
        │  Insère les anomalies dans raw.data_anomalies
        │
        ├── [hih_listener_manager]        → staging."HiH_ListenerManager"
        ├── [hih_hibroadcastutil]         → staging."HiH_HiBroadcastUtil"
        ├── [step_standstepcounter]       → staging."Step_StandStepCounter"
        ├── [step_sputils]               → staging."Step_SPUtils"
        ├── [step_lsc]                   → staging."Step_LSC"
        ├── [hih_hihealthdatainsertstore] → staging."HiH_HiHealthDataInsertStore"
        ├── [hih_datastatmanager]         → staging."HiH_DataStatManager"
        ├── [hih_hisyncutil]             → staging."HiH_HiSyncUtil"
        ├── [step_standreportreceiver]   → staging."Step_StandReportReceiver"
        └── [step_screenutil]            → staging."Step_ScreenUtil"
                │  (toutes en PARALLÈLE)
                │  Chacune appelle raw.enrich_data()
                │  Chacune logue dans raw.logging
                ▼
        [finalize_transformation] ← s'exécute TOUJOURS en dernier
                │
                ▼
        raw.logging (traçabilité complète)
```

### Les tables du projet

#### `raw.raw_events` — Source brute

```
event_id        : identifiant unique de l'événement
event_timestamp : date/heure de l'événement
process_name    : source applicative (ex: 'HiH_ListenerManager')
process_id      : identifiant du process
message         : message brut (contient log_trigger + message réel concaténés)
```

#### `raw.data_anomalies` — Quarantaine

```sql
CREATE OR ALTER TABLE raw.data_anomalies (
    event_id                INT,
    is_correct_timestamp    BOOLEAN,  -- FALSE si timestamp invalide
    is_correct_process_name BOOLEAN,  -- FALSE si process_name invalide
    created_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP(),
    graph_run_group_id      STRING
);
```

#### `raw.logging` — Traçabilité

```sql
CREATE OR ALTER TABLE raw.logging (
    created_at          TIMESTAMP DEFAULT CURRENT_TIMESTAMP(),
    graph_run_group_id  STRING,
    table_name          STRING,
    n_rows              NUMBER,
    error_message       STRING DEFAULT NULL
);
```

### Le LEFT JOIN anti-anomalie

```sql
-- Technique pour exclure les lignes marquées comme anomalies
-- sans les supprimer de la table source

SELECT re.*
FROM raw.raw_events re
LEFT JOIN raw.data_anomalies da ON re.event_id = da.event_id
WHERE da.event_id IS NULL;  -- ← garde UNIQUEMENT les lignes sans entrée dans data_anomalies

-- Visualisation :
-- raw_events          data_anomalies
-- event_id=1    ←──── event_id=1 (anomalie !)
-- event_id=2           (pas de correspondance)
-- event_id=3    ←──── event_id=3 (anomalie !)
--
-- Résultat : uniquement event_id=2
```

### Les UDFs requises (doivent exister préalablement)

| UDF | Rôle | Signature |
|---|---|---|
| `raw.check_correct_timestamp()` | Valide le format/plage du timestamp | `(TIMESTAMP) → BOOLEAN` |
| `raw.check_correct_process_name()` | Valide que le process_name est connu | `(STRING) → BOOLEAN` |
| `raw.extract_log_trigger()` | Extrait le type d'événement du message brut | `(STRING) → STRING` |
| `raw.extract_log_message()` | Nettoie et extrait le message réel | `(STRING) → STRING` |

---

## 10 · Glossaire complet

| Terme | Définition |
|---|---|
| **Task** | Tâche SQL automatisée dans Snowflake (planificateur) |
| **DAG** | Directed Acyclic Graph — graphe de tâches ordonnées sans cycle |
| **Root Task** | Tâche principale du DAG (a un `SCHEDULE`, démarre tout) |
| **Child Task** | Tâche enfant déclenchée après une autre (a un `AFTER`) |
| **Finalizer Task** | Tâche spéciale qui s'exécute après TOUTES les tasks du DAG |
| **Warehouse** | Moteur de calcul Snowflake (CPU/RAM), consomme des crédits |
| **SCHEDULE** | Fréquence de déclenchement automatique |
| **AFTER** | Dépendance — s'exécute après la task citée |
| **FINALIZE** | Paramètre désignant une task comme finaliseur du DAG |
| **WHEN** | Condition supplémentaire pour s'exécuter |
| **SUSPEND / RESUME** | Désactiver / activer une task |
| **RETRY LAST** | Relancer uniquement les tasks FAILED du dernier run |
| **Stream** | Objet qui capture les changements (CDC) sur une table |
| **Append-Only Stream** | Stream qui capture uniquement les INSERT (plus léger) |
| **SYSTEM$STREAMHASDATA** | Fonction qui indique si un stream a des données non consommées |
| **SYSTEM$TASK_RUNTIME_INFO** | Fonction exposant les infos du run en cours |
| **graph_run_group_id** | ID unique d'un run complet du DAG — partagé par toutes les tasks |
| **Procédure stockée** | Bloc de code SQL nommé et réutilisable (appelé avec `CALL`) |
| **UDF** | User Defined Function — fonction personnalisée |
| **IDENTIFIER()** | Permet d'utiliser une variable STRING comme nom de table |
| **SQLROWCOUNT** | Nb de lignes affectées par le dernier INSERT/UPDATE/DELETE |
| **SQLERRM** | Message texte de la dernière erreur (disponible dans EXCEPTION) |
| **SQLCODE** | Code numérique de la dernière erreur |
| **WHEN OTHER THEN** | Intercepte toutes les exceptions non gérées |
| **RAISE** | Re-propage une exception vers l'appelant |
| **EXECUTE AS CALLER** | La procédure s'exécute avec les droits de l'appelant |
| **DML** | Data Manipulation Language — INSERT, UPDATE, DELETE, MERGE |
| **DDL** | Data Definition Language — CREATE, ALTER, DROP |
| **CDC** | Change Data Capture — mécanisme de capture des changements |

---

## 11 · Checklist de déploiement

### Prérequis

```
[ ] Les UDFs existent dans le schéma raw :
    [ ] raw.check_correct_timestamp()
    [ ] raw.check_correct_process_name()
    [ ] raw.extract_log_trigger()
    [ ] raw.extract_log_message()

[ ] Les tables staging existent pour chaque process

[ ] Le warehouse COMPUTE_WH existe et est actif

[ ] Le rôle a les droits :
    [ ] CREATE TASK sur le schéma raw
    [ ] INSERT sur les tables staging
    [ ] EXECUTE sur les procédures
    [ ] CREATE STREAM (si architecture event-driven)
```

### Ordre d'exécution du script

```
[ ] 1.  USE ROLE / DATABASE / SCHEMA
[ ] 2.  CREATE TABLE raw.data_anomalies
[ ] 3.  CREATE TABLE raw.logging
[ ] 4.  CREATE PROCEDURE raw.log_results()
[ ] 5.  CREATE PROCEDURE raw.data_quality()
[ ] 6.  CREATE PROCEDURE raw.enrich_data()
[ ] 7.  (optionnel) CREATE STREAM raw.raw_events_stream
[ ] 8.  CREATE TASK raw.data_quality_task (root)
[ ] 9.  CREATE TASK raw.[toutes les tasks enfants]
[ ] 10. CREATE TASK raw.finalize_transformation (finalizer)
[ ] 11. ALTER TASK [toutes les enfants]        RESUME
[ ] 12. ALTER TASK raw.finalize_transformation RESUME
[ ] 13. ALTER TASK raw.data_quality_task       RESUME  ← EN DERNIER
[ ] 14. EXECUTE TASK raw.data_quality_task             ← test manuel
[ ] 15. Attendre 2-3 minutes, vérifier TASK_HISTORY
[ ] 16. Vérifier raw.logging pour les résultats
```

---

## 12 · Patterns réutilisables

### Pattern 1 — DAG complet avec logging et exceptions

```sql
-- Root task
CREATE OR ALTER TASK mon_schema.root_task
    WAREHOUSE = COMPUTE_WH
    SCHEDULE = '1 HOURS'
AS
DECLARE
    run_id STRING := SYSTEM$TASK_RUNTIME_INFO('CURRENT_TASK_GRAPH_RUN_GROUP_ID');
BEGIN
    CALL mon_schema.ma_procedure_principale(:run_id);
END;

-- Child task (pattern identique pour chaque enfant)
CREATE OR ALTER TASK mon_schema.child_task
    WAREHOUSE = COMPUTE_WH
    AFTER mon_schema.root_task
AS
DECLARE
    run_id STRING := SYSTEM$TASK_RUNTIME_INFO('CURRENT_TASK_GRAPH_RUN_GROUP_ID');
BEGIN
    CALL mon_schema.ma_procedure_secondaire(:run_id);
END;

-- Finalizer
CREATE OR ALTER TASK mon_schema.finalizer_task
    WAREHOUSE = COMPUTE_WH
    FINALIZE = 'mon_schema.root_task'
AS
    CALL mon_schema.ma_procedure_finale();

-- Activation (ordre obligatoire)
ALTER TASK mon_schema.child_task     RESUME;
ALTER TASK mon_schema.finalizer_task RESUME;
ALTER TASK mon_schema.root_task      RESUME;  -- EN DERNIER
```

### Pattern 2 — Task event-driven avec Stream

```sql
-- Créer le stream
CREATE OR REPLACE STREAM mon_schema.ma_table_stream
    ON TABLE mon_schema.ma_table
    APPEND_ONLY = TRUE;

-- Task déclenchée uniquement si le stream a des données
CREATE OR ALTER TASK mon_schema.ma_task_reactive
    WAREHOUSE = COMPUTE_WH
    SCHEDULE = '1 MINUTES'
    WHEN SYSTEM$STREAMHASDATA('mon_schema.ma_table_stream')
AS
DECLARE
    run_id STRING := SYSTEM$TASK_RUNTIME_INFO('CURRENT_TASK_GRAPH_RUN_GROUP_ID');
BEGIN
    CALL mon_schema.traiter_nouvelles_donnees(:run_id);
END;
```

### Pattern 3 — Procédure avec logging et gestion d'erreur

```sql
CREATE OR REPLACE PROCEDURE mon_schema.traiter_table(
    table_name STRING,
    run_id     STRING
)
RETURNS STRING
LANGUAGE SQL
EXECUTE AS CALLER
AS $$
DECLARE
    full_table  STRING    := CONCAT('mon_staging.', :table_name);
    mon_erreur  EXCEPTION (-9999, 'Erreur lors du traitement');
BEGIN
    Let n INT := 0;

    INSERT INTO IDENTIFIER(:full_table) (col1, col2)
    SELECT col1, col2
    FROM mon_schema.ma_source
    WHERE condition = :table_name;

    n := SQLROWCOUNT;

    -- Log succès
    INSERT INTO mon_schema.logs (run_id, table_name, n_rows, error_msg)
    VALUES (:run_id, :table_name, :n, NULL);

    RETURN :n;

EXCEPTION
    WHEN OTHER THEN
        -- Log échec + re-propagation
        INSERT INTO mon_schema.logs (run_id, table_name, n_rows, error_msg)
        VALUES (:run_id, :table_name, NULL, :SQLERRM);
        RAISE mon_erreur;

END;
$$;
```

### Pattern 4 — Requêtes de monitoring standard

```sql
-- Dashboard d'un run
SELECT
    table_name,
    n_rows,
    CASE WHEN error_message IS NULL THEN '✅ Succès' ELSE '❌ Échec' END AS statut,
    error_message
FROM raw.logging
WHERE graph_run_group_id = 'TON_RUN_ID'
ORDER BY created_at;

-- Tasks en échec des dernières 24h
SELECT name, state, error_message, scheduled_time
FROM TABLE(INFORMATION_SCHEMA.TASK_HISTORY(
    SCHEDULED_TIME_RANGE_START => DATEADD('hour', -24, current_timestamp())
))
WHERE state = 'FAILED'
  AND schema_name = 'RAW'
ORDER BY scheduled_time DESC;

-- Résumé par run (nb succès, nb échecs)
SELECT
    graph_run_group_id,
    COUNT(*) AS total_tasks,
    SUM(CASE WHEN error_message IS NULL THEN 1 ELSE 0 END) AS succes,
    SUM(CASE WHEN error_message IS NOT NULL THEN 1 ELSE 0 END) AS echecs
FROM raw.logging
GROUP BY graph_run_group_id
ORDER BY MIN(created_at) DESC;
```

---

## 13 · Conclusion du chapitre — Architecture complète

### Le schéma final

Ce diagramme résume l'architecture type d'un pipeline Snowflake bien construit. Chaque bloc représente une **unité indépendante** composée de 3 niveaux :

```
┌─────────────────────────────────┐
│            Task                 │  ← Planificateur (SCHEDULE ou AFTER)
│              │                  │
│              ▼                  │
│          Procédure              │  ← Orchestrateur (logique principale)
│           /     \               │
│     Fonction   Procédure        │  ← Briques réutilisables (UDFs + sous-procédures)
└─────────────────────────────────┘
```

### Le pipeline global — lecture du diagramme

```
                                         ┌── [Task → Proc → Fn + Proc]
                                         │
[Task → Proc → Fn + Proc] ──────────► [Task → Proc → Fn + Proc] ──► [Task → Proc → Fn + Proc]
        │                                                             │
        │ Finalize                                                    └── [Task → Proc → Fn + Proc]
        ▼
[Task → Proc → Fn + Proc]    ← Finalizer (s'exécute toujours en dernier)
```

**Lecture :**
- Les **flèches horizontales jaunes** = dépendances entre DAGs ou entre tasks (`AFTER`)
- La **flèche `Finalize`** = le Finalizer, déclenché après tout le DAG principal
- Chaque **bloc gris** = une unité autonome Task + Procédure + Fonctions/Sous-procédures

---

### Les 5 principes à retenir

**1. Toujours découpler Task et logique**
> La Task ne fait que déclencher. La logique métier vit dans la procédure. On peut tester la procédure indépendamment avec `CALL`.

**2. La hiérarchie dans chaque bloc**
> `Task → Procédure → Fonctions/Procédures` — chaque niveau a un rôle précis. Les UDFs (Fonctions) font des transformations atomiques, les Procédures orchestrent, les Tasks planifient.

**3. Le Finalizer pour clore proprement**
> Un pipeline sans Finalizer ne sait pas quand il est vraiment "terminé". Le Finalizer centralise le nettoyage, les notifications, et la traçabilité de fin de run.

**4. Le logging à chaque niveau**
> `graph_run_group_id` relie toutes les logs d'un même run. On peut toujours répondre à "qu'est-ce qui s'est passé lors du run de 10h ?" avec un simple filtre.

**5. Gérer les erreurs sans bloquer le pipeline**
> `WHEN OTHER THEN` + `RAISE` = on loggue l'erreur ET on la propage. La task est marquée `FAILED` dans `TASK_HISTORY`, les autres tasks continuent, et on a la trace de ce qui a planté.

---

### Récapitulatif des commandes essentielles

```sql
-- Créer / modifier
CREATE OR ALTER TASK ...
CREATE OR REPLACE PROCEDURE ...
CREATE OR REPLACE STREAM ...

-- Cycle de vie
ALTER TASK mon_schema.ma_task SUSPEND;   -- avant modification
ALTER TASK mon_schema.ma_task RESUME;    -- après configuration

-- Déclencher
EXECUTE TASK mon_schema.root_task;            -- run complet
EXECUTE TASK mon_schema.root_task RETRY LAST; -- relancer les FAILED uniquement

-- Vérifier
SHOW TASKS IN SCHEMA mon_schema;
SELECT * FROM TABLE(INFORMATION_SCHEMA.TASK_HISTORY(...));
SELECT SYSTEM$STREAMHASDATA('mon_schema.mon_stream');

-- Variables système dans les procédures
SYSTEM$TASK_RUNTIME_INFO('CURRENT_TASK_GRAPH_RUN_GROUP_ID')  -- ID du run
SQLROWCOUNT   -- nb de lignes du dernier DML
SQLERRM       -- message de la dernière erreur (dans EXCEPTION)
```

---

*Documentation générée le 2026-02-23 · Cours Snowflake Tasks · Projet `health_app`*
