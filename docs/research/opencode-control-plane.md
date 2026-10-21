# Cartographie du plan de contrôle OpenCode (v2.1.2)

> Recherche pour le ticket [#3](https://github.com/COOPER2703/dev-auto/issues/3) — prérequis de la spec du harness vocal.
> Date : 2026-10-21. Périmètre : OpenCode **v2.1.2** installé localement, source publique `anomalyco/opencode` branche `dev`, doc V2.

## 0. Résumé exécutif

Le plan de contrôle V2 est un **serveur HTTP local unique** (un processus pour toutes les sessions, tous projets et tous worktrees confondus) exposant :

- des **sessions durables** identifiées par `ses_…`, rattachées à un `projectID` et à un répertoire (`location.directory`) ;
- une **inbox durable** par session (`steer` / `queue`) qui découple l'admission d'un prompt de son exécution ;
- un **flux SSE global** `/api/event` (tous projets, toutes sessions, + événements plugin) ;
- des **primitives d'attente** (`wait`), d'**interruption**, de **compaction**, de **déplacement**, de **fork** et de **suppression en cascade** ;
- deux mécanismes d'« entrée attendue » : **permissions** (`permission.asked`) et **formulaires** (`form.created`), ce dernier étant celui qu'utilise l'outil `question` de l'agent — donc le point d'ancrage naturel du grilling vocal.

Trois constats structurants pour le harness :

1. **Les `/…` ne sont pas interprétés par le serveur.** Le parsing du slash est fait par le TUI (`packages/tui/src/prompt/parse.ts`) ; en HTTP il faut appeler `session.command` (commande déclarée) ou activer un **skill** (`session.skill` / attachement `skills`). Dans cet environnement, `grill-with-docs`, `to-spec`, `to-tickets`, `implement`, `wayfinder` sont des **skills**, pas des commandes.
2. **`session.wait` bloque aussi quand la session attend une permission ou un formulaire.** Il ne suffit donc pas à détecter la fin d'un tour : il faut courir `wait` contre les événements `permission.asked` / `form.created` (ou polling).
3. **Le flux d'événements est volatile** : pas de rejeu à la reconnexion, pas d'`id:` SSE, consommateur lent = flux coupé. La vérité terrain après reconnexion est `message.list` / `session.context` + les listes de permissions/formulaires en attente.

---

## 1. Méthode et sources

| Réf. | Source | Détail |
|---|---|---|
| **[S1]** | `GET /api/info` sur le serveur local | `{"version":"2.1.2","pid":12183,"urls":["http://127.0.0.1:49374"],"paths":{"tmp":"/tmp/opencode"},"capabilities":{"persistentPty":true}}` (relevé 2026-10-21) |
| **[S2]** | OpenAPI locale `GET /openapi.json` | Contrat de la version installée. 139+ opérations, `openapi 3.1.0`, sha256 `b9392083c8a0f69043c8f618f32c8ce7a36bf1c23792498813150ff552dc9cfd` |
| **[S3]** | Source `anomalyco/opencode` @ `863ea413649484b5761cd0c5c0e54c559a5c8b52` (`dev`, 2026-10-21T13:34:23-05:00) | `packages/protocol/src/groups/*.ts`, `packages/schema/src/*.ts`, `packages/client/src/**`, `packages/core/src/**`, `packages/tui/src/prompt/parse.ts`, `V2_HTTP_API_AUDIT.md` |
| **[S4]** | Doc V2 | <https://opencode.ai/v2/docs/> (consultée 2026-10-21) |
| **[S5]** | CLI locale | `opencode --help`, `opencode serve --help`, `opencode session --help`, `opencode service status` (v2.1.2) |
| **[S6]** | npm | `@opencode/client@2.1.2` et `@opencode/sdk@2.1.2` publiés (`registry.npmjs.org`) |
| **[P1]** | **Probe live** — session `ses_ea00f2628ffeCYHQuPMf3X6yOo` | prompt trivial + modèle par défaut gratuit, tour réussi (PONG) |
| **[P2]** | **Probe live** — session `ses_ea00d2447ffeP8pPf5n84rPpu7` | permission `shell` demandée puis approuvée, outil exécuté, tour réussi |
| **[P3]** | **Probe live** — sessions jetables | formulaire créé/répondu, skill activé, `queue` vs `steer`, `move`, `wait` bloqué, `@opencode/client` |

Les probes ont été faites sur le serveur de fond existant (`127.0.0.1:49374`), avec des sessions créées dans `/tmp/opencode/research-opencode-control-plane` puis **supprimées**. Aucune tâche longue n'a été lancée.

> ⚠️ La source `dev` est en avance sur 2.1.2 (ex. elle contient `GET /api/experimental/voice/models`, absent de l'OpenAPI locale [S2]). En cas de divergence, **l'OpenAPI locale [S2] fait foi**.

---

## 2. Modèle mental

```
Serveur OpenCode (1 processus)
├── Projects (id opaque stable, ex. 2772b1… ; "global" pour le projet global)
│   ├── /home/cooper/Documents/perso/auto-dev          (projectID 2772b1…)
│   └── worktrees : /home/.../auto-dev, /tmp/opencode/research-…  [P3]
├── Sessions (durables, DB ~/.local/share/opencode/opencode.db)
│   ├── session racine  (parentID absent)
│   └── session fille   (parentID = ses_…)  ← fork ou subagent
├── Inbox par session : items user/synthetic/compaction/move, delivery steer|queue
└── Bus d'événements (durable + ephemeral) → GET /api/event (SSE global)
```

- Une **location** est un couple répertoire + projet ; `GET /api/location?location[directory]=…` renvoie `{directory, project}` [S2][P3].
- Un **projet** a un identifiant opaque stable (`Project.ID`, ou `"global"`), un `canonical` et éventuellement `name`/`vcs` ; un worktree est un répertoire du même projet (`subpath` éventuel) [S2 `Project`, `Worktree.Directory`][P3].
- Les sessions sont **persistées sur disque** : on peut reprendre une session ancienne en la réadressant par son `sessionID` [S2 `session.list`/`session.get`][S5 `opencode session list`].

### Vocabulaire d'états

| Concept | Source de vérité | Remarque |
|---|---|---|
| Session en cours d'exécution | `GET /api/session/active` → `{ [sessionID]: {type:"running"} }` | Global au processus, pas filtré par projet [S2][P1] |
| Dernier résultat d'un tour | `Session.Info.outcome` ∈ `succeeded\|failed\|interrupted`, `time.idle` | `session.get` [S2] |
| Message de fin de tour | message `type:"idle"` avec `outcome` et `error?` | `message.list` [S2][P1] |
| Attente d'une permission | `permission.asked` / `GET /session/{id}/permission` | [S2][P2] |
| Attente d'une réponse humaine | `form.created` / `GET /session/{id}/form` (state `pending`) | [S2][P3] |
| Travail en file | `GET /session/{id}/inbox` | items non encore délivrés [S2][P3] |

---

## 3. Démarrer / découvrir le serveur

- Service de fond : `opencode service status` affiche l'URL (ici `http://127.0.0.1:49374`) [S5][P1].
- Démarrage dédié : `opencode serve --hostname … --port …` (aussi `--stdio`, `--service`, `--cors`) [S5].
- **Authentification** : le serveur local exige Basic auth. Utilisateur `opencode`, mot de passe lu dans `~/.config/opencode/service.json` (clé `password`) [S3 `packages/server/src/auth.ts` + `middleware/authorization.ts`][P1]. `opencode api …` injecte ce header automatiquement ; `curl` nu renvoie `401 {"_tag":"UnauthorizedError"}` [P1].
- Les liens `opencode pair` / `GET /api/pair` servent au web (cookie de session, code à usage unique, 5 min) — inutile pour un harness local same-host [S4][S2].

```ts
import { OpenCode } from "@opencode/client"
import { readFileSync } from "node:fs"

const { password } = JSON.parse(
  readFileSync(`${process.env.HOME}/.config/opencode/service.json`, "utf8"),
)
const client = OpenCode.make({
  baseUrl: "http://127.0.0.1:49374",
  headers: { authorization: `Basic ${Buffer.from(`opencode:${password}`).toString("base64")}` },
})
```

`@opencode/client` est publié en 2.1.2 et exporte un client Promise sans dépendance à Effect (`import { OpenCode } from "@opencode/client"`) [S6][S3 `packages/client/README.md`]. Un client Effect existe aussi (`@opencode/client/effect`) et un host in-process (`@opencode/sdk`), mais le client Promise suffit pour un harness externe.

> ⚠️ Piège CLI : `opencode api get /api/location --param 'location[directory]=…'` n'a **pas** transmis le paramètre profond dans notre test (réponse sur `/home/cooper`) [P3]. Utiliser le client TS ou une query encodée `location%5Bdirectory%5D=…`.

---

## 4. Les opérations clés

### 4.1 Lister les projets, worktrees et sessions récentes

```ts
const projects = await client.project.list()                    // Project[]
const worktrees = await client.worktree.list({ projectID })     // { directory, strategy? }[]
const recent = await client.session.list({
  project: projectID,        // ou directory
  parentID: null,            // null = racines seulement ; omis = racines + filles
  order: "desc",
  limit: 20,
})
// recent = { data: Session.Info[], cursor: { previous?, next? } }
```

| Besoin | HTTP | Notes |
|---|---|---|
| Projets connus | `GET /api/project` | `archived=false` par défaut ; `Project.name` optionnel → dériver le libellé du basename de `canonical` [S2][P3] |
| Location → projet | `GET /api/location?location[directory]=<abs>` | deepObject encodé [S2][P3] |
| Worktrees d'un projet | `GET /api/worktree?projectID=…` | inventaire sauvegardé, pas de découverte [S2][P3] |
| Créer un worktree | `POST /api/worktree` `{projectID, from?, branch?, directory?, name?}` | crée puis lance le setup script du projet [S2] |
| Sessions récentes | `GET /api/session?limit=&order=&project=&directory=&parentID=&archived=&search=&cursor=` | défaut : 50 plus récentes [S2] |
| Sessions filles | `GET /api/session?parentID=ses_…` | [S2] |
| Session active | `GET /api/session/active` | global processus [S2][P1] |

### 4.2 Créer une session dans un projet / worktree

```ts
const session = await client.session.create({
  location: { directory: "/tmp/opencode/research-opencode-control-plane" },
  title: "Ticket #3 — implémentation",
  // parentID, agent, model, metadata, permissions optionnels
})
// session.id = "ses_…", session.projectID, session.location.directory
```

- `location` est optionnel mais **toujours le passer** pour un harness multi-projets [S2].
- `parentID` crée une session fille à la location du parent [S2].
- `agent` par défaut = `build` (observé dans `session.step.started`) ; `model` par défaut = `GET /api/model/default` (`{providerID:"opencode", id:"flint-40b-free"}`, coût 0) [P1][S2].
- `permissions` permet de fixer une **ruleset par session** (`[{action, resource, effect:"allow"|"deny"|"ask"}]`) — clé pour l'autonomie. Les `action` sont les noms d'outils (`shell`, `edit` pour write/edit/patch, `webfetch`, `skill`, …) [S2][S3 `packages/core/src/tool/plugin/*.ts`][P2].

### 4.3 Envoyer un prompt

```ts
const item = await client.session.prompt({
  sessionID: session.id,
  text: "Implémente le ticket #3",
  skills: [{ id: "implement" }],   // attache le contenu du skill au message user
  delivery: "steer",               // défaut ; ou "queue"
  // resume: false                 // admet sans exécuter (utile pour préparer)
})
// item = Session.Inbox.User { id, sessionID, time, type:"user", payload, delivery }
```

- Réponse **immédiate** : admission durable + planification de l'exécution ; ce n'est pas la réponse de l'agent [S2].
- `delivery:"steer"` (défaut) injecte au prochain boundary d'étape et réveille l'exécution ; `"queue"` attend la fin du tour [S2][P3].
- `resume:false` laisse l'item dans l'inbox sans l'exécuter [S2][P3] — vérifié : l'item n'apparaît pas encore dans `message.list`.
- 409 `ConflictError` possible selon l'état de la session [S2].

### 4.4 Exécuter une commande slash

```ts
const { data: commands } = await client.command.list({ location: { directory } })
// [{ name:"init", description:"…" }, { name:"review", … }]

if (commands.some((c) => c.name === "implement")) {
  await client.session.command({ sessionID: session.id, name: "implement", text: "#3" })
}
```

- Le **serveur ne parse pas** `/implement` dans `prompt.text`. Le TUI découpe le slash et appelle `client.session.command` avec `name` sans slash [S3 `packages/tui/src/prompt/parse.ts` + `packages/tui/src/mini/stream-v2.transport.ts`].
- Les commandes viennent de `.opencode/command(s)/**/*.md` (+ built-ins) ; `command.list` est **location-scoped** [S3 `packages/core/src/config/plugin/command.ts`][S2].
- Une commande avec `subagent`/`subtask` (ou agent `mode:"subagent"`) crée une **session fille** (`parentID`) et la lance en arrière-plan [S3 même fichier].
- Erreurs : 404 `CommandNotFoundError`, 500 `CommandExecutionError` [S2].

### 4.5 Activer un skill (le vrai équivalent de `/implement` ici)

```ts
const { data: skills } = await client.skill.list({ location: { directory } })
// ids observés : grill-with-docs, to-spec, to-tickets, implement, wayfinder, research, tdd, …

// (a) activation seule — ajoute un message `skill` et relance l'exécution
await client.session.skill({ sessionID: session.id, id: "implement", resume: true })

// (b) préférable pour porter une consigne : prompt + attachement skill
await client.session.prompt({
  sessionID: session.id,
  text: "Implémente le ticket #3",
  skills: [{ id: "implement" }],
})
```

- `skill.list` renvoie `{ location, data: Skill.Info[] }` avec `id`, `name`, `description`, `path`, `content` [S2][P3].
- `session.skill` est **expérimental** ; `resume:false` permet d'attacher sans exécuter [S2][S3 `V2_HTTP_API_AUDIT.md`][P3].
- L'attachement `skills` est matérialisé dans le message user sous la forme `<skill_content name="implement">…</skill_content>` [P3].
- L'audit V2 acte la suppression du comportement « skill = slash command » : les skills ne sont plus des commandes [S3 `V2_HTTP_API_AUDIT.md` §019].

> **Conséquence pour la carte vocale** : la table de correspondance du harness doit être `intent → skill id`, pas `intent → /commande`. Les cinq étapes du processus (`grill-with-docs`, `to-spec`, `to-tickets`, `implement`, + `wayfinder`) existent toutes comme skills dans cet environnement [P3].

### 4.6 Suivre les événements

```ts
const controller = new AbortController()
for await (const event of client.event.subscribe({ signal: controller.signal })) {
  if (event.type === "session.text.delta" && event.data.sessionID === session.id) {
    process.stdout.write(event.data.delta)     // streaming live
  }
  if (event.type === "session.execution.succeeded" && event.data.sessionID === session.id) break
}
```

- `GET /api/event` est un flux **SSE global** : toutes locations, toutes sessions, plus les événements plugin (`rpc.*`) [S2][S3 `packages/protocol/src/groups/event.ts`].
- Trame observée : `data: {json}\n\n`, plus des commentaires `: heartbeat` ; **pas de ligne `event:` ni `id:`** [P1].
- Enveloppe : `{ id, created, type, location?: {directory}, data, durable?: {aggregateID, seq, version} }` [P1][S2].
- Le client `@opencode/client` multiplexe le flux et bufferise jusqu'à **4096** événements par abonné, puis termine l'abonnement en erreur [S3 `packages/client/src/shared-events.ts`].
- Le serveur documente : « *Volatile by contract: a slow consumer overflows and fails the stream, and events during disconnection are missed* » [S2 `event.subscribe`].

**Catalogue utile** (les événements de session ci-dessous portent `data.sessionID`) :

| Type | Durabilité | Charge utile utile |
|---|---|---|
| `session.execution.started` | durable | — (début de tour) |
| `session.execution.succeeded` | durable | — (fin de tour OK) |
| `session.execution.failed` | durable | `error: {type, message, status?, response?, retryAfterMs?}` |
| `session.execution.interrupted` | durable | `reason: "user"\|"shutdown"\|"superseded"\|"inactivity"` |
| `session.step.started` | durable | `agent`, `model`, `assistantMessageID`, `snapshot?` |
| `session.text.started/delta/ended` | delta éphémère ; ended durable | `delta` (live), `text` (valeur complète rejouable) |
| `session.reasoning.*` | idem | raisonnement du modèle (à ne pas lire en voiture) |
| `session.tool.input.*` | delta éphémère ; ended durable | `name`, `text` (JSON d'entrée) |
| `session.tool.called` | durable | `id`, `input`, `executed` |
| `session.tool.progress` | éphémère | `metadata` |
| `session.tool.success` | durable | `content[]`, `metadata`, `executed` |
| `session.tool.failed` | durable | `error`, `content?` |
| `session.retry.scheduled` | durable | `attempt`, `at`, `error` (backoff provider) |
| `session.compaction.started/ended/failed` | durable | `reason`, `text`, coût/tokens |
| `permission.asked` / `permission.replied` | éphémère | requête / décision |
| `form.created` / `form.replied` / `form.cancelled` | éphémère | `form`, `answer`, `id` |
| `session.inbox.enqueued/delivered/cancelled` | durable | `inboxID`, `item` |
| `session.skill.activated` | durable | `id`, `name`, `text` |
| `session.shell.started/ended` | durable | `shell`, `output` (commandes shell de session) |
| `session.moved` | durable | **`location` = ancienne**, `data.location` = nouvelle, `projectID` |
| `session.created/renamed/archived/metadata.updated/permissions` | durable | métadonnées |
| `server.connected` | — | émis à l'ouverture du flux |
| `rpc.*` | — | événements plugins |

Champs `session.tool.called` observés en live : `{assistantMessageID, id, input:{command:"echo BONJOUR"}, executed:false}` ; `session.tool.success` : `{content:[{type:"text",text:"BONJOUR\n"}], metadata:{status:"completed",truncated:false,exit:0}, executed:false}` [P2].

> ⚠️ **`session.status` et `session.idle` existent dans le manifeste** (compatibilité TUI) mais **n'ont jamais été émis** dans nos trois sessions / un tour complet [S3 `packages/schema/src/session-status-event.ts`][P1][P2]. Ne pas bâtir la détection d'inactivité dessus : utiliser `session.execution.*`, le message `idle`, `session.get.outcome` ou `wait`.

### 4.7 Attendre la fin d'un tour

```ts
await client.session.wait({ sessionID: session.id })   // résout quand la boucle agent est idle
```

- `POST /api/experimental/session/{id}/wait` → `204` quand idle, `404` session inconnue, `503` service indisponible [S2].
- Qualifié de « **race-free idle barrier** » dans l'audit V2 [S3 `V2_HTTP_API_AUDIT.md` §065].
- Vérifié : `204` immédiat sur session déjà idle ; ~10 s et ~82 s sur deux tours réussis ; ~93 s avant un échec provider ; **> 120 s (timeout client) quand une permission était en attente** [P1][P2][P3].
- **Donc** : `wait` ne distingue pas « réfléchit » de « attend un humain ». Toujours l'utiliser en course avec la détection d'entrée (§5).

### 4.8 Interrompre

```ts
const { interrupted } = await client.session.interrupt({ sessionID })
// interrupted: true si une exécution active a été interrompue, false si déjà idle
await client.session.interrupt({ sessionID, resume: true }) // reprend les inputs de steering en attente
```

- Émet `session.execution.interrupted` avec `reason:"user"` [P2].
- N'interrompt que l'exécution **possédée par ce processus** ; les prompts en `queue` restent parqués [S2].
- `resume:true` reprend le steering et les contrôles en attente (compaction, move), pas les prompts `queue` [S2].

### 4.9 Compacter

```ts
const item = await client.session.compact({ sessionID: session.id })
// item = Session.Inbox.Compaction
```

- Admission durable ; **steer par défaut** : s'exécute au prochain boundary d'étape, devant les prompts en file [S2].
- Événements `session.compaction.started` → `delta` (éphémère) → `ended` (résumé complet) ou `failed` [S2][S3].
- La compaction crée un message `type:"compaction"` ; `session.context` ne renvoie que les messages **après** la dernière compaction [S2].

### 4.10 Déplacer une session

```ts
await client.session.move({
  sessionID: session.id,
  directory: "/autre/projet/ou/worktree",
  delivery: "steer",     // défaut
})
```

- Change la location **et le projet** si le répertoire appartient à un autre projet. Vérifié : `projectID` est passé de `2772b1…` à `3c45a4…`, et `session.moved` porte l'ancienne location dans `location` et la nouvelle dans `data.location` [P2].
- Un `move` en cours d'exécution est un item d'inbox (`type:"move"`), livré au boundary demandé [S2].

### 4.11 Supprimer / fork / exporter

| Opération | Appel | Effet |
|---|---|---|
| Supprimer | `client.session.remove({sessionID})` | supprime **aussi les sessions filles** [S2] |
| Fork | `client.session.fork({sessionID, before?})` | enfant avec historique copié jusqu'à `before` [S2] |
| Exporter | `GET /api/experimental/session/{id}/export` | transcript projeté complet [S2] |
| Importer | `POST /api/experimental/session/import` | importe parent avant enfants [S2] |
| Contexte actif | `GET /api/session/{id}/context` | messages après la dernière compaction [S2] |
| Diff d'un tour | `GET /api/session/{id}/diff?from=&to=&context=` | `{file, patch, additions, deletions, status}` [S2] |
| Génération éphémère | `POST /api/session/{id}/generate` `{prompt}` | texte transitoire, **ne mute pas** l'historique — idéal pour un résumé vocal [S2] |

---

## 5. Détecter qu'une session attend une entrée

C'est **le** point critique pour un conductor vocal : un agent qui pose une question ou demande une permission ne produit ni texte final ni fin de tour.

### 5.1 Permissions

Événement (éphémère) :

```json
{
  "type": "permission.asked",
  "data": {
    "id": "per_…",
    "sessionID": "ses_…",
    "action": "shell",
    "resources": ["echo BONJOUR"],
    "save": ["echo *"],
    "source": { "type": "tool", "messageID": "msg_…", "id": "call_…" }
  }
}
```

Réponse :

```ts
await client.permission.reply({
  sessionID,
  requestID: "per_…",
  decision: "once",          // "once" | "always" | "reject"
  // message: "refusé : commande destructrice"   // feedback au modèle
})
```

- Le champ de requête s'appelle **`decision`**, pas `reply` (l'événement `permission.replied`, lui, contient `reply`) [S2][S3 `V2_HTTP_API_AUDIT.md` §100][P2].
- Listes faisant autorité (à poller après reconnexion) :
  - `GET /api/session/{id}/permission` → permissions en attente de la session ;
  - `GET /api/permission/request?location[directory]=…` → toutes les permissions en attente d'une location [S2].
- Une permission non répondue **bloque la boucle** : `wait` ne rend pas la main [P2].
- Pour l'autonomie : règles par session (`permissions` à la création) ou `opencode --auto` (auto-approve tout ce qui n'est pas explicitement `deny`) [S5]. Sans ça, `/implement` déclenchera des demandes à répétition.

### 5.2 Formulaires (questions de l'agent)

L'outil `question` de l'agent crée un **Form** et attend la réponse ; c'est le canal des questions de grilling [S3 `packages/core/src/tool/plugin/question.ts`].

Événement :

```json
{
  "type": "form.created",
  "data": {
    "form": {
      "id": "frm_…",
      "sessionID": "ses_…",
      "title": "Questions",
      "metadata": { "kind": "question" },
      "fields": [
        {
          "key": "q0",
          "title": "Couleur",
          "description": "Quelle couleur préfères-tu ?",
          "type": "string",
          "options": [
            { "value": "Rouge", "label": "Rouge" },
            { "value": "Bleu", "label": "Bleu" }
          ],
          "custom": true
        }
      ]
    }
  }
}
```

Réponse :

```ts
await client.session.form.reply({
  sessionID,
  formID: "frm_…",
  answer: { q0: "Rouge" },       // clé = field.key ; valeur string|number|boolean|string[]
})
// ou, pour un client non interactif :
await client.session.form.cancel({ sessionID, formID, message: "Pas de réponse vocale" })
```

- `GET /api/session/{id}/form/{formID}` renvoie `state: {status:"pending"}` puis `{status:"answered", answer}` ou `{status:"cancelled", message?}` [S2][P3].
- L'outil `question` mappe chaque question sur un champ `q0…qn` : `type:"string"` avec `options` + `custom:true` (réponse libre autorisée), ou `type:"multiselect"` si `multiple:true` [S3 `question.ts`].- Une fois répondu, l'outil reprend **automatiquement** : pas besoin de renvoyer un prompt [S3 `Form.ask` / `question.ts`].
- Annuler sans message tue l'outil (`CancelledError`) ; annuler **avec** message renvoie le message au modèle qui continue [S3 `question.ts`].
- Formulaires en attente : `GET /api/session/{id}/form` ou `GET /api/form?location[directory]=…` [S2].

> **Niveau de preuve** : le cycle `form.created` → `state:"pending"` → `reply` → `form.replied` a été vérifié en live via l'API [P3]. La forme exacte produite par l'outil `question` (clés `q0…`, `custom:true`, options) est établie par la source [S3 `question.ts`] ; le modèle gratuit n'a pas émis de question pendant la fenêtre de probe (endpoint provider en 503).

### 5.3 Algorithme de détection recommandé

```ts
type InputNeeded =
  | { kind: "permission"; request: PermissionRequest }
  | { kind: "form"; form: FormInfo }

async function waitForTurn(sessionID: string): Promise<"done" | "failed" | "interrupted"> {
  const controller = new AbortController()
  const events = client.event.subscribe({ signal: controller.signal })

  const input = (async (): Promise<InputNeeded> => {
    for await (const e of events) {
      if (e.type === "permission.asked" && e.data.sessionID === sessionID)
        return { kind: "permission", request: e.data }
      if (e.type === "form.created" && e.data.form.sessionID === sessionID)
        return { kind: "form", form: e.data.form }
    }
    throw new Error("stream closed")
  })()

  const done = client.session.wait({ sessionID }).then(() => "done" as const)
  const result = await Promise.race([done, input])
  controller.abort()

  if (result === "done") {
    const s = await client.session.get({ sessionID })
    return s.outcome ?? "done"
  }
  // parler la question/permission, attendre la réponse, puis relancer waitForTurn
  return handleInput(result)   // à implémenter : TTS + STT + reply
}
```

Points d'implémentation :

1. **S'abonner avant** de prompter (sinon on peut manquer `permission.asked`, éphémère). En pratique, garder **un seul abonnement de longue durée** par session et un registre d'état, plutôt que se ré-abonner à chaque tour.
2. Après reconnexion : ne pas se fier aux événements manqués ; **poller** `permission.list` + `session.form.list` + `session.active` + `message.list`.
3. Un formulaire peut être créé par une elicitation MCP avec `sessionID:"global"` (sentinel temporaire) — filtrer prudemment [S3 `Form.Info`].
4. Un `retry.scheduled` n'est **pas** une attente d'entrée : c'est un backoff provider. Le dire à l'utilisateur (« le modèle réessaie »), et éventuellement attendre.

---

## 6. Récupérer le résultat

```ts
const page = await client.message.list({ sessionID, order: "asc", limit: 200 })
const assistant = page.data.filter((m) => m.type === "assistant")
const last = assistant.at(-1)
const text = last?.content.filter((c) => c.type === "text").map((c) => c.text).join("\n")
const idle = page.data.filter((m) => m.type === "idle").at(-1)
// idle.outcome = "succeeded" | "failed" | "interrupted", idle.error?
```

- `message.list` renvoie **du plus récent au plus ancien par défaut** (`desc`) — utiliser `order:"asc"` pour un transcript chronologique [S2][P1].
- Types de messages : `user`, `assistant`, `idle`, `system`, `synthetic`, `skill`, `shell`, `compaction`, `agent-switched`, `model-switched`, `location-switched` ; filtre `type=` [S2].
- Le texte complet est dans `assistant.content[]` (`text`, `reasoning`, outils) ; les deltas SSE ne sont qu'un confort de streaming [S2].
- Coût/tokens : `session.get` (`cost`, `tokens`) ou événements `session.usage.updated` [S2][P1].
- Fichiers touchés : `session.step.ended.files?` ou `GET /session/{id}/diff` [S2].

---

## 7. Séquence complète : créer → lancer `/implement` → suivre → récupérer

```ts
import { OpenCode } from "@opencode/client"

const client = OpenCode.make({ baseUrl, headers })
const directory = "/tmp/opencode/research-opencode-control-plane"

// 1. Résoudre le projet et le worktree
const projects = await client.project.list()
const project = projects.find((p) => directory.startsWith(p.canonical))!
await client.worktree.refresh({ projectID: project.id })          // optionnel
const worktrees = await client.worktree.list({ projectID: project.id })

// 2. Créer la session (permissions adaptées à l'autonomie)
const session = await client.session.create({
  location: { directory },
  title: "Ticket #3 — implémentation",
  permissions: [
    { action: "edit",  resource: "*", effect: "allow" },
    { action: "shell", resource: "*", effect: "ask" },   // ou allow si --auto
  ],
})

// 3. Démarrer la consommation AVANT de lancer : le flux ne démarre qu'au premier next()
const controller = new AbortController()
const followed = (async () => {
  for await (const e of client.event.subscribe({ signal: controller.signal })) {
    const id = (e.data as any)?.sessionID
    if (id !== session.id) continue
    switch (e.type) {
      case "session.text.delta":
        speakStream(e.data.delta)          // pousser dans une file, ne pas bloquer ici
        break
      case "session.tool.called":
        say("Appel d'outil")
        break
      case "permission.asked":
        await askVoiceAndReplyPermission(e.data)
        break
      case "form.created":
        await askVoiceAndReplyForm(e.data.form)
        break
      case "session.retry.scheduled":
        say(`Le modèle réessaie (tentative ${e.data.attempt})`)
        break
      case "session.execution.succeeded":
      case "session.execution.failed":
      case "session.execution.interrupted":
        return
    }
  }
})()

// 4. Lancer l'implémentation : skill (recommandé ici) ou commande si elle existe
const { data: commands } = await client.command.list({ location: { directory } })
if (commands.some((c) => c.name === "implement")) {
  await client.session.command({ sessionID: session.id, name: "implement", text: "#3" })
} else {
  await client.session.prompt({
    sessionID: session.id,
    text: "Implémente le ticket #3. Lis le ticket, annonce son titre, puis travaille sur la branche courante.",
    skills: [{ id: "implement" }],
  })
}

// 5. Attendre la fin du tour
await followed
controller.abort()

// 6. Résultat
const page = await client.message.list({ sessionID: session.id, order: "asc" })
const last = page.data.filter((m) => m.type === "assistant").at(-1)
const text = last?.content.filter((c) => c.type === "text").map((c) => c.text).join("\n")
const diff = await client.session.diff({ sessionID: session.id })
say(text ?? "Pas de réponse texte")
say(`${diff.data.length} fichier(s) modifié(s)`)
```

Variante « grilling » : même squelette avec `skills:[{id:"grill-with-docs"}]` et un texte décrivant l'idée ; les questions arrivent en `form.created` (outil `question`), on répond par `form.reply`, puis on enchaîne `to-spec` / `to-tickets` dans la **même session** pour conserver le contexte.

> Le `await` du dialogue vocal dans la boucle suspend la consommation des événements pendant l'échange. En production, séparer une boucle « ingestion d'événements → file » d'une boucle « dialogue vocal » pour ne pas risquer l'overflow du flux.

---

## 8. Pièges (vérifiés ou documentés)

### 8.1 Streaming

- **Flux global** : `/api/event` mélange toutes les sessions, tous les projets, et des événements globaux (`shell.*`, `project.updated`, `worktree.*`) sans `sessionID`. **Filtrer par `data.sessionID`** et router par `location.directory` [P1].
- **Éphémère vs durable** : `*.delta` (texte, raisonnement, input d'outil) et `permission.*`, `form.*` sont éphémères ; la valeur finale rejouable est `*.ended`, `tool.success/failed`, etc. [S3 `packages/schema/src/event.ts`].
- **Consommateur lent** : le serveur coupe le flux ; le client bufferise 4096 événements puis erreur. Ne pas faire de TTS synchrone dans la boucle d'événements : pousser dans une file [S2][S3 `shared-events.ts`].
- **Heartbeats** : lignes `: heartbeat` à ignorer ; le client expose `onActivity` pour un watchdog [S3 `shared-events.ts`][P1].

### 8.2 Reconnexion

- Aucun `id:` SSE, aucun paramètre `after` sur `/api/event` : **pas de reprise automatique** ; « events during disconnection are missed » [S2][P1].
- Après reconnexion : relire `message.list` (ou `session.context`), `session.get`, `session.active`, `permission.list`, `session.form.list`, `session.inbox.list`.
- `GET /api/experimental/session/{id}/log?after=N&follow=true` est censé rejouer les événements durables. **Dans nos probes 2.1.2, il n'a émis que le marqueur `log.synced`** (seq 0/1/3, sans les événements) [P3]. À considérer comme expérimental/non fiable ; ne pas en dépendre pour reconstruire l'historique.

### 8.3 Concurrence

- Plusieurs sessions tournent **en parallèle** dans le même processus (5 simultanément observées) [P1]. `session.active` est global.
- `prompt` n'échoue pas parce qu'une session est occupée : il admet l'item. `steer` (défaut) s'injecte au prochain boundary ; `queue` reste dans l'inbox jusqu'à la fin du tour [S2][P3].
- Vérifié : un prompt `steer` est délivré, un second en `queue` reste listé dans `/inbox` [P3].
- `interrupt` n'affecte que l'exécution du processus courant ; un `queue` reste parqué après interruption [S2].
- `wait` est **par session** : il ne attend pas les sessions filles.
- `session.move` peut changer de projet ; les événements suivants portent la nouvelle location [P2].

### 8.4 Ordre des événements

- Les événements durables portent `durable.seq` par agrégat (`sessionID`) : c'est l'ordre autoritaire pour un rejeu/tri [P1][S2].
- Les deltas éphémères n'ont pas de `seq` : l'ordre est celui de livraison du flux, pas un ordre causal garanti entre types.
- Un tour : `inbox.enqueued` → `execution.started` → `instructions.updated` (si delta) → `inbox.delivered` → `step.started` → (`retry.scheduled` → `step.started`)* → `text/reasoning` → `tool.*` → `step.ended` → `usage.updated` → `execution.succeeded` [P1].
- `session.moved` : `location` = **origine**, `data.location` = destination [P2].

### 8.5 Sessions filles

- `parentID` crée une session fille à la location du parent ; `session.fork` copie l'historique ; les commandes `subagent` créent une fille et la lancent en arrière-plan [S2][S3 `config/plugin/command.ts`].
- Supprimer le parent supprime les filles [S2].
- Les événements des filles portent **leur** `sessionID` : un suivi naïf du parent les rate.
- La fin du parent ≠ fin des filles ; surveiller `session.active` ou les `execution.*` des filles.
- `session.list({parentID: null})` pour les racines, `{parentID: "ses_…"}` pour les filles [S2].

### 8.6 Modèle par défaut et erreurs provider

- Le modèle par défaut local est **`flint-40b-free`** (provider `opencode`, coût 0, outils oui, entrée texte) [S2 `model.default`][P1].
- Il est **lent et instable** : 503 « Upstream request failed: Endpoint is unavailable », jusqu'à 8 tentatives avec backoff (10 s, 4 s, 8 s, 11 s…), parfois échec final `execution.failed` [P1][P2][P3].
- Un tour réussi a pris ~10 s et ~82 s ; le probe `@opencode/client` a échoué après ~93 s (retries épuisés) ; un tour avec permission en attente a dépassé 120 s [P1][P2][P3].
- Toujours traiter `session.retry.scheduled` et `execution.failed` ; ne pas confondre silence et réflexion. Pour un usage quotidien, prévoir un modèle payant/rapide.

### 8.7 Autres

- **`wait` + input** : bloquant (voir §5).
- **`session.status` / `session.idle`** : non émis (voir §4.6).
- **`decision` vs `reply`** : la requête de permission utilise `decision` (voir §5.1).
- **`opencode api --param`** : deepObject non transmis (voir §3).
- **OpenAPI « expérimental »** : `wait`, `skill`, `log`, `export/import`, `stats` sont explicitement hors engagement stable [S3 `V2_HTTP_API_AUDIT.md`]. Prévoir une couche d'abstraction.
- **Doc V2 maigre** : la page V2 consultée est une intro (CLI/web/pair/docker) ; les URLs testées pour le client/API (`/v2/docs/sdk`) renvoient 404. Le contrat réel est l'OpenAPI générée + les packages client [S4][S2].

---

## 9. Implications pour le harness vocal (à valider dans la spec)

1. **Un seul serveur, plusieurs sessions** : le conductor peut lancer N implémentations parallèles (worktrees distincts) et n'a qu'un flux à consommer, filtré par session.
2. **La voix n'est pas le canal de l'agent** : l'agent pose ses questions via `Form` (outil `question`), ce qui est parfait : texte + options structurées → TTS, réponse STT → `form.reply`.
3. **Le grilling est déjà outillé** : `grill-with-docs` + `question` produisent une alternance question/réponse sans prompt additionnel.
4. **Permissions** : pour `/implement`, prévoir une ruleset par session (`allow` lecture/édition, `ask` shell destructif) ou `--auto`, sinon le tour se bloque.
5. **Reprise** : après coupure réseau, reconstruire depuis `message.list` + listes `permission`/`form` ; ne pas compter sur le flux.
6. **Annonces vocales** : lire les `text.delta` est trop verbeux ; préférer des jalons (`tool.called` sur `edit`/`shell`, `step.ended`, `execution.succeeded`) + `session.generate` pour un résumé parlé.
7. **Modèle** : le gratuit ne tient pas la charge d'un `/implement` ; budgéter un modèle payant pour les sessions de dev, garder le gratuit pour des essais.

---

## 10. Reproductibilité

```bash
# version + URL
opencode --version
opencode service status

# contrat
opencode api get /openapi.json > openapi.json

# auth pour curl
PW=$(jq -r .password ~/.config/opencode/service.json)
curl -s -u "opencode:$PW" "http://127.0.0.1:49374/api/model/default?location%5Bdirectory%5D=$PWD"

# session de test + prompt + wait + messages + delete
SID=$(curl -s -u "opencode:$PW" -H 'content-type: application/json' \
  -X POST http://127.0.0.1:49374/api/session \
  -d "{\"location\":{\"directory\":\"$PWD\"},\"title\":\"probe\"}" | jq -r .data.id)
curl -s -u "opencode:$PW" -H 'content-type: application/json' \
  -X POST "http://127.0.0.1:49374/api/session/$SID/prompt" \
  -d '{"text":"Réponds uniquement: PONG"}'
curl -s -o /dev/null -w '%{http_code}\n' -u "opencode:$PW" \
  -X POST "http://127.0.0.1:49374/api/experimental/session/$SID/wait"
curl -s -u "opencode:$PW" "http://127.0.0.1:49374/api/session/$SID/message?order=asc" | jq
curl -s -o /dev/null -w '%{http_code}\n' -u "opencode:$PW" \
  -X DELETE "http://127.0.0.1:49374/api/session/$SID"
```

---

## 11. Sources

- **[S1]** `GET http://127.0.0.1:49374/api/info`, OpenCode 2.1.2, 2026-10-21.
- **[S2]** `GET /openapi.json` du serveur 2.1.2 (sha256 `b9392083…`), opérations `session.*`, `event.subscribe`, `permission.*`, `form.*`, `worktree.*`, `project.*`, `model.default`.
- **[S3]** `anomalyco/opencode@863ea41` (`dev`) :
  - `packages/protocol/src/groups/session.ts`, `event.ts`, `permission.ts`, `form.ts`, `worktree.ts`, `command.ts`, `skill.ts`
  - `packages/schema/src/session-event.ts`, `session-status-event.ts`, `session-inbox.ts`, `session-message.ts`, `session-error.ts`, `form.ts`, `question.ts`, `prompt-input.ts`, `command.ts`, `worktree.ts`, `file-diff.ts`
  - `packages/client/src/promise/generated/client.ts` / `types.ts`, `packages/client/src/shared-events.ts`, `packages/client/README.md`
  - `packages/core/src/tool/plugin/question.ts`, `packages/core/src/config/plugin/command.ts`, `packages/core/src/session/prompt.ts`, `packages/core/src/bus.ts`
  - `packages/tui/src/prompt/parse.ts`, `packages/tui/src/mini/stream-v2.transport.ts`
  - `packages/server/src/auth.ts`, `packages/server/src/middleware/authorization.ts`
  - `V2_HTTP_API_AUDIT.md`
- **[S4]** <https://opencode.ai/v2/docs/> (intro CLI / web / pair / docker).
- **[S5]** `opencode --help`, `opencode serve --help`, `opencode session --help`, `opencode service status`, `opencode service get` (2.1.2).
- **[S6]** `https://registry.npmjs.org/@opencode%2Fclient`, `…/@opencode%2Fsdk` (latest 2.1.2), et installation réelle de `@opencode/client@2.1.2`.
- **[P1]** Probe session `ses_ea00f2628ffeCYHQuPMf3X6yOo` : create → prompt « Réponds uniquement par le mot PONG » → events → wait (10 s) → messages → delete.
- **[P2]** Probe session `ses_ea00d2447ffeP8pPf5n84rPpu7` : ruleset `shell:ask`, prompt forçant `echo BONJOUR`, `permission.asked`, reply `decision:"once"`, `tool.success`, wait (82 s), skill `implement` (`resume:false`), move vers `/tmp/opencode`.
- **[P3]** Probes jetables : formulaire `frm_…` créé/répondu, `prompt` avec `skills:[{id:"implement"}]` + `resume:false` (inbox), `queue` vs `steer`, `GET /log` (marqueur seul), `@opencode/client` (create/subscribe/prompt/wait/messages/delete), `command.list`/`skill.list` location-scoped.
