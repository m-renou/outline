# Recherche : titre d'un document, état Y.js et écriture hors éditeur

Ticket : m-renou/outline#2 (enfant de #1). Spec : `docs/evolution/02-specifications-databases-outline.md` §8.4 et §10.3 (« point de vérification n°1 »).

Toutes les références sont relatives au commit `0545db908` du fork. Hocuspocus `1.1.3`, `yjs ^13.6.33`, `y-prosemirror ^1.3.7` (`package.json:80-83,276,280`). Le document consigne des faits, pas de décision.

## Réponse courte

- **Le titre ne fait pas partie de l'état Y.js dans le fork.** Le Y.Doc ne contient qu'un seul type partagé, le `Y.XmlFragment` nommé `"default"`, qui porte le corps ProseMirror. Le titre vit seulement dans la colonne SQL `documents.title` et dans l'observable MobX `Document.title` côté client.
- **`documents.update` avec `title` est donc une écriture SQL classique**, qui ne touche pas `state` et ne déclenche pas `APIUpdateExtension`. La diffusion vers un éditeur ouvert passe par l'événement `documents.update` sur le websocket applicatif (Socket.IO), pas par Hocuspocus.
- **La cohérence titre grille ↔ éditeur est du « dernier qui écrit gagne » au niveau du document entier**, sans fusion CRDT. Il existe une fenêtre de course : un client dont l'éditeur est ouvert renvoie tous ses champs `@Field`, titre compris, à chaque sauvegarde ; il peut donc écraser un titre posé par l'API s'il n'a pas encore reçu l'événement websocket, ou si l'utilisateur a le focus dans le champ titre.
- L'hypothèse de la spec (« le titre peut faire partie de l'état Y.js ») **ne s'applique pas à ce fork**. Il n'existe aucun « état collaboratif côté serveur » du titre à modifier.

## 1. Où vit le titre

| Couche | Emplacement | Preuve |
|---|---|---|
| Y.Doc (Hocuspocus) | Un seul type `XmlFragment` `"default"` = corps | `app/editor/extensions/Multiplayer.ts:56` (`doc.get("default", Y.XmlFragment)`), lié par `ySyncPlugin(type)` `:140`. Côté serveur, même nom : `server/collaboration/PersistenceExtension.ts:39` (`fieldName = "default"`), `server/commands/documentCollaborativeUpdater.ts:57` (`yDocToProsemirrorJSON(ydoc, "default")`), `server/models/helpers/DocumentHelper.tsx:832`. |
| SQL | Colonne `documents.title` | Écrite par `server/commands/documentUpdater.ts:78-80` (`document.title = title.trim()`), puis `saveWithCtx` `:170`. |
| Client | `@Field @observable title` du modèle MobX | `app/models/Document.ts:147-149`, rendu par `app/scenes/Document/components/Editor.tsx:229-230` (`title={... document.title}`) via `DocumentTitle` → `ContentEditable`. |

Aucune occurrence de `getMap`, `getText("title")` ou d'un champ titre dans le Y.Doc n'existe dans `app/`, `shared/` ou `server/` (recherche `grep`).

## 2. Chemin d'écriture depuis l'éditeur (multiplayer)

1. **Corps.** Les frappes passent par y-prosemirror vers Hocuspocus. `PersistenceExtension.onStoreDocument` (`server/collaboration/PersistenceExtension.ts:165-256`, avec `debounce: 3000` et `maxDebounce: 10000` dans `server/services/collaboration.ts:47-49`) appelle `documentCollaborativeUpdater`. Ce dernier écrit **uniquement** `content`, `state`, `lastModifiedById`, `collaboratorIds`, `editorVersion` et `revisionCount`, avec `hooks: false` (`server/commands/documentCollaborativeUpdater.ts:131-148`). Il **ne lit le titre que pour le recopier dans l'événement** (`:161`) et ne l'écrit jamais.
2. **Titre.** `handleChangeTitle` modifie `document.title` (MobX) puis déclenche un autosave débouncé de 3 s (`app/scenes/Document/hooks/useDocumentSave.ts:19,200-211,306-315`). Le blur du titre déclenche aussi une sauvegarde (`app/scenes/Document/components/Editor.tsx:121-124`). `onSave` appelle `document.save()` (`useDocumentSave.ts:168`), qui envoie `this.toAPI()`, c'est-à-dire **tous les `@Field`** (title, icon, color, fullWidth, preferences, templateId, collectionId, parentDocumentId…), vers `POST documents.update` (`app/models/Document.ts:635-658`, `app/stores/base/Store.ts:423-444`).

