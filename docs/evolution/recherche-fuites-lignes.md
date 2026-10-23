# Recherche — Inventaire des fuites possibles des documents-lignes dans le core

Ticket : m-renou/outline#3 (enfant de #1). Spec de référence : `docs/evolution/02-specifications-databases-outline.md` (RG-20, RG-21, RG-24, RG-51, §5 « Patch de la table documents », §11.4, §12, §16.2).

Source : code du fork au commit `0545db908` (branche `main`). Toutes les références `fichier:ligne` pointent sur ce commit. **Faits, pas décisions** : chaque entrée liste les options de patch, sans trancher.

## Prémisses

- La colonne `documents.databaseId` **n'existe pas encore** dans le fork (`grep databaseId` ne trouve que des occurrences sans rapport dans `server/utils/oauth/` et `plugins/notion/`). Tous les patchs ci-dessous supposent la colonne de la spec §5 (« Renseigné uniquement pour les documents-lignes »).
- Selon RG-20, une ligne est un document **publié**, dans la collection de la database, avec `parentDocumentId` = document hôte. Conséquence structurelle : **tout code qui parcourt les enfants via `parentDocumentId`** (et non via `documentStructure`) verra les lignes comme des enfants ordinaires de l'hôte.
- `documentStructure` est une colonne JSONB de `collections` (`server/models/Collection.ts:275`), mise en cache Redis (`Collection.ts:390-415`, `getCachedDocumentStructure` `Collection.ts:892`). Elle n'est jamais reconstruite en masse : elle est modifiée incrémentalement par les méthodes listées en §1.

---

## 1. Écrivains de `documentStructure`

### 1.1 Primitives (`server/models/Collection.ts`)

| fichier:ligne | Rôle | Risque « ligne » | Patchs envisageables |
|---|---|---|---|
| `Collection.ts:1176` `addDocumentToStructure` | Point d'entrée unique d'insertion. Construit le nœud via `document.toNavigationNode()` (sauf `options.documentJson`), l'insère à la racine ou sous `parentDocumentId`. | Insère une ligne si on l'appelle avec une ligne ; insère les lignes d'un hôte via `toNavigationNode` (cf. 1.2). | (a) garde en tête : `if (document.databaseId) return this;` ; (b) laisser la primitive neutre et filtrer chez les appelants ; (c) combiner (a) + filtre dans `toNavigationNode`. |
| `Collection.ts:1054` `removeDocumentInStructure` | Retire un nœud (et renvoie `[node, index]`, réutilisé par le déplacement). | Aucun (retirer un absent est un no-op), mais **sauvegarde quand même** la collection (`changed(...,true)` + `save`). | Garde précoce si `document.databaseId` pour éviter un `save` inutile ; sinon rien. |
| `Collection.ts:1144` `updateDocument` | Met à jour titre/icône/couleur d'un nœud. | Pas de fuite (ne crée pas de nœud), **mais** appelé à chaque changement de titre d'un document publié (cf. hook 1.2) : verrou `NO_KEY_UPDATE` sur la collection + `save` + invalidation/écriture du cache Redis **même si le nœud est absent**. Pour une ligne, chaque édition de titre depuis la grille déclenche ce coût. | Garde `databaseId` dans le hook appelant (`Document.ts:516`) ou dans `updateDocument` ; ou ne sauvegarder que si un nœud a été trouvé. |
| `Collection.ts:1025` `deleteDocument` | Retire de la structure puis détruit le document et **tous ses descendants par `parentDocumentId`** (`findAllChildDocumentIds`). | Supprimer l'hôte supprime les lignes (conforme RG-04), une à une avec événements → chaque ligne arrive dans la corbeille (cf. §2.3 `documents.deleted`). | Conserver (cascade voulue) ; ou marquer les lignes supprimées par cascade pour les filtrer de la corbeille ; ou cascade dédiée côté plugin. |
| `Collection.ts:957` `archiveWithCtx` | Archive la collection et met `archivedAt` sur **tous** ses documents (`Document.update where collectionId`). | Les lignes sont archivées aussi (cohérent) et apparaissent dans `documents.archived` (cf. §2.3). | Idem `documents.archived`. |
| `Collection.ts:390-415` hooks `BeforeSave`/`AfterSave` | Invalident/écrivent le cache Redis quand `documentStructure` change. | Aucun en soi. | — |

