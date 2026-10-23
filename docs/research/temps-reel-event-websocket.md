# Recherche : chaîne temps réel Event → websocket → stores MobX

Ticket : m-renou/outline#6 (enfant de #1). Spec de référence : `02-specifications-databases-outline.md` §10.1.
Source : code du fork au commit `0545db908`. Toutes les références sont `fichier:ligne` sur ce commit.
Ce document constate des faits et liste des options ; il ne tranche pas.

## 1. Vue d'ensemble

```
mutation (createWithCtx / updateWithCtx / destroyWithCtx / Event.createFromContext)
  └─ Model.insertEvent → INSERT events (même transaction)          server/models/base/Model.ts:292-397
       └─ Event @AfterSave → afterCommit → globalEventQueue().add   server/models/Event.ts:80-95
            └─ worker : boucle sur le registre des processors       server/services/worker.ts:22-97
                 ├─ "WebsocketsProcessor" → websocketQueue().add     server/services/worker.ts:63-66
                 └─ autres → processorEventQueue (applicableEvents)  server/services/worker.ts:67-78
                      └─ service websockets consomme websocketQueue  server/services/websockets.ts:153-174
                           └─ WebsocketsProcessor.perform(event, io) server/queues/processors/WebsocketsProcessor.ts:42
                                └─ io.to(rooms).emit(nom, payload)   (adapter Redis multi-instance, websockets.ts:95)
                                     └─ client socket.on(nom) → store.add/remove   app/components/WebsocketProvider.tsx
```

## 2. Création de l'Event

- `Event.name` est une colonne `STRING` libre (≤ 255 car.) : aucune contrainte d'énumération en base — `server/models/Event.ts:41-47`.
- Les helpers `createWithCtx` / `saveWithCtx` / `updateWithCtx` / `destroyWithCtx` posent `event.publish = true` dans le contexte de hook — `server/models/base/Model.ts:69-130, 201-240`.
- Les hooks Sequelize `AfterCreate/AfterUpsert/AfterUpdate/AfterDestroy/AfterRestore` appellent `insertEvent("create"|"update"|"delete", …)` — `server/models/base/Model.ts:245-286`.
- `insertEvent` (`server/models/base/Model.ts:292-397`) :
  - nom = `${eventNamespace ?? tableName}.${eventOpts.name ?? verbe}` (l. 297, 363) ; un `eventOpts.name` contenant un point lève `InternalError` (l. 317-321).
  - déduit `collectionId`, `documentId`, `userId`, `teamId` des attributs du modèle s'ils existent (l. 323-360) ; `actorId` = utilisateur authentifié (l. 375-379).
  - `changes` = `model.previousChangeset` (diff attributs/précédent calculé en `BeforeSave`, l. 240-243, 382) ; `data` = `eventOpts.data` libre (l. 383).
  - `persist !== false` → `event.create` dans la transaction (l. 386-389) ; `persist: false` → `Event.schedule(...)` en `afterCommit`, sans ligne en base (l. 390-396 ; `Event.schedule` : `server/models/Event.ts:134-147`).
- Création manuelle possible : `Event.createFromContext(ctx, attrs)` — `server/models/Event.ts:160-189`.
- **Fait utile pour la spec §10.2** : une mise à jour SQL brute (`values = values || $patch`) ne déclenche pas les hooks de modèle ; l'Event doit alors être créé explicitement (`Event.createFromContext` ou `Event.schedule`).
- Enfilement : `@AfterSave enqueue` met l'event dans `globalEventQueue` après commit (parent de transaction si `findOrCreate`) — `server/models/Event.ts:80-95`.

## 3. Dispatch worker et file websocket

- Files Bull : `globalEvents`, `processorEvents`, `websockets` (timeout 10 s, pas de retry configuré), `tasks` — `server/queues/index.ts:5-54`.
- Registre des processors : tous les fichiers de `server/queues/processors/` + `PluginManager.getHooks(Hook.Processor)` — `server/queues/processors/index.ts:9-23`.
- Le worker parcourt le registre pour chaque event (`server/services/worker.ts:53-87`) :
  - `WebsocketsProcessor` est un cas spécial par **nom** : **tout** event est poussé dans `websocketQueue`, sans filtre `applicableEvents` (l. 63-66).
  - les autres processors reçoivent un job `processorEventQueue` si `applicableEvents` contient le nom ou `"*"`, et si `shouldQueue` (optionnel) l'accepte (l. 67-78 ; contrat dans `server/queues/processors/BaseProcessor.ts:4-40`).
