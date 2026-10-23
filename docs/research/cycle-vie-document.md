# Recherche : cycle de vie d'un document et points de greffe des cascades

Ticket : m-renou/outline#4 (enfant de #1). Spec : `docs/evolution/02-specifications-databases-outline.md` RG-04, RG-07, RG-21, RG-24, §12.
Source : code du fork au commit `0545db908`. Les références `fichier:ligne` pointent vers ce commit.
Ce document liste des **faits** et des **options**. Il ne tranche pas.

## 1. Carte des opérations

| Opération | Route (`server/routes/api/documents/documents.ts`) | Commande / méthode modèle | Événement émis |
|---|---|---|---|
| Création | `documents.create` :1838 | `documentCreator` `server/commands/documentCreator.ts:228-277` (build + `saveWithCtx`, puis `document.publish` si `publish`) | `documents.create`, puis `documents.publish` |
| Import | `documents.import` :1762 | `DocumentImportTask.scheduleAndWait` → `documentCreator` | `documents.create` (`source: "import"`) |
| Publication | `documents.update` avec `publish` :1504-1515 | `documentUpdater` `server/commands/documentUpdater.ts:162-166` → `Document.publish` `server/models/Document.ts:1282-1369` | `documents.publish` |
| Dépublication | `documents.unpublish` :1738 | `Document.unpublishWithCtx` `Document.ts:1395-1456` | `documents.unpublish` |
| Déplacement | `documents.move` :1602 | `documentMover` `server/commands/documentMover.ts:23-222` | `documents.move` (`data.documentIds`, `data.collectionIds`) |
| Archivage | `documents.archive` :1658 | `Document.archiveWithCtx` `Document.ts:1460-1479` → `archiveWithChildren` :1710-1739 | `documents.archive` |
| Corbeille | `documents.delete` (sans `permanent`) :1717-1730 | `Document.destroyWithCtx` `Document.ts:1536-1565` → `Collection.deleteDocument` `server/models/Collection.ts:1025-1052` | `documents.delete` |
| Restauration (corbeille / archive / révision) | `documents.restore` :1167 | `documentRestorer` `server/commands/documentRestorer.ts:29-98` → `Document.restoreTo` `Document.ts:1482-1533` | `documents.restore` ou `documents.unarchive` |
| Suppression définitive (unitaire) | `documents.delete` `permanent: true` :1700-1716 | `documentPermanentDeleter([document])` `server/commands/documentPermanentDeleter.ts:11-135` | `documents.permanent_delete` (émis par la route) |
| Vider la corbeille | `documents.empty_trash` :2286 | `EmptyTrashTask` `server/queues/tasks/EmptyTrashTask.ts` → `documentPermanentDeleter` | `documents.empty_trash` |
| Purge automatique (> 30 j) | — (cron horaire) | `CleanupDeletedDocumentsTask` `server/queues/tasks/CleanupDeletedDocumentsTask.ts:13-34` → `documentPermanentDeleter` | aucun |
| Duplication | `documents.duplicate` :1524 | `documentDuplicator` (option `recursive`) | `documents.create` par copie |

Opérations au niveau collection qui touchent les documents en masse (`server/routes/api/collections/collections.ts`) :

| Opération | Code | Effet sur les documents |
|---|---|---|
| `collections.delete` :866 | hook `@BeforeDestroy Collection.deleteDocuments` `Collection.ts:440-461` | `Document.update` en masse : `deletedAt`, `deletedById` sur tous les documents de la collection non archivés. Pas de hook ni d'événement par document. |
| `collections.archive` :895 | `Collection.archiveWithCtx` `Collection.ts:957-985` | `Document.update` en masse de `archivedAt`. |
| `collections.restore` :926 | route, :940-954 | `Document.update` en masse `archivedAt = null` pour les documents dont `archivedAt` = celui de la collection. |

## 2. Comment les cascades parent → enfants fonctionnent aujourd'hui

Le lien parent → enfant est la seule colonne `documents.parentDocumentId` (`Document.ts:693-698`). La découverte des descendants passe par `findAllChildDocumentIds` (`Document.ts:1196-1240`) : une CTE récursive, qui filtre `deletedAt IS NULL` par défaut (`paranoid`) et accepte un `where` supplémentaire. `findChildDocuments` (`Document.ts:1176-1186`) ne renvoie que le premier niveau.

Chaque opération a sa propre stratégie, sans mécanisme commun :