### 1.2 Appelants côté `Document` (`server/models/Document.ts`)

| fichier:ligne | Rôle | Risque « ligne » | Patchs envisageables |
|---|---|---|---|
| `Document.ts:549` hook `@AfterCreate addDocumentToCollectionStructure` | Tout document créé **publié** avec `collectionId` est ajouté à l'index 0. | **Fuite directe n°1** : une ligne étant publiée dès sa création (spec §12), elle entre dans la structure. | Ajouter `model.databaseId` à la garde `if (model.archivedAt \|\| !model.publishedAt ...)` ; ou garde dans `addDocumentToStructure`. |
| `Document.ts:516` hook `updateCollectionStructure` | Sur changement titre/icône/couleur d'un document publié, appelle `collection.updateDocument`. | Pas de fuite, coût (verrou + save + cache) à chaque édition de titre de ligne (cf. `updateDocument`). | Ajouter `model.databaseId` à la garde. |
| `Document.ts:1282` `publish` (insertion `:1337`) | Publication : `addDocumentToStructure(this, index)`. | Fuite si une ligne passe par `publish` (création via `documentCreator` avec `publish: true`, `server/commands/documentCreator.ts:265-270`) ; fuite des lignes de l'hôte si l'hôte est publié après ses lignes (via `toNavigationNode`). | Garde `databaseId` ; filtre dans `toNavigationNode`. |
| `Document.ts:1592` `toNavigationNode` (requête `:1606-1630`) | Construit récursivement le sous-arbre d'un document **depuis la base**, par `parentDocumentId`, `publishedAt != null`, (`archivedAt` null sauf `includeArchived`). | **Fuite indirecte n°2** : chaque fois qu'un hôte est (ré)inséré sans `documentJson` (publication, restauration, déplacement quand le nœud n'était pas dans l'arbre source), **toutes ses lignes publiées sont injectées comme enfants**. | Ajouter `databaseId: null` (`{ [Op.is]: null }`) dans le `where` ; ou paramètre `includeDatabaseRows`. |
| `Document.ts:1395` `unpublishWithCtx` (contrôle `:1404-1412`) | Refuse de dépublier si `findAllChildDocumentIds` non vide, puis retire de la structure. | Pas de fuite, mais **un hôte ayant des lignes ne peut plus être dépublié** (« Cannot unpublish document with child documents »). | Exclure `databaseId IS NOT NULL` du comptage ; ou décider que l'hôte reste non dépubliable. |
| `Document.ts:1460` `archiveWithCtx` → `:1710` `archiveWithChildren` | Retire de la structure puis archive récursivement les enfants par `parentDocumentId`. | Les lignes sont archivées avec l'hôte (cohérent RG-04-like) → listées individuellement dans l'archive (§2.3). | Conserver ; filtrer les listes ; ou cascade sans exposition. |
| `Document.ts:1482` `restoreTo` (insertion `:1510`, `includeArchived: true`) puis `:1672` `restoreArchivedWithChildren` | Réinsère le document via `addDocumentToStructure` (donc `toNavigationNode`, archives incluses) puis désarchive les enfants par `parentDocumentId`. | **Fuite n°2** à la restauration d'un hôte archivé : ses lignes (archivées) sont injectées dans l'arbre. Restaurer une ligne seule (RG-24) passe aussi par `addDocumentToStructure` → **fuite n°1**. | Garde `databaseId` + filtre `toNavigationNode`. |
| `Document.ts:1536` `destroyWithCtx` | Délègue à `collection.deleteDocument`. | Cf. `deleteDocument`. | — |

### 1.3 Commandes et tâches

