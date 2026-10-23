# Recherche : points d'extension du front et de l'éditeur

Ticket : m-renou/outline#5 (enfant de #1). Spec : `docs/evolution/02-specifications-databases-outline.md` §3.1, §3.2, §11.3, §11.4.
Source unique : le code du fork au commit `0545db908`. Toutes les références sont `fichier:ligne`. Ce document liste des faits et des options ; il ne tranche pas.

## 1. Ce qu'un plugin peut enregistrer côté client

### 1.1 Mécanisme

- Les plugins client sont chargés par un glob Vite sur `plugins/*/client/index.{ts,js,tsx,jsx}` : `app/utils/PluginManager.ts:145`, appelé au démarrage dans `app/index.tsx:35`.
- Enregistrement via `PluginManager.add([...])` : `app/utils/PluginManager.ts:77-111`. Filtrage possible par `deployments` (cloud / community / enterprise) : `app/utils/PluginManager.ts:87-95`. Ordre par `priority` : `app/utils/PluginManager.ts:119-122`.
- L'enum `Hook` client ne contient que **trois** types : `Settings`, `Imports`, `Icon` (`app/utils/PluginManager.ts:14-18`). Formes des valeurs : `app/utils/PluginManager.ts:23-49`.

### 1.2 Où ces hooks sont consommés (exhaustif)