| Opération | Cascade vers les enfants ? | Mécanisme | Événements par enfant ? |
|---|---|---|---|
| Publication | Non | `publish` n'ajoute que le document, avec ses enfants publiés déjà présents (`toNavigationNode`, `Document.ts:1592-1653`). Copie les memberships du parent (`:1344-1360`). | — |
| Dépublication | **Refusée** si des enfants non archivés existent | `unpublishWithCtx` lève `InvalidRequestError` (`Document.ts:1404-1412`) | — |
| Déplacement, même collection | Implicite | Seul `parentDocumentId` du document change. Le sous-arbre JSON est retiré puis réinséré (`documentMover.ts:63-102`). | Non |
| Déplacement, autre collection | Oui | `Document.update({ collectionId })` en masse sur `findAllChildDocumentIds()` (`documentMover.ts:116-142`) ; pins nettoyés (`:183-207`) | Non : un seul `documents.move` avec la liste `documentIds` (`:212-218`) |
| Déplacement vers brouillons (`collectionId` null) | Non : les enfants sont **rattachés au grand-parent** | `documentMover.ts:143-159` | Non |
| Recalcul des permissions après move | Oui | `DocumentMovedProcessor` (`server/queues/processors/DocumentMovedProcessor.ts:11-55`) supprime puis recalcule les memberships sourcées du document et de tous ses descendants | — |
| Archivage | Oui | `archiveWithChildren` : parcours récursif niveau par niveau, `child.save()` (`Document.ts:1710-1739`) | Non : seul le parent émet `documents.archive` |
| Désarchivage | Oui | `restoreArchivedWithChildren` (`Document.ts:1672-1708`), pose aussi `collectionId` sur chaque enfant | Non : seul le parent émet `documents.unarchive` |
| Corbeille (document dans une collection, non archivé) | Oui | `Collection.deleteDocument` : retire du `documentStructure`, puis `destroy()` un par un des descendants, du plus profond au moins profond (`Collection.ts:1025-1052`). Les hooks de modèle tournent (`ParanoidModel.setDeletedBy`, `server/models/base/ParanoidModel.ts:33-40`). | Non : seul le parent émet `documents.delete` (`Document.ts:1564`) |
| Corbeille (brouillon sans collection, ou archivé dans une collection non archivée) | **Non** | branche `if (!deleted) this.destroy()` (`Document.ts:1544-1560`) | — |
| Restauration depuis la corbeille | **Non** | `restoreTo` ne restaure que le document lui-même (`Document.ts:1516-1522`). Les enfants supprimés en cascade restent dans la corbeille. | — |
| Restauration d'un enfant dont le parent est supprimé / brouillon / archivé | — | `parentDocumentId` est remis à `null` : le document revient à la racine de la collection (`Document.ts:1495-1507`) | — |
| Suppression définitive | Non (détachement) | Les enfants du lot ont `parentDocumentId = null` (`documentPermanentDeleter.ts:98-114`), puis destruction `force: true` par lots de 10 (`:119-133`). Les FK (vecteurs, révisions, commentaires…) cascadent en base. | Non |
| Duplication | Optionnelle | `documentDuplicator` `recursive` (`server/commands/documentDuplicator.ts:158`) | un `documents.create` par copie |

Points notables :

- L'arbre de la sidebar est un JSONB dénormalisé `collections.documentStructure`. Il est modifié par `addDocumentToStructure` (`Collection.ts:1176-1251`), `removeDocumentInStructure` (`:1054-1115`), `updateDocument` (`:1144-1174`), et par les hooks `Document.updateCollectionStructure` (`@BeforeSave`, `Document.ts:515-546`, sur changement de titre/icône/couleur) et `Document.addDocumentToCollectionStructure` (`@AfterCreate`, `Document.ts:548-567`, si `publishedAt` est posé à la création).
- `toNavigationNode` construit le sous-arbre à partir de **tous** les enfants publiés et non archivés par `parentDocumentId` (`Document.ts:1607-1634`). Tout document dont `parentDocumentId` pointe vers un document de l'arbre y entre donc à la publication, au déplacement ou à la restauration.
- Le hook `@BeforeUpdate checkParentDocument` (`Document.ts:616-640`) interdit les cycles via `parentDocumentId`.
- `WebsocketsProcessor` pousse `documents.delete` / `documents.archive` pour le seul document de l'événement (`server/queues/processors/WebsocketsProcessor.ts:221-232`), et pour `documents.move` invalide chaque id de `data.documentIds` (`:235-260`). `documents.permanent_delete` est diffusé à toute l'équipe (`:212-219`).
- Autres consommateurs des événements de cycle de vie : `BacklinksProcessor` (publish/update/delete), `SearchIndexProcessor` (publish/update.delayed/archive…), `NotificationsProcessor`, `RevisionsProcessor`, `DocumentArchivedProcessor` (retire l'étoile de l'acteur, `DocumentArchivedProcessor.ts:17-29`).
- La liste de la corbeille `documents.deleted` (:533) renvoie tous les documents `deletedAt IS NOT NULL` des collections de l'utilisateur, sans filtre sur la hiérarchie : les enfants supprimés en cascade y apparaissent individuellement.
- Aucun code `databaseId` / `DatabaseRow` n'existe encore dans `server/`, `shared/` ou `app/`.