| fichier:ligne | Rôle | Risque « ligne » | Patchs envisageables |
|---|---|---|---|
| `server/commands/documentMover.ts:65` / `:74` / `:98` | Retire de l'arbre source, réutilise le nœud retiré (`documentJson`) ou le reconstruit via `toNavigationNode`, puis l'insère dans la destination. | Déplacer une ligne seule → fuite n°1 (la spec interdit ce cas au MVP, §12). Déplacer un hôte : si l'arbre source était propre, le nœud réutilisé est propre ; s'il faut reconstruire (`:74`), fuite n°2. | Garde/refus pour `databaseId` ; filtre `toNavigationNode`. |
| `server/commands/documentMover.ts:119-155` | Si changement de collection : met à jour `collectionId` de tous les descendants (`findAllChildDocumentIds`) ; si destination = brouillons (`collectionId` null), **reparente les enfants vers le parent de l'hôte** (`:150`) et dépublie l'hôte (`:145`). | Lignes déplacées avec l'hôte (conforme RG-07 pour `collectionId`). Hôte → brouillons : les lignes deviennent enfants du grand-parent, **hors de leur hôte**, toujours avec `databaseId`. | Interdire la mise en brouillon d'un hôte ; ou traiter les lignes à part (exclure de la reparentalité). |
| `server/commands/documentRestorer.ts:64` | Retire de l'arbre source si la collection change, puis `restoreTo`. | Cf. `restoreTo`. | — |
| `server/commands/documentDuplicator.ts:164-190` (`buildChildItems`) | Duplication récursive : enfants via `findChildDocuments` (par `parentDocumentId`), triés selon `getDocumentTree(...).children`, puis recréés et **publiés** (`publish`). | **Fuite n°3** : dupliquer un hôte (récursif) duplique ses lignes **comme documents ordinaires** (sans `databaseId`, sans `database_rows`) qui sont alors insérés dans la structure. | Exclure `databaseId` dans `findChildDocuments` ici ; ou dupliquer les lignes via le plugin (RG-06, L2). |
| `server/queues/tasks/DuplicateCollectionDocumentsTask.ts:36-75` | Duplique une collection : racines par `parentDocumentId IS NULL`, puis `documentsDuplicator(..., recursive: true)`. | Même fuite n°3 pour toutes les lignes de la collection. | Idem. |
| `server/queues/processors/ImportsProcessor.ts:218` | Ajoute les documents importés à la structure. | Aucun au MVP (pas de lignes importées) ; à revoir pour l'import Notion L3. | — |
| `server/test/factories.ts:457`, `:691` | `buildDocument` / `buildCollection` ajoutent à la structure. | Une future `buildDatabaseRow` construite sur `buildDocument` fuirait. | Option dans la factory ou garde dans la primitive. |

---

## 2. Lecteurs de `documentStructure` et listes de documents

### 2.1 Lecteurs de l'arbre (bénéficient d'une structure propre)