- Les processors du worker (`processorEventQueue`, `server/services/worker.ts:101-160`) ne reçoivent que `event` : pas d'instance socket.io.
- `websocketQueue` n'est consommée que par le service `websockets` (`server/services/websockets.ts:152-174`), qui instancie `WebsocketsProcessor` et appelle `perform(event, io)`. Les erreurs sont loguées, pas relancées (l. 165-170).
- Multi-instance : `io.adapter(createAdapter(Redis.defaultClient, Redis.defaultSubscriber))` (`@socket.io/redis-adapter`) — `server/services/websockets.ts:95`, dépendance `package.json:115`.

## 4. Rooms

À la connexion (`server/services/websockets.ts:97-150`), authentification par cookie `accessToken` (JWT, l. 205-221), puis `authenticated()` (l. 176-203) joint :

| Room | Qui | Réf. |
|---|---|---|
| `team-<teamId>` | tout utilisateur de l'équipe | websockets.ts:183 |
| `user-<userId>` | l'utilisateur | websockets.ts:183 |
| `team-<teamId>.members` | non-invités uniquement | websockets.ts:187-189 |
| `collection-<id>` | chaque id de `user.collectionIds()` | websockets.ts:193-199 |
| `group-<id>` | chaque groupe de l'utilisateur | websockets.ts:193-200 |

- **Il n'existe pas de room par document.** Un document est diffusé via `collection-*` (+ `team-*.members` si collection non privée) s'il est publié, plus `user-*` / `group-*` des memberships directs du document — `getDocumentEventChannels`, `WebsocketsProcessor.ts:1055-1097` ; `getCollectionEventChannels`, l. 1035-1053. L'acteur reçoit toujours via `user-<actorId>`.
- Les rooms sont maintenues dynamiquement par le processor : `socketsJoin`/`socketsLeave` sur `collections.create|update|add_user|remove_user|add_group|remove_group`, `groups.add_user|remove_user|delete`, `users.promote|demote` — p. ex. `WebsocketsProcessor.ts:364, 385, 409-416, 480, 506-510, 765-785, 815-852, 970-1006`.

## 5. Filtrage par droits

- Le filtrage est **par appartenance aux rooms**, pas par vérification de politique au moment de l'émission. `perform` ne fait aucun `can()` par socket destinataire.
- `cannot(user, "read", collection)` n'est utilisé que pour décider de faire quitter une room (`WebsocketsProcessor.ts:506, 562, 849, 905`).
- Les presenters sont appelés sans contexte utilisateur (`presentDocument(undefined, document)`, l. 230 ; `presentCollection(undefined, …)`, l. 362), donc payload identique pour tous les destinataires, sans `policies`.
- Pour les mises à jour sensibles, le serveur émet un message `entities` avec ids + `updatedAt` + `invalidatedPolicies` (documents, templates, l. 44-120) : le client re-fetch via l'API, qui applique les politiques (`app/components/WebsocketProvider.tsx:176-277`). Pour d'autres événements (`documents.update`, `comments.*`), le payload complet est poussé (l. 221-232, 624-699).
- Le processor relit systématiquement le modèle en base (`findByPk`) plutôt que de faire confiance au payload de l'event (ex. l. 47, 78, 224, 626).
- Événements non gérés : `default: return;` — `WebsocketsProcessor.ts:1030-1032`. Un nouvel event `databases.*` n'est donc diffusé à personne tant qu'un cas n'existe pas.

## 6. Côté client

- Connexion : `io(origin, { path: "/realtime", transports: ["websocket"], withCredentials: true })` — `app/components/WebsocketProvider.tsx:915-922`.
- Les handlers sont enregistrés par familles codées en dur (`useEntityHandlers`, `useDocumentHandlers`, `useCommentHandlers`, … `registerXHandlers(currentSocket)`) — `WebsocketProvider.tsx:898-939`.
- Application aux stores : en général `store.add(payload)` (fusion), `store.remove(modelId)`, invalidation `policies.remove(id)` — ex. `documents.update` l. 293-303, `comments.*` l. 631-680.
- `Store.add` : si le modèle existe, `existingModel.updateData(item)` (fusion partielle, persistance IndexedDB éventuelle) ; sinon instanciation et `fetchAllResults.clear()` — `app/stores/base/Store.ts:247-273`. Un patch partiel `PartialExcept<T, "id">` est donc déjà le mode nominal.
- `WebsocketContext` est exporté (`WebsocketProvider.tsx:56`) et fourni (l. 968-970) mais aucun autre fichier de `app/` ou `plugins/` ne le consomme aujourd'hui.
- Types de payload partagés côté client : `app/types.ts:258-274` (`WebsocketEntitiesEvent`, `WebsocketEntityDeletedEvent`, …).
- Reconnexion : la connexion est recréée quand la page redevient visible (`WebsocketProvider.tsx:943-951`) ; il n'y a pas de rejeu des événements manqués (pas de numéro de séquence dans le flux).