## 3. Points de greffe possibles pour hôte → database → lignes

Ordonnés par opération de la spec §12. Pour chacune : les options visibles dans le code, sans recommandation.

### 3.1 Exclusion de `documentStructure` (§12 ligne 1, RG-21)

Tous les chemins d'insertion passent par `Collection.addDocumentToStructure` ou `Document.toNavigationNode` :
`Document.addDocumentToCollectionStructure` (`Document.ts:548`), `Document.publish` (`:1337`), `Document.restoreTo` (`:1510`), `documentMover` (`:98`), et la lecture des enfants dans `toNavigationNode` (`:1608-1634`).

- Option A : garde au début de `addDocumentToStructure` (`Collection.ts:1176`) + filtre `databaseId IS NULL` dans `toNavigationNode` (`Document.ts:1619-1632`). Couvre tous les appelants actuels en deux endroits.
- Option B : garde dans chaque appelant (publish, restoreTo, mover, hook AfterCreate).
- Option C : les lignes n'utilisent pas `parentDocumentId` vers l'hôte (lien porté uniquement par `databaseId`), ce qui les sort naturellement de `toNavigationNode` et de toutes les cascades `parentDocumentId` — mais alors aucune cascade existante ne les atteint (voir 3.2-3.4).

### 3.2 Déplacement de l'hôte (RG-07)

- Cascade existante : si les lignes (ou la database full page) sont des descendants via `parentDocumentId`, `documentMover.ts:119-142` met déjà à jour leur `collectionId` en masse, et `DocumentMovedProcessor` recalcule leurs memberships sourcées.
- Si le lien est `databaseId` (option C) : greffe dans `documentMover` après `:119` (ajouter les ids des lignes à `childDocumentIds`, ou un `Document.update` sur `databaseId IN (...)`), et éventuellement dans `DocumentMovedProcessor.destroyUserMemberships` / `destroyGroupMemberships` (`:57-89`).
- Cas « move vers brouillons » (`collectionId` null, `documentMover.ts:143-159`) : les enfants sont rattachés au grand-parent ; comportement à définir pour les lignes.
- Interdire le déplacement d'une ligne seule (§12) : garde possible dans la route `documents.move` (:1617, après `authorize(user, "move")`), dans la policy `move` (`server/policies/document.ts`) ou dans `documentMover`.
- Nombre de lignes jusqu'à 50 000 (§13.1) : le `Document.update` en masse existant est un seul `UPDATE ... WHERE id IN (...)` ; `documentMover.ts:162-167` recharge ensuite tous les descendants pour la réponse API et l'événement `data.documentIds`.

### 3.3 Corbeille de l'hôte ou de la database (RG-04)

- Cascade existante : `Collection.deleteDocument` (`Collection.ts:1025-1052`) détruit un par un tous les descendants `parentDocumentId`. Si les lignes sont descendantes, elles partent déjà en corbeille — mais un `destroy()` par ligne, sans événement par ligne, et seulement quand le document est dans une collection et non archivé (`Document.ts:1552`).
- Options de greffe :
  - dans `Document.destroyWithCtx` (`Document.ts:1536-1565`), qui couvre aussi la branche brouillon/archivé ;
  - dans `Collection.deleteDocument` (`Collection.ts:1030-1049`), à côté de la cascade existante ;
  - via un hook `@BeforeDestroy` / `@AfterDestroy` sur `Document` (aucun n'existe aujourd'hui hors `ParanoidModel.setDeletedBy`) ;
  - via un processor sur `documents.delete` (asynchrone, hors transaction de la requête).