| fichier:ligne | Rôle | Effet si les lignes sont exclues de la structure |
|---|---|---|
| `server/routes/api/collections/collections.ts:181` `collections.documents` | Renvoie l'arbre (cache) à la sidebar web. | Sidebar sans lignes (but de RG-21). |
| `server/routes/api/documents/documents.ts:933-960` `documents.documents` | Sous-arbre d'un document (`getDocumentTree`) ; utilisé par `app/models/UserMembership.ts:65`, `GroupMembership.ts:59`. | Enfants de l'hôte sans lignes. |
| `server/routes/api/documents/documents.ts:297-315` `documents.list` `sort=index` | Pagination par tranche de `documentStructure`. | Pas de lignes. |
| `server/mcp/tools/documents.ts:353` `list_collection_documents`, `server/mcp/tools/fetch.ts:180`, `server/mcp/tools/documents.ts:134` | Arbre/ordre exposés à l'MCP. | Pas de lignes. |
| `server/mcp/util.ts:242-290` `getDocumentBreadcrumb` / `getBreadcrumbsForDocuments` | Fil d'Ariane par recherche dans l'arbre. | **Effet de bord** : une ligne n'a plus de fil d'Ariane MCP. |
| `app/models/Document.ts:476` `pathTo` → `Collection.pathToDocument` (`app/models/Collection.ts:398`) | Fil d'Ariane web depuis l'arbre de collection. | **Effet de bord** : pas de chemin pour une ligne ; RG-27 (`Collection › Hôte › Database › Ligne`) demande une construction spécifique. |
| `app/scenes/Collection/components/CollectionMeta.tsx:44` | Compteur de documents de la collection = `flattenTree(collection.asNavigationNode)`. | Lignes non comptées (= proposition Q5 « Non »). |
| `server/commands/shareLoader.ts:95-120` | Partage : `sharedTree` = arbre de collection ou `getDocumentTree(hôte)` ; **l'accès à un document via un partage `includeChildDocuments` est autorisé seulement s'il est dans `getAllIdsInSharedTree(sharedTree)`**. | **Effet de bord** : les lignes deviennent inaccessibles via partage public (contredit RG-51, L2). |
| `server/commands/shareLoader.ts:218` | `getDocumentParents` pour trouver un partage parent. | Une ligne ne retrouve pas le partage de son hôte. |
| `server/routes/app.ts:290-310`, `server/utils/sitemap.ts:10-30` | Liste markdown des enfants d'un partage, sitemap. | Lignes absentes (cohérent ou non selon RG-51). |
| `plugins/search-postgres/server/PostgresSearchProvider.ts:233-238` | Recherche dans un partage de collection : restreint aux `getAllDocumentIds()`. | Lignes non trouvables dans un partage de collection ; en revanche le partage de document (`:239-252`) passe par `findAllChildDocumentIds` (par `parentDocumentId`) et **inclut** les lignes → asymétrie. |
| `server/queues/tasks/ExportTask.ts:144-152` | Export d'un document seul : `getDocumentTree(document.id)` ; **lève « Document not found in collection tree »** si absent. | **Effet de bord** : exporter une ligne échoue. |
| `server/queues/tasks/ExportDocumentTreeTask.ts:415-425`, `ExportJSONTask.ts:82`, `:183` (+ `ExportMarkdownZipTask`, `ExportHTMLZipTask`, `ExportTextBundleZipTask`, `ExportOKFZipTask`) | Exports de collection parcourent la structure. | Lignes absentes des exports (spec : export lignes + front-matter en L2). |
| `server/queues/tasks/JSONAPIImportTask.ts:397-447` | Ré-import d'un export JSON par sa `documentStructure`. | Cohérent avec l'export. |
| `server/models/helpers/DocumentHelper.tsx:1430` `sortDocumentsByStructure` | Tri selon l'arbre (duplication). | Les lignes non trouvées dans l'arbre sont triées en fin (sans erreur). |

### 2.2 Listes de documents côté serveur (requêtes SQL, ignorent `documentStructure`)

Le `defaultScope` de `Document` (`server/models/Document.ts:150-168`, `publishedWhere` `:138-148`) filtre `publishedAt`, `template`, `sourceMetadata.trial` : c'est le seul filtre transversal existant. **Aucune** de ces listes ne filtre aujourd'hui sur un critère équivalent à `databaseId`.