Le client éditeur utilise donc **déjà** `documents.update` pour le titre, exactement comme le prévoit la spec §8.4 pour la grille.

## 3. Ce que fait `documents.update` avec `title`

- La route se trouve dans `server/routes/api/documents/documents.ts:1463-1512` : `auth`, `rateLimiter(OneHundredPerMinute)`, `transaction()`, `authorize(user, "update")`, puis `documentUpdater`.
- `documentUpdater` (`server/commands/documentUpdater.ts`) :
  - affecte `title.trim()` (`:78-80`) ;
  - prend un verrou de ligne `SELECT … FOR UPDATE`, et, si `lastRevision` est fourni, l'ajoute au prédicat ; un écart lève `DocumentConflictError` (`:120-135`) ;
  - appelle `saveWithCtx(ctx, undefined, { data })` (`:167-170`), ce qui crée un `Event` `documents.update` via les hooks du modèle. Le `BeforeUpdate` incrémente `revisionCount` (`server/models/Document.ts:612-613`) ;
  - le hook `publishTitleChangeEvent` publie aussi `documents.title_change` (`server/models/Document.ts:642-654`, avec `publish: true, persist: false`). Aucun processeur ni handler client ne s'abonne à `title_change` (seule occurrence : `server/types.ts:261`).
- **`state` ne change que si `text` est fourni** : `DocumentHelper.applyMarkdownToDocument` réapplique le markdown dans le Y.Doc stocké et force `changed("state", true)` (`server/models/helpers/DocumentHelper.tsx:826-846`).

## 4. `APIUpdateExtension` : quand elle s'applique et ce qu'elle fait

- Elle est déclenchée par le hook `notifyCollaborationServer`, **uniquement si `model.changed("state")`** et seulement après commit (`server/models/Document.ts:657-671`). Une mise à jour du seul titre ne la déclenche donc pas.
- Quand elle est déclenchée, elle publie sur Redis `collaboration:api-update:<id>` (`server/collaboration/APIUpdateExtension.ts:180-196`). Chaque instance collaboration qui a ce document en mémoire relit `state` en base, calcule le diff `encodeStateAsUpdate(dbYdoc, encodeStateVector(memDoc))` et l'applique au Y.Doc en mémoire (`:112-171`). Ce diff ne contient que des **ajouts** de structures Y ; c'est une fusion CRDT du corps, pas un remplacement. Le titre n'y figure pas.

## 5. Scénario : titre modifié par l'API pendant qu'un éditeur est ouvert

1. L'API écrit `documents.title` et crée l'`Event` `documents.update`.
2. `WebsocketsProcessor` relit le document et émet `documents.update` avec `presentDocument` (titre à jour) sur les canaux du document (`server/queues/processors/WebsocketsProcessor.ts:221-232`).
3. Le client exécute `documents.add(event)` → `Model.updateData`, qui **écrase sans condition** chaque champ différent puis réinitialise `persistedAttributes` (`app/components/WebsocketProvider.tsx:293-303`, `app/stores/base/Store.ts:248-258`, `app/models/base/Model.ts:221-255`).
4. Affichage du titre :
   - champ **non focalisé** : `ContentEditable` remplace le texte affiché (`app/components/ContentEditable.tsx:150-170`, branche `setInnerValue`) ;
   - champ **focalisé** : le DOM n'est **pas** remplacé ; seul `lastValue` est mis à jour (`:160-166`). À la prochaine frappe ou au prochain blur, le texte du DOM est renvoyé à `onChange`, puis `documents.update`. **Le titre local de l'utilisateur réécrase alors le titre API.**