- Mise à jour en masse (`Document.update` comme `Collection.deleteDocuments`, `Collection.ts:440-461`) vs `destroy()` par ligne : la première ne déclenche ni hooks ni `deletedById` automatique ; la seconde est ce que fait déjà `deleteDocument`. AGENTS.md demande, pour les suppressions en masse qui alimentent l'autorisation ou les websockets, de charger les modèles dans la transaction et de supprimer chacun avec le contexte.
- La suppression d'une collection (`Collection.deleteDocuments`) couvre déjà toute ligne portant le `collectionId` de la collection.

### 3.4 Restauration (RG-04, RG-24)

- Fait clé : **aucune restauration en cascade n'existe pour la corbeille**. `restoreTo` ne restaure que le document (`Document.ts:1516-1522`). Pour « la restauration restaure l'ensemble » (RG-04), il faut un nouveau code.
- Options de greffe : `Document.restoreTo` (branche `deletedAt`, `:1516`), `documentRestorer` (`documentRestorer.ts:70-75`), ou un processor sur `documents.restore`.
- Critère de sélection des lignes à restaurer : rien n'enregistre aujourd'hui qu'un document a été supprimé « par cascade ». Données disponibles : `deletedAt` (même horodatage ? non garanti, `destroy()` par ligne), `deletedById`. Options : comparer `deletedAt`, ajouter une colonne de lot, ou restaurer toutes les lignes supprimées de la database.
- Restauration d'une ligne seule (RG-24) : le comportement actuel pour un enfant dont le parent n'est pas actif est de le remettre à la racine (`Document.ts:1497-1507`). Une garde analogue sur `databaseId` (database supprimée → ligne convertie en document simple) se placerait au même endroit.
- `documentRestorer` exige une collection de destination active (`documentRestorer.ts:56-60`) ; les lignes héritent donc de cette contrainte.

### 3.5 Suppression définitive

- `documentPermanentDeleter` est le point unique pour les trois déclencheurs (route `permanent`, `EmptyTrashTask`, `CleanupDeletedDocumentsTask`). Il détache les enfants (`parentDocumentId = null`) au lieu de les détruire (`documentPermanentDeleter.ts:98-114`).
- Options : cascade FK `ON DELETE CASCADE` de `databases` → lignes dans la migration ; ou suppression explicite dans `documentPermanentDeleter` ; ou laisser la purge à 30 jours traiter chaque ligne indépendamment (les lignes ont leur propre `deletedAt`).
- Attention au détachement existant : si les lignes ont `parentDocumentId` = hôte et que l'hôte est purgé avant elles, elles deviennent des documents racines (`:98-114`).

### 3.6 Publication / brouillons (§12 : ligne publiée dès sa création)

- `documentCreator` publie via `document.publish` seulement si `publish` est vrai et `collectionId` fourni (`documentCreator.ts:265-277`). `publish` ajoute au `documentStructure` (`Document.ts:1329-1342`) et copie les memberships du parent (`:1344-1360`).
- Options : appeler `publish` avec une garde structure (3.1), ou poser `publishedAt` directement à la construction (`documentCreator.ts:240`, déjà accepté en paramètre) — ce qui déclenche le hook `@AfterCreate addDocumentToCollectionStructure` (`Document.ts:548-567`) sauf garde.
- `unpublishWithCtx` refuse si des enfants non archivés existent (`Document.ts:1404-1412`) : si les lignes sont descendantes `parentDocumentId` de l'hôte, l'hôte devient non dépubliable.

### 3.7 Archivage (non cité par la spec)

- `archiveWithChildren` / `restoreArchivedWithChildren` (`Document.ts:1672-1739`) et `Collection.archiveWithCtx` / `collections.restore` cascadent par `parentDocumentId` ou `collectionId`. La spec ne décrit pas l'archivage d'un hôte ; les lignes y seront prises ou non selon le choix de lien (3.1).

## 4. Synthèse des faits structurants

1. Le lien hiérarchique est `parentDocumentId` seul ; toutes les cascades existantes l'utilisent, chacune avec sa propre implémentation (pas de service commun).
2. Les cascades émettent **un seul événement** pour la racine ; les descendants ne reçoivent pas d'événement propre.
3. Cascades existantes : move inter-collection (`collectionId`), archive/unarchive, corbeille (si dans une collection et non archivé), permissions sourcées après move.
4. Cascades absentes : restauration depuis la corbeille, suppression définitive (détachement à la place), dépublication (refus).
5. Tout enfant `parentDocumentId` publié entre dans `documentStructure` via `toNavigationNode` ; exclure les lignes demande soit une garde dans `addDocumentToStructure`/`toNavigationNode`, soit de ne pas utiliser `parentDocumentId` pour elles.