| fichier:ligne | Endpoint / rôle | Inclut les lignes aujourd'hui ? | Patchs envisageables |
|---|---|---|---|
| `documents.ts:200-475` `documents.list` | Liste générique : onglets de collection (récents, publiés, alphabétique, populaires, moins récents), Home « Recently updated » / « Popular » / « Created by me », enfants via `parentDocumentId` (`app/stores/DocumentsStore.ts:396` `fetchChildDocuments`). | Oui | (a) `where[Op.and].push({ databaseId: null })` par défaut + paramètre `includeDatabaseRows` (spec §12) ; (b) champ `databaseId` dans le DSL de filtres (`filters`) ; (c) exception quand `backlinkDocumentId` est fourni (RG-21 : backlinks inclus). |
| `documents.ts:476-532` `documents.archived` | Archive. | Oui (et en masse après archivage d'un hôte ou d'une collection). | Filtre par défaut ; ou n'exclure que les lignes dont l'hôte est lui-même archivé. |
| `documents.ts:533-604` `documents.deleted` | Corbeille. | Oui (en masse après suppression d'un hôte, cascade `deleteDocument`). | Filtre partiel (exclure les lignes supprimées par cascade, garder les lignes supprimées seules — RG-24) ; ou affichage groupé. |
| `documents.ts:605-671` `documents.viewed` | Récemment consultés (Home, sidebar `HistoryNavigation`, menu `@`/liens `app/editor/components/DocumentMenu.tsx:61`, command bar). | Oui | Filtre (spec RG-21 « liste des documents récents ») ; à arbitrer pour le menu de liens où RG-21 inclut les mentions. |
| `documents.ts:672-728` `documents.drafts` | Brouillons. | Seulement si une ligne est non publiée (cas non prévu, spec §12). | Filtre défensif. |
| `server/routes/api/pins/pins.ts:134`, `server/routes/api/stars/stars.ts:101` | Épingles / favoris. | Oui si l'utilisateur épingle/étoile une ligne (action explicite). | Probablement légitime ; ou interdire l'épinglage de lignes. |
| `PostgresSearchProvider.ts:311` `searchTitlesForUser`, `:368` `searchForUser`, `:754` `buildWhere` | Recherche, `documents.search_titles` (mentions `@`, liens), MCP `list_documents` (`server/mcp/tools/documents.ts:176`, `:277`). | Oui (souhaité RG-21/RG-45). | Ajouter un filtre optionnel « inclure les lignes » / par database dans `buildWhere`. |
| `server/routes/api/developer/developer.ts:85` | Génération de notifications de démo. | Oui (outil de dev). | Aucun nécessaire. |
| `server/utils/updates.ts:27` | Télémétrie : `Document.count()` envoyé au serveur de mises à jour. | Oui | Neutre ou filtrer. |

### 2.3 Listes côté client (MobX, calculées depuis le store)

Fait important : les listes web ne rendent **pas** la réponse API mais des `computed` sur `DocumentsStore` (`PaginatedDocumentList documents={...} fetch={...}`). **Tout document-ligne présent dans le store y apparaîtra**, quel que soit le filtre serveur. Les lignes entrent dans le store notamment par :

- `socket.on("documents.update")` → `documents.add(event)` (`app/components/WebsocketProvider.tsx:293-301`), alimenté par `WebsocketsProcessor` qui émet le document présenté sur les canaux de la collection (`server/queues/processors/WebsocketsProcessor.ts:221-233`) → **chaque membre de la collection reçoit chaque ligne modifiée** ;
- `fetchChildDocuments` (`DocumentsStore.ts:396`) appelé par `app/components/Sidebar/components/DocumentTree.tsx:209` et `SharedWithMeLink.tsx:93` sur le document actif (l'hôte) → `documents.list { parentDocumentId: hôte }` → lignes ;
- ouverture d'une ligne (peek/pleine page), résultats de recherche.

| fichier:ligne | Liste | Patchs envisageables |
|---|---|---|
| `app/stores/DocumentsStore.ts:104` `all` → `:109` `recentlyViewed`, `:118` `recentlyUpdated`, `:123` `popular`, `:133` `createdByUser` | Home (`app/scenes/Home.tsx:51-145`), command bar (`useRecentDocumentActions.tsx`, `SearchActions.tsx:46`), menu de liens éditeur (`DocumentMenu.tsx:83`). | (a) filtrer `!d.databaseId` dans `all` (toutes les listes dérivées) ; (b) filtrer liste par liste ; (c) ne pas ajouter au store les événements de lignes (côté `WebsocketsProcessor` ou handler). |
| `DocumentsStore.ts:149` `inCollection`, `:176`, `:193`, `:209` `rootInCollection`, `:230`, `:242`, `:254`, `:266`, `:278` | Onglets de la scène collection (`app/scenes/Collection/index.tsx:257-313`). | Idem. |
| `DocumentsStore.ts:302` `archived`, `:841` `deletedFiltered`, `:869` `draftsFiltered`, `:325` `totalDrafts` | Archive, corbeille, brouillons. | Idem, selon les choix §2.2. |
| `app/models/Document.ts:704` `childDocuments` → `:712` `asNavigationNode` | Enfants calculés par `parentDocumentId` dans le store ; `asNavigationNode` est utilisé par `SharedWithMeLink.tsx:127`, `SharedDocumentLink.tsx:86`, `DocumentTree.tsx:99`, `useCollectionDocuments.ts:22`, `DocumentListItem.tsx:132`, `Collection.addDocument` (`app/models/Collection.ts:353-373`), `NavigableModel.addDocument` (`app/models/base/NavigableModel.ts:139-157`), et `pathTo` (`Document.ts:490`). | **Fuite client** : l'hôte, une fois ses lignes dans le store, expose ses lignes comme enfants dans ces arbres (notamment « Partagés avec moi »). Filtrer `!doc.databaseId` dans `childDocuments`. |
| `app/components/WebsocketProvider.tsx:66` `invalidateChildPolicies`, `app/models/Policy.ts:61` | Parcourent `childDocuments` pour invalider des policies. | Inclure les lignes y est plutôt souhaitable (permissions héritées) ; à garder en tête si `childDocuments` est filtré. |
| `app/models/Collection.ts:298` `updateDocument`, `:327` `removeDocument` | Mise à jour en mémoire de l'arbre sur événements. | No-op pour un nœud absent : pas de fuite. |

---

## 3. Autres canaux où une ligne « sort » de sa database

| fichier:ligne | Canal | Fait | Options |
|---|---|---|---|
| `server/queues/tasks/DocumentPublishedNotificationsTask.ts:121-137` + `server/models/helpers/NotificationHelper.ts:157-200` | Notification « document publié » aux abonnés de la collection / du document. | Une ligne étant publiée à sa création, **chaque nouvelle ligne notifie** les abonnés de la collection. RG-21 inclut les notifications (mentions) sans préciser ce type. | Exclure `PublishDocument` pour les lignes ; garder les mentions. |
| `plugins/slack/server/processors/SlackProcessor.ts:20-27`, `:137` | Post Slack « nouveau document publié ». | Chaque ligne publiée serait postée. | Filtrer `databaseId`. |
| `plugins/webhooks/server/tasks/DeliverWebhookTask.ts:201` | Webhook `documents.publish`. | Les lignes déclenchent `documents.*` (spec L2 : événements `database_rows.*` dédiés). | Conserver ou filtrer selon le contrat webhook. |
| `server/queues/processors/WebsocketsProcessor.ts:74-118` (`documents.create/publish/restore`) | Envoie `entities` avec `collectionIds` → le client refetch `collections.documents`. | Pas de fuite (le client ignore les documents absents du store, `WebsocketProvider.tsx:191-193`), coût : refetch d'arbre à chaque création de ligne. | — |

---

## 4. Synthèse factuelle

1. **Un seul point d'insertion** dans `documentStructure` : `Collection.addDocumentToStructure` (`Collection.ts:1176`). Ses entrées sont : hook `@AfterCreate` (`Document.ts:549`), `publish` (`:1337`), `restoreTo` (`:1510`), `documentMover` (`:98`), `ImportsProcessor` (`:218`), factories de test.
2. **Deux mécanismes de fuite distincts** : (n°1) insertion de la ligne elle-même ; (n°2) injection des lignes d'un hôte via `Document.toNavigationNode` (`Document.ts:1592`), qui relit les enfants en base par `parentDocumentId`. Filtrer seulement les appelants ne couvre pas n°2.
3. **Fuite n°3** : la duplication (document récursif et collection) recrée les lignes en documents ordinaires publiés (`documentDuplicator.ts:164-190`, `DuplicateCollectionDocumentsTask.ts:36-75`).
4. **Toutes les listes SQL** (`documents.list/archived/deleted/viewed/drafts`) incluent les lignes ; le `defaultScope` (`Document.ts:150`) est l'unique filtre transversal existant et un candidat de patch (avec le coût d'opt-out partout où l'on veut les lignes : recherche, backlinks, chargement par id).
5. **Le client recalcule ses listes depuis le store** : un filtre serveur ne suffit pas, car les événements websocket `documents.update` (envoyés à toute la collection) et `fetchChildDocuments` y injectent les lignes ; `Document.childDocuments` (`app/models/Document.ts:704`) les expose comme enfants de l'hôte.
6. **Effets de bord de l'exclusion de la structure** (à traiter, pas des fuites) : export d'une ligne seule en erreur (`ExportTask.ts:148`), lignes inaccessibles via partage `includeChildDocuments` (`shareLoader.ts:114-118`), fil d'Ariane absent (web `pathTo`, MCP `getDocumentBreadcrumb`), hôte non dépubliable (`Document.ts:1404-1412`), hôte → brouillons reparente les lignes (`documentMover.ts:145-155`), coût de verrou/sauvegarde de collection à chaque édition de titre de ligne (`Document.ts:516` → `Collection.ts:1144`).