| Hook | Consommateur | Effet |
|---|---|---|
| `Hook.Settings` | `app/hooks/useSettingsConfig.ts:263-283` | Ajoute une entrée + une route dans les paramètres (`integrationSettingsPath(id)` ou `settingsPath(id)`), montée par `app/routes/settings.tsx:24-26`. Composant lazy, `enabled(team, user)`. |
| `Hook.Imports` | `app/scenes/Settings/Import.tsx:114`, `app/scenes/Settings/components/ImportListItem.tsx:119` | Ajoute une source d'import dans Paramètres > Import. |
| `Hook.Icon` | `app/components/PluginIcon.tsx:16-29` (utilisé par les écrans d'authentification) | Icône d'un fournisseur (ex. `plugins/slack/client/index.tsx:21`). |

Aucun autre appel à `PluginManager` n'existe dans `app/`. Faits qui en découlent :

- **Pas de hook de route/scène** : les routes authentifiées sont codées en dur dans `app/routes/authenticated.tsx` (ex. scène document `app/routes/authenticated.tsx:149-152`). Seules les routes de paramètres sont extensibles (via `Hook.Settings`).
- **Pas de hook de menu, d'action, de sidebar, ni d'éditeur** : les listes d'extensions de l'éditeur sont des constantes statiques (voir §2.1).
- Le serveur a un `PluginManager` distinct avec 11 hooks (`server/utils/PluginManager.ts:35-47`), aucun ne touchant au front ni au schéma de l'éditeur.

### 1.3 Ce que fait réellement `plugins/diagrams`

- `plugins/diagrams/client/index.tsx:7-20` n'enregistre **qu'un `Hook.Settings`** (URL d'instance diagrams.net, admin seulement). Il n'y a pas de dossier `server/`.
- Toute la partie éditeur des diagrammes vit dans le **core** :
  - extension `shared/editor/extensions/Diagrams.ts:34-56` (commande `editDiagram`, sans nœud propre : un diagramme est un nœud `image`, cf. `Diagrams.ts:64-72`) ;
  - ajoutée en dur dans `app/editor/extensions/index.ts:5,33` ;
  - entrée de menu slash en dur `app/editor/menus/block.tsx:280-285` ;
  - le réglage du plugin rejoint l'éditeur via les intégrations d'embed : `app/hooks/useEmbeds.ts:43-45` injecte les settings, lus par `shared/editor/extensions/Diagrams.ts:270-271`.
- Conséquence factuelle : `plugins/diagrams` n'est **pas** un exemple de plugin qui ajoute un nœud ; c'est un exemple de « réglage en plugin + code éditeur en core ».

### 1.4 Précédents de couplage core → plugins

Le core importe déjà du code de plugins via l'alias `plugins/*` (`tsconfig.json:36`) : `shared/schema.ts:12` (`plugins/notion/shared/types`), `server/models/AuthenticationProvider.ts:30-33`. Six plugins ont un dossier `shared/` (figma, github, gitlab, linear, notion, slack).

## 2. Ajouter un nœud ProseMirror `atom` + NodeView React

### 2.1 Où les nœuds sont déclarés

- Liste canonique : `shared/editor/nodes/index.ts`. `richExtensions` (`shared/editor/nodes/index.ts:111-134`), `withComments` (`:139-143`).
- Cette liste est consommée en dur par :
  - l'éditeur de document : `app/scenes/Document/components/Editor.tsx:46-49` (`[CodeWordBreak, ...withUIExtensions(withComments(richExtensions))]`) ;
  - les extensions UI (menus, slash, diagrams) : `app/editor/extensions/index.ts:20-38` ;
  - autres éditeurs : `RevisionViewer.tsx:96`, `PresentationMode.tsx:291`, `Collection/components/Overview.tsx:23`, aperçus de survol (`app/components/HoverPreview/*`) ;
  - le helper client `app/models/helpers/ProsemirrorHelper.ts:12` ;
  - **le schéma serveur** : `server/editor/index.ts:33-50` (schéma, parser, serializer) ;
  - le diff : `shared/editor/lib/ChangesetHelper.ts:301`.
- Le prop `extensions` de l'éditeur accepte classes ou instances : `app/editor/index.tsx:93`, instanciation dans `shared/editor/lib/ExtensionManager.ts:21-60` (les `Extension` pures sont ignorées en lecture seule, `:33-39`).

### 2.2 Contrainte : le nœud doit exister dans le schéma serveur

- Le serveur reconstruit le document avec `Node.fromJSON(schema, …)` depuis le Y.js ou le JSON : `server/models/helpers/DocumentHelper.tsx:139-141,199-203`. Un type de nœud absent du schéma `server/editor/index.ts` n'est pas reconnu par ce schéma.
- Le texte Markdown du document est recalculé côté serveur : `document.text = serializer.serialize(doc)` (`server/models/helpers/DocumentHelper.tsx:827`). Le `toMarkdown` par défaut d'un `Node` lève une erreur (`shared/editor/nodes/Node.ts:48-50`) : un nœud sans `toMarkdown` casse cette sérialisation.
- L'export HTML serveur rend les `component` React via des NodeViews serveur (`server/editor/nodeViews.tsx:37-…`), sauf si `allowComponentInStaticHTML` vaut `false` (`shared/editor/lib/Extension.ts:64-71`), auquel cas `toDOM` est utilisé (`server/editor/nodeViews.tsx:20-35`).

### 2.3 Anatomie d'un nœud (modèle : `Attachment`)

- Base : `Node` (`shared/editor/nodes/Node.ts:14-55`) — `name`, `schema` (NodeSpec), `commands`, `keys`, `inputRules`, `toMarkdown`, `parseMarkdown`, `markdownToken`. `ReactNode` ajoute un `component` abstrait (`shared/editor/nodes/ReactNode.ts:4-10`).
- Exemple atom de bloc avec composant React : `shared/editor/nodes/Attachment.tsx` — `group: "block"`, `atom: true` (`:62-64`), `parseDOM`/`toDOM` avec `sanitizeUrl` (`:65-87`), `leafText` (`:88`), `component = (props) => …` (`:134-174`), commandes `create<Name>` (`:177-180`), sélection par `NodeSelection` (`:92-104`).
- Branchement NodeView : l'éditeur crée une `ComponentView` pour **toute extension ayant une propriété `component`** (`app/editor/index.tsx:466-487`). La `ComponentView` (`app/editor/components/ComponentView.tsx:54-93`) rend le composant via un `NodeViewRenderer` (portail React) ajouté à `editor.nodeRenderers` (`app/editor/index.tsx:256`, rendu `:1197`), donc **dans l'arbre React de l'app** (accès possible aux stores/contextes).
- Comportement événements : `stopEvent` laisse passer seulement `mousedown`/drag/drop à ProseMirror (`ComponentView.tsx:185-197`) ; `ignoreMutation` ignore les mutations hors `contentDOM` (`:206-211`) — adapté à un bloc interactif (grille, inputs).
- Props reçues : `node`, `view`, `isSelected`, `isEditable`, `getPos`, `decorations`, `contentRef` (`ComponentView.tsx:213-223`).
- Barre d'outils de nœud : `selectionToolbarMenus()` (`shared/editor/lib/Extension.ts:126-135`).

### 2.4 Commande slash

- Le menu `/` est l'extension `BlockMenuExtension` (`app/editor/extensions/BlockMenu.tsx:13-26`), dont les items viennent d'une liste statique `blockMenuItems()` (`app/editor/menus/block.tsx:44-…`), complétée seulement par les modèles (`app/editor/components/BlockMenu.tsx:118-126`) et par les `embeds` (`app/editor/components/SuggestionsMenu.tsx:570-578`).
- Exécution d'un item : `commands[item.name]` sinon `commands["create" + Capitalize(item.name)]` (`SuggestionsMenu.tsx:358-370`). Un item est masqué si aucune de ces commandes n'existe (`SuggestionsMenu.tsx:635-638`). Il suffit donc qu'un nœud nommé `database` expose `createDatabase` (ou une extension expose une commande du même nom que l'item) pour que l'item fonctionne.
- Les attrs peuvent être une fonction de l'état (`SuggestionsMenu.tsx:359-360`, ex. date `block.tsx:191`). Filtrage conditionnel déjà pratiqué (Electron, `block.tsx:287-290`).

### 2.5 Sérialisation Markdown

- Le serializer est assemblé par `ExtensionManager.serializer()` à partir des `toMarkdown` (`shared/editor/lib/ExtensionManager.ts:126-143`) ; parsing via `parseMarkdown()` + `markdownToken` (`:150-169`), optionnel (`Node.ts:52-54`).
- Précédents de sérialisation en lien : `Embed.toMarkdown` écrit `[href](href)` (`shared/editor/nodes/Embed.tsx:196-209`) ; `Mention` écrit `[label](/doc/<id>)` pour un document (`shared/editor/nodes/Mention.tsx:396-401`). Un `[🗃 Titre](url)` (spec §11.3) suit ce motif ; sans `parseMarkdown`, ce lien serait relu comme lien ordinaire.

### 2.6 Options de placement du code du nœud (non tranché)

| Option | Faits pertinents |
|---|---|
| A. Nœud dans `shared/editor/nodes/` + ajout à `richExtensions` | Comme tous les nœuds actuels ; atteint automatiquement client, serveur, diff, aperçus (§2.1). |
| B. Nœud dans `plugins/databases/shared/` importé par le core | Précédent d'import core → plugin (`shared/schema.ts:12`) ; nécessite quand même de modifier `shared/editor/nodes/index.ts` ou `server/editor/index.ts` + `app/scenes/Document/components/Editor.tsx`. |
| C. Nouveau hook client (ex. `Hook.EditorExtension`) | N'existe pas (`app/utils/PluginManager.ts:14-18`). Les plugins sont chargés de façon asynchrone (`PluginManager.ts:140-150`) alors que les listes d'extensions sont des constantes de module ; le schéma serveur n'a aucun équivalent. |

Dans tous les cas, le schéma serveur (`server/editor/index.ts:33`) et la liste de l'éditeur document doivent contenir le nœud (§2.2).

## 3. Scène document : database pleine largeur ou bloc sous le titre

### 3.1 Structure actuelle

- Scène : `app/scenes/Document/components/Document.tsx`. Grille `Main` dépendant de `document.fullWidth` (`:431-435`, styles `:521-552`) ; `EditorContainer` (`:447-498`) contient `Notices` (`:461`), sommaire (`:463-467`), puis `Editor` (`:468-495`) dont les `children` sont les `References` (`:491-494`). `{children}` de la scène après `Main` (`:509`).
- Composant éditeur : `app/scenes/Document/components/Editor.tsx` — `DocumentTitle` (`:225-239`), puis `DocumentMeta` (`:247-258`), puis l'éditeur ProseMirror dans un `Suspense` (`:262-300`), puis `<div ref={childRef}>{children}</div>` (`:301`).
- `fullWidth` est un attribut existant du document (`server/models/Document.ts:369`), qui pilote la grille et la variable `--full-width-transform-offset` (`Document.tsx:377-378`).
- Des nœuds savent déjà sortir de la colonne : images `full-width` (`shared/editor/nodes/Image.tsx:66,657`, CSS `shared/editor/components/Styles.ts:1149-1167`) et tables `TableLayout.fullWidth` (`shared/editor/nodes/TableView.ts:82-85`).
- Le nœud racine `doc` a `content: "block+"` (`shared/editor/nodes/Doc.ts:19-23`). Le verrouillage de transactions existe sous forme de `filterTransaction` (`shared/editor/extensions/MaxLength.ts:21`, `Diff.ts:172`).

### 3.2 Points d'injection possibles (options, non tranchées)

**Bloc sous le titre (RowProperties, §11.2 / §11.4)**

1. Entre `DocumentTitle` et `DocumentMeta` ou juste après `DocumentMeta` dans `Editor.tsx:239-258` (rendu conditionnel sur `document.databaseId`).
2. Dans `Document.tsx` avant `<Editor>` (au niveau de `Notices`, `:461`) — mais ce serait au-dessus du titre, le titre étant rendu dans `Editor.tsx`.
3. Via le slot `children` de `Editor` (`Editor.tsx:301`) — rendu **après** le corps, donc pas sous le titre.
4. Comme décoration/widget ProseMirror en tête du document (`Extension.widget`, `shared/editor/lib/Extension.ts:77-86`) — dans l'éditeur, donc soumis à son cycle de vie.

**Database « full page » (§11.3)**

1. Nœud `database` dans le corps + mode pleine largeur du nœud (motif Image/Table `full-width`, §3.1) et/ou `document.fullWidth = true`.
2. Branche conditionnelle dans `Document.tsx:447-498` (ou `Editor.tsx:262-300`) qui rend le composant database à la place de l'éditeur quand le document est un hôte full page.
3. Scène dédiée : nécessite une nouvelle `Route` dans `app/routes/authenticated.tsx` (aucun hook de route, §1.2).
4. Non-suppression du nœud unique : `filterTransaction` dans une extension (précédent `MaxLength.ts:21`) ; `Doc.ts:21` impose au moins un bloc.

Toutes ces options modifient des fichiers du core (`app/scenes/Document/**`, `app/routes/**`) : aucune n'est atteignable par un hook plugin existant.

## 4. Réponse courte à la question du ticket

- Un plugin client ne peut enregistrer que des écrans de paramètres, des sources d'import et des icônes (`app/utils/PluginManager.ts:14-18`). Pas de routes/scènes, menus, actions ni extensions d'éditeur.
- Un nœud atom + NodeView React = une classe `Node` avec `component` (modèle `Attachment.tsx`), câblée automatiquement en `ComponentView` (`app/editor/index.tsx:466-487`), à ajouter aux listes statiques **et** au schéma serveur (`server/editor/index.ts:33`), avec un `toMarkdown` obligatoire (`DocumentHelper.tsx:827`, `Node.ts:48-50`).
- Slash command = item dans `app/editor/menus/block.tsx` + commande `create<Name>` ou commande nommée (`SuggestionsMenu.tsx:358-370`).
- `plugins/diagrams` ne contient que des réglages ; son code éditeur est dans le core (`shared/editor/extensions/Diagrams.ts`, `app/editor/menus/block.tsx:280-285`).
- La scène document offre plusieurs points d'injection (sous le titre dans `Editor.tsx`, branche dans `Document.tsx`, nœud full-width, nouvelle route), tous en patch core.
