# Recherche #7 — Recherche plein texte, Postgres et fractional indexing

Ticket : m-renou/outline#7 (enfant de #1). Spec : `docs/evolution/02-specifications-databases-outline.md` RG-36 (l. 288), RG-45 (l. 314), §5.1 (l. 373).
Code de référence : commit `0545db908` du fork. Faits uniquement ; les options sont listées sans être tranchées.

## 1. Architecture de la recherche

### 1.1 Abstraction `SearchProvider`

- Contrat abstrait `BaseSearchProvider` : `server/utils/BaseSearchProvider.ts:55-145`. Méthodes : `searchForUser`, `searchForTeam` (partages), `searchTitlesForUser`, `searchCollectionsForUser`, `index`, `remove`, `updateMetadata`.
- `SearchOptions` (`server/utils/BaseSearchProvider.ts:24-49`) transporte un `filter?: Filter` (arbre d'expressions `@shared/helpers/FilterHelper`). Le commentaire l. 31-35 précise que l'autorisation des champs porteurs de droits (ex. `collectionId`) est faite **avant** d'atteindre le provider.
- `SearchResponse` (`server/utils/BaseSearchProvider.ts:11-22`) : `{ ranking, context?, document }[]` + `total`. Pas de champ annexe pour d'autres métadonnées (ex. database).
- Sélection : `SearchProviderManager.getProvider()` (`server/utils/SearchProviderManager.ts:20-41`) choisit le plugin `Hook.SearchProvider` dont l'`id` vaut `env.SEARCH_PROVIDER` ; le résultat est mis en cache.
- Plugin : `plugins/search-postgres/server/index.ts:1-13` enregistre `PostgresSearchProvider` (id `"postgres"`, `plugins/search-postgres/server/PostgresSearchProvider.ts:71`).

### 1.2 Appelants du provider

- `documents.search` : `server/routes/api/documents/documents.ts:1244-1400` → `searchForTeam` (l. 1330, partage public) ou `searchForUser` (l. 1357). Filtre construit via `combineFilters(rawFilters) ?? legacyParamsToFilter(...)` (l. 1261-1269), autorisé par `authorizeFilterFields` (l. 1349 ; impl. `server/models/helpers/Filters.ts:460-481`, ne vérifie que `collectionId`), puis `expandDocumentIdLeaves`.
- `documents.search_titles` : `server/routes/api/documents/documents.ts:1194-1233` → `searchTitlesForUser`.
- Suggestions de mention / liens : `server/routes/api/suggestions/suggestions.ts:80` (`searchTitlesForUser`), `:110` (collections).
- MCP : `server/mcp/tools/documents.ts:140`, `:263`. Slack : `plugins/slack/server/api/hooks.ts:256`.
- Indexation : `server/queues/processors/SearchIndexProcessor.ts:22-61` appelle `provider.index(...)` sur `documents.publish`, `documents.update.delayed`, `documents.unarchive`, etc. Pour Postgres ce sont des no-op (`PostgresSearchProvider.ts:426-461`) : l'indexation est faite par trigger.

### 1.3 Implémentation Postgres

- Indexation par trigger `documents_search_trigger()` ; version courante : `server/migrations/20260921120000-tolerate-tsvector-limit-in-document-search-trigger.js:7-24`. Poids : `title` → A, `previousTitles` → C, `text` (1 000 000 premiers caractères, 500 000 en repli) → D. Config `'english'` en dur. Le trigger ne lit que des colonnes de la ligne `documents` (pas de jointure).
- Index GIN : `documents_tsv_idx` (`server/migrations/baseline/schema.sql:1365`) et `documents_team_id_tsv_idx` sur `("teamId", "searchVector")` avec `btree_gin` (`server/migrations/20260803143858-add-documents-team-search-vector-index.js:7-10`).
- Clause de base commune à toutes les recherches : `buildWhere` (`PostgresSearchProvider.ts:754-835`) : `teamId`, `deletedAt IS NULL`, `template: false`, `sourceMetadata.trial IS NULL` (l. 756-772), puis visibilité (adhésions, collections accessibles, brouillons propres, l. 794-828), puis `buildSearchWhere(filter)` (l. 830-832).
- Classement : `ts_rank("searchVector", to_tsquery('english', :query))`, multiplié par un boost de popularité (l. 648-655). Requête sur réplique en lecture (l. 578-599).
- Correspondance : `to_tsquery('english', …)` + `ILIKE` pour les segments entre guillemets, URLs et identifiants (l. 843-942). Repli automatique en recherche entre guillemets si 0 résultat (l. 469-513).
- `searchTitlesForUser` : `title ILIKE` + même `buildWhere` (l. 311-337). `searchCollectionsForUser` : `unaccent(LOWER(name)) like unaccent(LOWER(:query))` (l. 352-354).
- Extrait (`context`) : calculé en JS à partir de `DocumentHelper.toPlainText(document)` (l. 684-732).

### 1.4 Champs de filtre exposés

- Allowlist côté API : `documentFilterFields` (`server/routes/api/documents/schema.ts:19-33`) : `createdAt`, `updatedAt`, `publishedAt`, `archivedAt`, `title`, `templateId`, `collectionId`, `createdById`, `userId`, `documentId`, `parentDocumentId`. Le schéma de recherche les réutilise (`schema.ts:44-47`). Aucun champ lié aux databases.
- Front : `app/scenes/Search/Search.tsx:103-163` construit l'arbre `filters` (collection, user, document, date, statut). Composants de filtre : `app/scenes/Search/components/` (`CollectionFilter`, `DateFilter`, `DocumentFilter`, `DocumentTypeFilter`, `UserFilter`). `DocumentTypeFilter` propose Published / Archived / Drafts (l. 15-27).
- Affichage d'un résultat : `DocumentListItem` avec `context` et `showCollection` (`app/scenes/Search/Search.tsx` ~l. 440-441) → `DocumentMeta` affiche la collection via `collections.get(document.collectionId)` (`app/components/DocumentMeta.tsx:86-87`, `:227`). Type front `SearchResult` : `app/types.ts:251-256`.

### 1.5 Précédents d'exclusion d'une catégorie de documents

- `template: false` et `sourceMetadata.trial IS NULL` sont codés en dur dans `buildWhere` (`PostgresSearchProvider.ts:764-769`) et dans `publishedWhere` du modèle (`server/models/Document.ts:136-148`).

### 1.6 Options pour RG-45 (non tranchées)

Inclusion par défaut / filtre « Inclure les lignes de bases » :

- A. Ajouter `databaseId` (colonne prévue §5.1 « Patch de la table documents ») à `documentFilterFields` et laisser le front émettre `{ field: "databaseId", operator: "isNull" }` quand la case est décochée. Passe par le mécanisme `Filter` générique existant, sans toucher au provider.
- B. Ajouter une option booléenne dédiée dans `SearchOptions` (ex. `includeDatabaseRows`) traitée dans `buildWhere`, sur le modèle de `template: false`. Modifie le contrat `BaseSearchProvider` (tous les providers).
- C. Ne rien filtrer côté serveur ; filtrer côté client (incompatible avec la pagination et `total`).

Mention de la database dans un résultat :

- D. Exposer `databaseId` dans `presentDocument` et résoudre le titre côté front via un store `databases` (même schéma que `collections.get` dans `DocumentMeta`).
- E. Inclure la database (titre/icône) dans la réponse `documents.search` en élargissant `SearchResponse` ou en post-traitant dans la route (l. 1371-1379).
- F. Pour les lignes de database inline, la database est rattachée à un document hôte : afficher le titre du document hôte via `parentDocumentId` (déjà supporté par `showParentDocuments` de `DocumentMeta`) — dépend du modèle de parenté retenu pour les lignes.

Indexation des lignes (RG-45 MVP, RG-46 L2) :

- Les lignes étant des `documents`, le trigger existant indexe déjà titre + corps sans changement.
- RG-46 (valeurs de propriétés, poids C) : le trigger ne voit que la ligne `documents`. Options : (i) étendre `documents_search_trigger` avec une sous-requête sur `database_rows.values` (et un trigger sur `database_rows` qui « touche » le document) ; (ii) dénormaliser un texte des valeurs dans une colonne de `documents` alimentée par l'application ; (iii) colonne `searchVector` propre à `database_rows` et `UNION`/jointure dans le provider.
- Remarque : `searchTitlesForUser` (mentions, liens, `documents.search_titles`) applique le même `buildWhere` ; toute exclusion ajoutée dans `buildWhere` y est aussi appliquée.

## 2. `unaccent` et ICU

### 2.1 Dans le code / les migrations

- `unaccent` est **créée** par le schéma de base : `server/migrations/baseline/schema.sql:8-11` (`CREATE EXTENSION IF NOT EXISTS unaccent`) ; migration historique listée : `server/migrations/baseline/migrations.json:214` (`20240912222438-add-unaccent-extension.js`). Également créées : `pg_trgm` (l. 2-5), `uuid-ossp` (l. 14-17), `btree_gin` (migration 20260803143858, l. 7).
- `unaccent` est déjà **utilisée** en production de code : `PostgresSearchProvider.ts:353`, `server/routes/api/documents/documents.ts:900`, `server/routes/api/suggestions/suggestions.ts:54,57,100`, ainsi que `templates.ts`, `users.ts`, `apiKeys.ts`, `server/mcp/tools/users.ts`, `plugins/webhooks/server/api/webhookSubscriptions.ts` (motif `unaccent(LOWER(col)) like unaccent(LOWER(:query))`).
- Le baseline est appliqué par `server/migrations/20260104155140-baseline-schema.js:51-57` sur une base vide (donc toute nouvelle installation a `unaccent`).
- **ICU** : aucune collation ICU ni `CREATE COLLATION` dans `server/migrations` ni `server/models`. Seule collation explicite utilisée : `collate "C"` pour trier les index fractionnels (ex. `server/models/Collection.ts:477-478`, `server/routes/api/pins/pins.ts:125`, `server/routes/api/stars/stars.ts:79`, `server/utils/removeIndexCollision.ts:34-39`).
- La recherche plein texte utilise `'english'` partout ; aucune config `unaccent` dans le dictionnaire tsvector.

### 2.2 Infra

- `docker-compose.yml:7-15` : image `postgres` **sans tag** (donc `latest`), pas de script d'init d'extension.
- Skill de vérification : `.claude/skills/verify-outline/scripts/verify-outline.sh:21` : `postgres:latest` par défaut.
- CI : `.github/workflows/ci.yml:123-130` utilise le PostgreSQL préinstallé du runner GitHub, puis `yarn sequelize db:migrate` (l. 132) ; la création d'`unaccent` passe donc en CI.
- `app.json:20,37` : add-on `heroku-postgresql` (déploiement Heroku). Pas de version minimale de Postgres déclarée dans le repo.
- L'environnement de production du fork n'est pas décrit dans le repo : **non vérifiable ici**.

### 2.3 Vérification empirique (image locale `postgres:16-alpine`, PostgreSQL 16.15)

| Test | Résultat |
|---|---|
| `count(*) from pg_collation where collprovider='i'` | 908 collations ICU (`und-x-icu`, `fr-x-icu` présentes) |
| `CREATE EXTENSION unaccent; select unaccent('Éléphant Ça')` | `Elephant Ca` |
| Volatilité de `unaccent(text)` | `s` (STABLE) → non utilisable tel quel dans un index d'expression (exige IMMUTABLE) |
| `CREATE COLLATION ci_ai (provider=icu, locale='und-u-ks-level1', deterministic=false)` puis `'élephant' = 'ELEPHANT' collate ci_ai` | `t` |
| `'a' like 'A' collate ci_ai` | `ERROR: nondeterministic collations are not supported for LIKE` |
| `to_tsvector('english','Éléphant')` | `'éléphant'` (accents conservés par la config `english`) |
| `unaccent('Éléphant') ilike unaccent('%élep%')` vs `'Éléphant' ilike '%elep%'` | `t` vs `f` |

### 2.4 Options pour RG-36 (non tranchées)

- A. `unaccent(lower(x)) LIKE unaccent(lower(:q))` — motif déjà en place dans le code ; extension garantie par le baseline. Pour l'indexer : wrapper IMMUTABLE (`unaccent('unaccent', …)` encapsulé) + index d'expression / `pg_trgm`.
- B. Collation ICU non déterministe (`und-u-ks-level1`) — disponible dans l'image testée, mais `LIKE`/`ILIKE` refusé en PG16 (vérifié) ; utilisable pour `=` et `ORDER BY`. La prise en charge de `LIKE` avec collations non déterministes est annoncée pour PostgreSQL 18 (notes de version PG18, à confirmer contre la version de prod).
- C. Normaliser en JS (NFD + suppression des diacritiques) au moment d'écrire les valeurs dans `database_rows.values` / une colonne dérivée, et comparer en `ILIKE` simple.
- Point ouvert d'infra : version Postgres et disponibilité ICU en prod (non décrite dans le repo).

## 3. Fractional indexing

- Dépendance existante : `fractional-index` `^1.0.0` (`package.json:144`), résolue en `1.0.0` (`yarn.lock:10464-10466`). Import par défaut `fractionalIndex(a, b)`.
- Usages serveur : `server/utils/indexing.ts:1-97` (`collectionIndexing`, `starIndexing` : remplissent les index manquants), `server/utils/removeIndexCollision.ts:1-49` (résolution de collision pour collections), `server/commands/pinCreator.ts:1,48-57`, `server/commands/starCreator.ts:1`, `server/models/Collection.ts:1`, `server/routes/api/documents/documents.ts:2`, `server/queues/processors/ImportsProcessor.ts:1`, script `server/scripts/20250327062414-resolve-collection-index-collisions.ts`.
- Usages front : `app/components/Sidebar/hooks/useDragAndDrop.tsx:1`, `Collections.tsx`, `DraggableCollectionLink.tsx`, `StarredLink.tsx`, `SharedWithMe*.tsx`, `PinnedDocuments.tsx`.
- Store générique : `app/stores/base/IndexedStore.ts:6-19` (tri par `index` en comparaison de chaînes).
- Colonnes : `index character varying(255)` (ex. `server/migrations/baseline/schema.sql:251,601,693`) et `index text` (l. 291). La spec propose `varchar(256)`.
- Tri SQL : systématiquement `"x"."index" collate "C"` pour un ordre octet (commentaire `server/models/Collection.ts:477`).
- Il n'existe pas d'utilitaire générique « collision » : `removeIndexCollision` est spécifique à `Collection`.

Options pour `database_rows.index` / `database_properties.index` / `database_views.index` :

- A. Réutiliser `fractional-index` + tri `collate "C"` + un helper de collision à généraliser depuis `removeIndexCollision`.
- B. Ajouter une autre bibliothèque (ex. `fractional-indexing` avec `generateNKeysBetween` pour insertions en lot) — nouvelle dépendance.