## 7. Extensibilité par un plugin

### Serveur — ce qui existe

- Hooks serveur : `API, AuthProvider, EmailTemplate, IssueProvider, Processor, SearchProvider, MentionProvider, Task, UnfurlProvider, Uninstall, GroupSyncProvider` — `server/utils/PluginManager.ts:35-47`. **Aucun hook websocket.**
- Un plugin peut enregistrer un `Hook.Processor` (ex. `plugins/webhooks/server/index.ts:8-26`), exécuté dans le worker, sans accès à `io`.
- Le type `Event` est une union fermée dans le core — `server/types.ts:508-539` (`BaseEvent`, l. 112-121). `BaseProcessor.applicableEvents` est typé `(Event["name"] | "*")[]` (`BaseProcessor.ts:9`) : un nom `databases.*` hors union ne compile pas sans élargir le type.
- Les modèles Sequelize sont chargés depuis `server/models` (`import * as models from "../models"`, `server/storage/database.ts:19, 171, 399`) ; aucun plugin n'a de répertoire `server/models` (le modèle du plugin webhooks vit dans le core : `server/models/WebhookSubscription.ts`).
- Avec un modèle de table `databases` / `database_rows` utilisant les helpers `*WithCtx`, les events `databases.create|update|delete`, `database_rows.create|update|delete` sont produits automatiquement ; `move` s'obtient via `eventOpts.name: "move"`, et le patch via `eventOpts.data` ou `changes` (§2).
- Effet de bord : `WebhookProcessor` écoute `"*"` (`plugins/webhooks/server/processors/WebhookProcessor.ts:8`) ; un abonnement `*` ou `databases` matche (`server/models/WebhookSubscription.ts:98-110`) et `DeliverWebhookTask` tombe sur `assertUnreachable`, qui ne fait que logguer un warning (`plugins/webhooks/server/tasks/DeliverWebhookTask.ts:79-81, 328-329`).

### Client — ce qui existe

- Hooks client : `Settings, Imports, Icon` seulement — `app/utils/PluginManager.ts:14-18`.
- `RootStore.registerStore` est `private` (`app/stores/RootStore.ts:184-194`) ; les stores sont déclarés en dur (l. 78 et suiv.).

### Options constatées (non tranchées)

Serveur, diffusion :

1. **Ajouter des `case` dans `WebsocketsProcessor`** du core (`databases.*`, `database_rows.*`) en réutilisant `getCollectionEventChannels`/`getDocumentEventChannels` ; élargir l'union `Event` dans `server/types.ts`. Modifie le core.
2. **Nouveau hook `PluginManager` (ex. `Hook.WebsocketHandler`)** appelé depuis le `default:` de `WebsocketsProcessor` avec `(event, io)`. Garde la logique dans le plugin ; ajoute un point d'extension au core.
3. **Processor de plugin dans le worker + émetteur Redis** (`@socket.io/redis-emitter`, compatible avec l'adapter Redis en place pour cibler les rooms). Nouvelle dépendance ; contourne `websocketQueue`.
4. **Message générique `entities`** (ids + `updatedAt` + `invalidatedPolicies`) et re-fetch API côté client : réutilise le modèle de filtrage le plus sûr (politiques appliquées par l'API) au prix d'un aller-retour HTTP par événement.

Serveur, émission de l'event :

- a. Modèles avec `*WithCtx` → noms automatiques `<table>.<verbe>`.
- b. `Event.createFromContext` / `Event.schedule` explicites (nécessaire pour un `UPDATE` SQL atomique hors hooks ; `schedule` = non persisté, pas d'audit).
- c. `persist: false` dans `eventOpts` pour des événements volatils (ex. présence §10.2).

Granularité de room (la spec vise « room de la collection ») :

- `collection-<id>` + `team-<id>.members` via `getCollectionEventChannels` ; ou `getDocumentEventChannels` du document porteur de la base pour inclure les memberships directs ; ou nouvelle room dédiée (ex. `database-<id>`), qui demanderait une gestion join/leave inexistante aujourd'hui (aucune room par document actuellement).

Client :

- i. Ajouter une famille `useDatabaseHandlers` dans `WebsocketProvider.tsx` et les stores dans `RootStore` (core).
- ii. Consommer `WebsocketContext` depuis les composants du plugin (`useContext(WebsocketContext)` + `socket.on/off` dans un effet) avec des stores propres au plugin.
- iii. Nouveau hook client (ex. `Hook.Websocket` / `Hook.Store`) dans `app/utils/PluginManager.ts`.