5. Le corps Y.js n'est pas affecté. La prochaine persistance collaborative (`documentCollaborativeUpdater`) ne touche pas `title` et émet un `documents.update` qui porte le titre courant en base (`:161`), donc sans régression du titre.

### Fenêtres de course identifiées (faits)

- **A. Autosave tardif.** Le client sauvegarde tous les `@Field` (`toAPI`). Si un autosave part (titre, icône, blur…) avant que l'événement websocket n'arrive, il renvoie l'**ancien** titre et annule l'écriture API. Le client n'envoie pas `lastRevision` (aucune occurrence de `lastRevision` dans `app/`), donc le verrou optimiste de `documentUpdater` n'intervient pas.
- **B. Focus dans le titre** : voir l'étape 4 ci-dessus.
- **C. État local `title` du hook.** `useDocumentSave` garde une copie `useState(document.title)` (`useDocumentSave.ts:106`), qui sert seulement aux tests de vacuité (`:119`) et n'est pas resynchronisée par le websocket. L'effet est mineur (auto-suppression de brouillon vide, `:54-82`).
- **D. Latence.** La propagation passe par la file d'événements (`Event` → `WebsocketsProcessor`), donc avec un délai asynchrone, contrairement à Y.js.

## 6. Options de chemin d'écriture pour la grille et leurs conséquences

| Option | Ce qu'elle implique | Conséquences |
|---|---|---|
| **1. `documents.update { id, title }`** (spec §8.4 actuelle) | Même chemin que l'éditeur lui-même. | Cohérent avec le reste d'Outline : événement, websocket, révisions, `title_change`, rate limit 100/min. Le comportement est le « dernier qui écrit gagne » au niveau du titre, avec les courses A et B. Aucune modification du core. |
| **2. Idem + `lastRevision`** | La grille envoie la `revisionCount` connue. | La grille détecte qu'elle écrase une modification concurrente (`DocumentConflictError`, `documentUpdater.ts:132-134`). Mais `revisionCount` est aussi incrémenté par chaque persistance collaborative du corps (`documentCollaborativeUpdater.ts:140`), donc des conflits « faux positifs » apparaissent pendant que quelqu'un tape dans le corps. Cela ne protège pas contre la course A côté éditeur. |
| **3. Écrire le titre via l'état collaboratif** (hypothèse spec §10.3) | Aucun support existant : il faudrait ajouter un type Y (par ex. `Y.Text`/`Y.Map`) au schéma Y.Doc, le lier au champ titre client, le persister vers `documents.title` dans `documentCollaborativeUpdater` et l'écrire côté serveur via un mécanisme du type `APIUpdateExtension`. | Fusion CRDT réelle du titre et élimination des courses A et B pour les clients connectés. Modification profonde du core (client, persistance, migration des documents existants dont le Y.Doc n'a pas ce type, compatibilité avec les clients d'ancienne version et `EditorVersionExtension`). Divergence durable avec l'upstream. |
| **4. Faire de `documents.update` une écriture sans `toAPI` complet côté éditeur** (envoyer seulement les champs modifiés) | Patch client dans `Document.save` / `useDocumentSave`. | Réduit la course A pour les autosaves qui ne concernent pas le titre (icône, blur sans changement…). La course reste si l'utilisateur édite lui-même le titre. Modification du core côté client. |

## 7. Points non vérifiés et pistes

- Ce document ne vérifie pas si l'upstream Outline (au-delà de `0545db908`) a migré le titre dans Y.js. Dans ce fork, ce n'est pas le cas.
- Le comportement multi-instance de la collaboration (`@hocuspocus/extension-redis`, `server/services/collaboration.ts:51-57`) ne change rien pour le titre, puisque celui-ci n'est pas dans le Y.Doc.
- Les courses A et B n'ont pas été reproduites en conditions réelles : elles sont déduites de la lecture du code. Le skill `verify-outline` permettrait de les démontrer.
