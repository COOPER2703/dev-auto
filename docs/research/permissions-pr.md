# Cartographie : permissions OpenCode, push et création de PR sans écran

> Ticket : [#4 — Cartographier permissions, push et création de PR](https://github.com/COOPER2703/dev-auto/issues/4)
> Branche : `research/permissions-pr` (jetable, jamais mergée)
> Date : 2026-10-21
> Versions observées : OpenCode CLI **2.1.2**, source `anomalyco/opencode` branche `dev` au commit
> `863ea41`, `gh` **2.101.0**.
>
> Méthode : documentation V2 officielle, code source, OpenAPI locale, manuel `gh`, puis tests sur un
> serveur OpenCode isolé et sur le dépôt `COOPER2703/dev-auto`. Chaque affirmation est suivie de sa
> source (`[D…]`, `[S…]`, `[G…]`), listée en fin de note.

---

## 0. Décisions courtes

1. **L'API de permission V2 est bien exposée en HTTP**, mais pas sous la forme `permission.reply`
   globale : on liste les demandes d'une session, puis on répond à **un `requestID` précis** avec
   `once`, `always` ou `reject` [S1][S2].
2. **`once` est déjà une capacité à portée limitée** : le serveur ne l'applique qu'à la demande
   portant cet identifiant. Le harness doit néanmoins ajouter ce que le serveur ne fournit pas :
   **expiration, usage unique côté harness, liaison à une confirmation orale, refus en cas de
   changement de la demande** [S3].
3. **Ne jamais utiliser `always` pour un push ou une PR.** `always` crée une règle `allow` durable et
   projet-scopée, proposée par l'outil (`save`), qui peut être plus large que l'action confirmée
   [D1][S3][S5].
4. **Le harness, pas OpenCode, doit pousser et créer les PR.** On refuse `git push` et `gh pr create`
   dans la configuration OpenCode, puis on exécute ces deux commandes dans un exécuteur allowlisté
   après confirmation orale. Cela évite de donner au modèle un shell capable de pousser [D1][S8].
5. **Pas d'intégration GitHub locale de création de PR dans OpenCode V2.** L'API `integration` V2
   gère des **credentials** (clé, OAuth, commande, externe), pas des opérations Git/PR [S9]. Le
   « GitHub OpenCode » historique est une **GitHub Action** déclenchée par commentaire, côté runner
   GitHub [S10]. Pour un harness vocal local, `gh` CLI est la voie principale.
6. **Vérification après coup** : `git ls-remote` pour la branche, `gh pr view --json …` pour l'URL et
   l'état, `gh pr checks` pour la CI (code de sortie 8 si checks en attente) [G1][G2][G3].

---

## 1. API de permission OpenCode V2

### 1.1 Endpoints exacts

Le groupe OpenAPI s'appelle `server.permission` [S1]. Les routes utiles sont :

| Méthode | Chemin | Operation ID | Rôle |
|---|---|---|---|
| `GET` | `/api/permission/request` | `permission.request.list` | Toutes les demandes en attente, toutes sessions, filtrées par `location` |
| `GET` | `/api/permission/saved` | `permission.saved.list` | Règles `allow` durables, filtrables par `projectID` |
| `PATCH` | `/api/permission/saved/:id` | `permission.saved.update` | Modifier une règle durable |
| `DELETE` | `/api/permission/saved/:id` | `permission.saved.remove` | Supprimer une règle durable |
| `POST` | `/api/session/:sessionID/permission` | `session.permission.create` | Évaluer une action et créer une demande si le résultat est `ask` |
| `GET` | `/api/session/:sessionID/permission` | `session.permission.list` | Demandes **en attente** d'une session |
| `GET` | `/api/session/:sessionID/permission/:requestID` | `session.permission.get` | Une demande précise, vérifiée comme appartenant à la session |
| `POST` | `/api/session/:sessionID/permission/:requestID/reply` | `session.permission.reply` | Répondre `once`, `always` ou `reject` |

Sources : routes dans [S1], chemins et schémas dans [S2]. Le handler vérifie bien
`request.sessionID === sessionID` avant de répondre ; sinon il renvoie `PermissionNotFoundError`
[S6].

> **Attention au libellé du ticket.** Le ticket parle de
> `/api/session/:id/permission` et de `permission.reply`. En V2 :
> - `POST /api/session/:id/permission` **crée** une demande (ce n'est pas la réponse) ;
> - la réponse est `POST /api/session/:id/permission/:requestID/reply` ;
> - `permission.reply` est le nom de la méthode SDK, pas une route globale [S1][D7].

### 1.2 Schémas

`Permission.Request` [S2][S4] :

```json
{
  "id": "per_...",
  "sessionID": "ses_...",
  "action": "shell",
  "resources": ["git push -u origin research/permissions-pr"],
  "save": ["git push *"],
  "metadata": { "anything": "optional" },
  "source": { "type": "tool", "messageID": "msg_...", "id": "call_..." },
  "message": "optional, fourni par un hook"
}
```

Champs requis : `id`, `sessionID`, `action`, `resources`. `save`, `metadata`, `source` et `message`
sont optionnels. `resources` est un **tableau** : une action composée peut en contenir plusieurs, et
la règle la plus restrictive s'applique (voir 1.4) [S2][D1].

`Permission.Reply` [S2][S4] :

```json
"once" | "always" | "reject"
```

`Permission.Effect` [S2][S4] :

```json
"allow" | "deny" | "ask"
```

`Permission.Rule` et `Permission.Ruleset` [S2][S4] :

```json
{ "action": "shell", "resource": "git status *", "effect": "allow" }
```

```json
[
  { "action": "shell", "resource": "*", "effect": "ask" },
  { "action": "shell", "resource": "git status *", "effect": "allow" }
]
```

`Permission.Source` [S2][S4] :

```json
{ "type": "tool", "messageID": "msg_...", "id": "call_..." }
```

Réponse de `POST .../permission` [S2] :

```json
{ "data": { "id": "per_...", "effect": "ask" } }
```

Si l'effet calculé est `allow` ou `deny`, **aucune demande en attente n'est créée** ; seul `ask`
crée une entrée dans la map `pending` [S3]. C'est utile au harness pour tester une politique sans
créer de demande.

Réponse de `GET .../permission` et `GET .../permission/:requestID` : `{ "data": [ ... ] }` ou
`{ "data": { ... } }` [S2]. `GET /api/permission/request` renvoie en plus un objet `location`
[S2].

Réponse de `POST .../reply` : **204 No Content**, ou 404 si la demande n'existe plus / n'appartient
pas à la session [S2][S6].

### 1.3 Événements

Deux événements natifs sont publiés sur le flux SSE `GET /api/event` [S1][S4] :

```json
{
  "type": "permission.asked",
  "data": {
    "id": "per_...",
    "sessionID": "ses_...",
    "action": "shell",
    "resources": ["..."],
    "save": ["..."],
    "metadata": {},
    "source": { "type": "tool", "messageID": "...", "id": "..." }
  }
}
```

```json
{
  "type": "permission.replied",
  "data": { "sessionID": "ses_...", "requestID": "per_...", "reply": "once" }
}
```

Le flux est **volatile** : « a slow consumer overflows and fails the stream, and events during
disconnection are missed » [S1]. Le harness doit donc, à la reconnexion, faire un
`GET /api/permission/request` pour récupérer l'état courant, et ne pas se contenter du flux.

### 1.4 Application des règles `allow` / `deny` / `ask`

La politique de base de **chaque agent**, y compris custom, est [D1] :

```jsonc
[
  { "action": "*", "resource": "*", "effect": "allow" },
  { "action": "read", "resource": "*.env", "effect": "ask" },
  { "action": "read", "resource": "*.env.*", "effect": "ask" },
  { "action": "read", "resource": "*.env.example", "effect": "allow" }
]
```

Donc, par défaut, **presque tout est autorisé**, y compris `shell`. C'est un point crucial : une
politique « vocale » doit explicitement introduire des `ask`/`deny`, sinon OpenCode n'appelle jamais
le harness.

Ordre d'évaluation, d'après le code [S3] et la doc [D1] :

1. Les règles de l'agent sont fusionnées avec celles de la session :
   `merge(agent.permissions, session.permissions)`. Les règles de session sont **ajoutées après**
   celles de l'agent.
2. Un `deny` sur **n'importe laquelle** des `resources` de l'action bloque toute l'opération.
3. Sinon, si au moins une ressource donne `ask`, l'opération est `ask`.
4. Sinon, elle est `allow`.
5. Les règles **sauvegardées** (`always`) sont ajoutées après, en `allow`, mais seulement pour les
   actions non déjà bloquées par un `deny` configuré. Le code teste d'abord `denied()` sur les
   règles configurées, puis ajoute les règles sauvegardées [S3]. **Un `deny` configuré gagne donc
   toujours sur un `always` sauvegardé.**
6. Les `policies` (`tool.use`, etc.) s'appliquent **après** ces règles et les approbations
   sauvegardées : elles ne peuvent que durcir, jamais autoriser [D4].

La règle gagnante est la **dernière qui matche** (« last matching rule wins »), pas la plus
spécifique. D'où l'ordre : règle large d'abord, exceptions ensuite [D1].

Le matching utilise des wildcards sur la **valeur entière** : `*` = zéro ou plusieurs caractères
(y compris `/`), `?` = un caractère, le reste est littéral. Un motif shell finissant par ` *`
matche aussi la commande sans argument : `git status *` matche `git status` et `git status --short`
[D1].

Portées :

- **Agent** : `agents.<id>.permissions`, ajoutées après les règles globales ; un subagent custom
  utilise ses propres permissions, pas un sous-ensemble de celles du parent [D1][D3].
- **Session** : `permissions` à la création de session, ou `ctx.session.update({ permissions })`
  côté plugin. Elles sont évaluées après celles de l'agent. Les sessions enfant héritent des règles
  en vigueur à leur création [D7].
- **Projet / global** : `permissions` dans `opencode.json(c)` ; les règles globales sont ajoutées
  après la configuration de base, les règles d'agent en dernier [D1].
- **Sauvegardées** : durables, **projet-scopées**, créées uniquement par un `always` avec un
  `save` non vide. Elles ne remplacent pas un `deny` [D1][S3][S5].

Actions et ressources intégrées [D1] :

| Action | Ressource |
|---|---|
| `read` | chemin interne ou absolu canonique |
| `edit` | chemin cible de `edit`, `write`, `patch` |
| `glob` | motif glob demandé |
| `grep` | expression régulière demandée |
| `shell` | commande produite par le scanner ; une commande composée peut en produire plusieurs |
| `subagent` | ID de l'agent cible |
| `skill` | ID du skill |
| `question` | `*` |
| `webfetch` | URL demandée |
| `websearch` | requête |
| `external_directory` | frontière de répertoire externe, souvent `…/*` |
| `<server>_<tool>` | outil MCP |
| `execute` | `*` (Code Mode ; les outils imbriqués gardent leurs règles) |

Pour `shell`, c'est le **scanner** qui produit les ressources. Le scanner par défaut est
tree-sitter ; un scanner portable expérimental existe (`experimental.portable_shell_scanner`) mais
reste « best effort » et peut ne pas analyser une commande (erreur de scanner, pas refus)
[D1][S8].

### 1.5 Répondre : `once`, `always`, `reject`

Code et doc [S3][D1] :

| Réponse | Effet |
|---|---|
| `once` | approuve **uniquement** la demande en attente `requestID` ; pas de règle sauvegardée |
| `always` | approuve la demande **et** ajoute les motifs `save` comme règles `allow` durables du projet |
| `reject` | refuse la demande **et toutes les autres demandes en attente de la même session** ; un `message` optionnel devient un feedback (`CorrectedError`) |

Détails importants observés dans [S3] :

- `always` ne sauvegarde que si `request.save?.length > 0`. Sans `save`, il se comporte comme un
  `once` (et n'auto-approuve pas les autres).
- Après un `always` avec `save`, le service réévalue les autres demandes **de la même session** et
  les approuve si elles passent à `allow` avec la nouvelle règle. Il ne répond pas aux demandes des
  autres sessions ; en revanche la règle sauvegardée est projet-scopée et s'appliquera à leurs
  futures demandes.
- Le `message` de `reject` s'applique à **tout le lot** de demandes en attente de la session.
- Une demande en attente **n'a pas d'expiration** dans le service : elle reste dans la map `pending`
  jusqu'à `reply`, `close`, ou l'arrêt du service. `close` rejette toutes les demandes en attente
  [S3]. Le harness doit donc imposer son propre TTL.

### 1.6 Exemples exacts (testés)

Serveur isolé (aucune interaction avec le service OpenCode de l'utilisateur) :

```bash
XDG_DATA_HOME=/tmp/opencode/ociso/data \
XDG_CONFIG_HOME=/tmp/opencode/ociso/config \
XDG_STATE_HOME=/tmp/opencode/ociso/state \
XDG_CACHE_HOME=/tmp/opencode/ociso/cache \
opencode serve --hostname 127.0.0.1 --port 4399
# -> server listening on http://127.0.0.1:4399
# -> server password <mot de passe>
```

Authentification : **HTTP Basic**, utilisateur `opencode`, mot de passe du serveur [S7].
`opencode api` lit `OPENCODE_PASSWORD` quand on cible un serveur avec `--server` [S11].

Créer une session avec une politique `shell * ask` :

```bash
U=http://127.0.0.1:4399
P='<server password>'

curl -s -u "opencode:$P" -H 'content-type: application/json' \
  -d '{
    "title": "perm test",
    "location": { "directory": "/tmp/opencode/ociso/work/testrepo" },
    "permissions": [{ "action": "shell", "resource": "*", "effect": "ask" }]
  }' \
  "$U/api/session"
```

Créer une demande :

```bash
S=ses_...
curl -s -u "opencode:$P" -H 'content-type: application/json' \
  -d '{
    "action": "shell",
    "resources": ["git push -u origin research/permissions-pr"],
    "save": ["git push *"],
    "metadata": { "origin": "voice-harness" }
  }' \
  "$U/api/session/$S/permission"
# {"data":{"id":"per_...","effect":"ask"}}
```

Lister :

```bash
curl -s -u "opencode:$P" "$U/api/session/$S/permission"
# {"data":[{"id":"per_...","sessionID":"ses_...","action":"shell", ...}]}

curl -s -u "opencode:$P" "$U/api/permission/request"
# {"location":{"directory":"..."},"data":[...]}
```

Répondre une seule fois :

```bash
curl -s -o /dev/null -w '%{http_code}\n' \
  -u "opencode:$P" -H 'content-type: application/json' \
  -d '{"decision":"once"}' \
  "$U/api/session/$S/permission/per_.../reply"
# 204
```

Rejouer la même réponse (demande déjà consommée) :

```bash
# -> 404, {"_tag":"PermissionNotFoundError", ...}
```

C'est exactement le comportement attendu pour un **usage unique** : après `once`, la demande est
supprimée de `pending` [S3][S6].

Avec la CLI OpenCode (pratique pour un harness en shell) :

```bash
OPENCODE_PASSWORD="$P" opencode api --server "$U" \
  session.permission.list --param sessionID="$S"

OPENCODE_PASSWORD="$P" opencode api --server "$U" \
  session.permission.reply \
  --param sessionID="$S" \
  --param requestID=per_... \
  --data '{"decision":"once"}'
```

`opencode api` accepte aussi une méthode et un chemin brut :
`opencode api --server "$U" GET /api/permission/saved` [D5].

### 1.7 Ce que le serveur ne fait pas

- Pas de TTL sur une demande en attente [S3].
- Pas de « confirmation » liée à une phrase ou à un secret : `reply` est une simple décision
  booléenne (`once`/`always`/`reject`) sur un `requestID` [S1][S3].
- Pas de granularité « approuver seulement le remote X » : la portée est la demande, donc
  l'ensemble de ses `resources`. Si une commande composée produit `git fetch && git push`, les deux
  ressources sont dans la même demande et un `once` approuve l'ensemble [S8].
- Pas de règle `allow` qui gagne sur un `deny` configuré [S3][D1].

---

## 2. Confirmation orale à portée limitée (conception harness)

### 2.1 Principe : deux couches distinctes

**Couche A — OpenCode demande, le harness répond.**
Pour les actions internes à OpenCode (édition, lecture, commandes de test), le harness écoute
`permission.asked`, parle la demande, puis répond `once` ou `reject` sur **ce `requestID`**.

**Couche B — push et PR sont hors OpenCode.**
OpenCode n'a **pas** le droit de pousser ni de créer une PR. On le bloque par configuration et
policy, puis le harness exécute lui-même `git push` / `gh pr create` après confirmation. C'est la
seule façon de garantir que « confirmer le push » n'ouvre pas aussi la porte à `git push --force`,
à un autre remote, ou à une commande composée.

Configuration OpenCode recommandée :

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "permissions": [
    { "action": "shell", "resource": "*", "effect": "ask" },
    { "action": "shell", "resource": "git status *", "effect": "allow" },
    { "action": "shell", "resource": "git diff *", "effect": "allow" },
    { "action": "shell", "resource": "git log *", "effect": "allow" },
    { "action": "shell", "resource": "git push *", "effect": "deny" },
    { "action": "shell", "resource": "gh pr create *", "effect": "deny" },
    { "action": "shell", "resource": "gh pr merge *", "effect": "deny" }
  ],
  "policies": [
    { "action": "tool.use", "resource": "shell:git push *", "effect": "deny" },
    { "action": "tool.use", "resource": "shell:gh pr *", "effect": "deny" }
  ]
}
```

Les règles sont ordonnées : le `deny` final gagne sur le `ask` large [D1]. La policy est une
seconde barrière, appliquée après les règles et les `always` [D4]. **Ne pas** mettre
`session.permissions: "autoaccept"` ni lancer `opencode run --auto` : cela approuve automatiquement
tout ce qui n'est pas explicitement refusé [D12][D11].

### 2.2 Objet de confirmation (côté harness)

À la réception d'un `permission.asked`, le harness construit un **grant** en mémoire :

```json
{
  "requestID": "per_...",
  "sessionID": "ses_...",
  "action": "shell",
  "resources": ["git push -u origin research/permissions-pr"],
  "digest": "sha256(canonical_json(action,resources,save,sessionID))",
  "createdAt": 1792611051,
  "expiresAt": 1792611141,
  "state": "pending_voice",
  "attempts": 0
}
```

Règles :

1. **Lier au contenu, pas seulement à l'ID.** Le digest couvre `action`, `resources`, `save` et
   `sessionID`. Avant de répondre, le harness refait un `GET .../permission/:requestID` et vérifie
   que le digest est identique. Si la demande a changé ou disparu → `reject`/abandon.
2. **TTL court** (par ex. 60–120 s). À l'expiration, le harness envoie `reject` (ou laisse la
   demande et la rejette explicitement) et ne répond plus. Le serveur, lui, n'expire pas [S3].
3. **Usage unique.** Après un `reply once`, le grant est marqué `consumed` et le `requestID` est
   retiré de la table. Toute nouvelle confirmation orale doit créer un nouveau grant. Un rejeu de
   `reply` renvoie 404 (testé).
4. **Jamais `always` pour push/PR.** Si l'utilisateur dit « toujours », le harness répond
   **`reject`** et explique : « je ne peux pas mémoriser cette autorisation ; dis-le à chaque
   fois ». Pour les actions internes sans risque, `always` reste possible mais doit être une
   décision explicite séparée, avec affichage/lecture du motif `save` exact.
5. **Une seule demande à la fois.** Si plusieurs demandes sont en attente dans la session, les
   présenter une par une, dans un ordre déterministe. Un `reject` en rejette plusieurs côté serveur
   [S3] : le harness doit donc savoir que son lot est annulé.
6. **Refus par défaut.** Silence, ambiguïté, ASR sous le seuil, bruit, coupure réseau, expiration,
   digest différent → `reject`, jamais `once`.

### 2.3 Preuve orale : éviter l'auto-confirmation

Le micro peut entendre le TTS du harness. Une phrase du type « dis oui pour confirmer » peut être
auto-approuvée par le STT. Le protocole doit donc combiner :

- **Lecture exacte de l'action et de la cible** par le harness :
  « Demande : pousser la branche `research/permissions-pr` vers le remote `origin`. Réponds
  “confirme push” suivi de ton code. »
- **Un secret personnel non prononcé par le TTS** : un PIN à 4 chiffres connu de l'utilisateur,
  stocké localement (jamais dans le prompt, jamais dans les logs). Le harness demande « donne ton
  code de confirmation ». Il vérifie le PIN côté harness. Le TTS ne le dit jamais, donc le micro ne
  peut pas le répéter par écho.
- **Un mot d'action obligatoire** : `confirme push` ou `confirme PR`, pas un simple « oui ».
- **Vérification sémantique de la cible** : pour un push, l'utilisateur doit prononcer le nom de
  branche et le remote ; pour une PR, le nom de branche source et la base. Une confirmation qui ne
  contient pas les deux est rejetée.
- **Anti-rejeu** : chaque grant a un nonce ; une confirmation ne vaut que pour le grant courant.
  Limiter à 3 tentatives, puis `reject` et verrouiller quelques minutes.
- **Anti-bruit** : echo cancellation, mute du micro pendant le TTS, seuil de confiance ASR,
  confirmation finale du type « j'ai entendu : … ; c'est bien ça ? » avec une seconde réponse
  courte. Si le harness ne peut pas isoler la voix, refuser.

Ce mécanisme n'est pas fourni par OpenCode : c'est une **couche harness**. OpenCode ne connaît que
`once`/`always`/`reject` sur un `requestID` [S1][S3].

### 2.4 Flux complet « demander → confirmer oralement → exécuter → vérifier »

```text
1. OpenCode veut exécuter une action.
   -> permission.asked { id: per_123, action: "shell", resources: ["git push ..."] }

2. Harness :
   - calcule digest(per_123)
   - crée grant TTL 90 s
   - parle : "OpenCode demande de pousser la branche X vers origin. Dis: confirme push, puis ton code."

3. Utilisateur :
   - "confirme push, code 4 7 2 1"

4. Harness :
   - vérifie mot d'action + nom de branche + PIN
   - GET /api/session/:id/permission/per_123
   - vérifie digest inchangé et non expiré
   - POST .../per_123/reply {"decision":"once"}
   - marque le grant consumed

5. Exécution (couche B, hors OpenCode) :
   - git push -u origin research/permissions-pr
   - (puis, après une NOUVELLE confirmation) gh pr create ...

6. Vérification :
   - git ls-remote --heads origin research/permissions-pr
   - gh pr view <url> --json url,state,isDraft,mergeable,mergeStateStatus
   - gh pr checks <url>

7. Si échec :
   - parler l'erreur classée (auth, branche protégée, conflit, réseau)
   - ne pas réessayer automatiquement avec des paramètres différents
   - proposer une action corrigée, qui repassera par une nouvelle confirmation.
```

Pour le push et la PR, le harness ne doit **pas** passer par le shell d'OpenCode. Il construit
l'argumentaire (`argv`) lui-même, à partir de champs validés (remote, branche, base, titre, corps).
Pas de concaténation de commande avec du texte du modèle. Les noms de branche/remote sont validés
par regex et par `git check-ref-format`.

---

## 3. Pousser une branche et créer une PR sans écran

### 3.1 Ce que V2 propose (et ne propose pas)

- **`integration` V2** : routes `/api/integration`, `/api/integration/:id/connect/key`,
  `/connect/oauth`, `/connect/command`, etc. Le schéma décrit des **méthodes de connexion** et des
  **credentials** (clé, OAuth, commande, externe, env), pas des opérations de dépôt [S9]. La doc
  plugins montre `ctx.integration.get({ integrationID: "github" })` et
  `ctx.integration.connection.resolve(...)` pour récupérer un token, pas pour créer une PR [D7].
- **GitHub Action OpenCode** : `anomalyco/opencode/github@latest`, déclenchée par `/oc` ou
  `/opencode` dans un commentaire ; elle peut créer une branche et une PR **dans le runner GitHub**
  [S10]. Ce n'est pas un composant local, et cela suppose un workflow, des secrets et un événement
  GitHub. Utile comme fallback CI, pas comme mécanisme du harness vocal.
- **Aucune route PR/Git dans l'OpenAPI V2** : le groupe `server.permission` et le reste de
  `packages/protocol/openapi.json` n'exposent pas de `git push` ou `pulls.create` [S2].

**Conclusion** : pour le laptop local, `gh` CLI + `git` sont l'implémentation de référence.

### 3.2 Authentification `gh`

```bash
gh auth status
# -> compte actif, scopes, protocole git
```

Le manuel indique que `gh auth status` sort en **1** si un compte a un problème d'authentification
[G4]. Les scopes nécessaires pour pousser et créer une PR sur un dépôt privé sont au minimum
`repo` ; pour un dépôt public, `public_repo` peut suffire. Le compte de test `COOPER2703` a `repo`,
`read:org`, `gist`, `admin:public_key`.

Deux modes de push :

```bash
# A. SSH (si la clé est disponible)
git push -u origin research/permissions-pr
```

```bash
# B. HTTPS + credential helper gh (utile quand SSH n'est pas configuré)
git config --local credential.helper '!gh auth git-credential'
git push -u origin research/permissions-pr
```

Sur la machine de test, `git@github.com` a échoué (`Permission denied (publickey)`) alors que
`gh` était authentifié. Le mode HTTPS avec le helper `gh` a fonctionné. C'est un cas d'échec à
gérer : **tester `git ls-remote` avant de promettre que le push va marcher**, et proposer
`gh auth setup-git` ou le helper explicite.

### 3.3 Push exact

```bash
# Vérifications avant
git status --short
git rev-parse --abbrev-ref HEAD
git log --oneline -1
git ls-remote --heads origin "refs/heads/research/permissions-pr"

# Push (sans force, avec upstream)
git push -u origin research/permissions-pr
```

Le push crée la branche distante si elle n'existe pas. `-u` configure l'upstream, ce qui permet
ensuite à `gh pr create` de déduire `head` sans prompt [G1].

**Ne jamais** utiliser `--force` / `--force-with-lease` dans le flux automatique. Si le push est
rejeté pour non-fast-forward, le harness doit s'arrêter et demander une décision explicite, pas
forcer.

### 3.4 PR avec `gh pr create` (non interactif)

Le manuel confirme : sans `--title`/`--body`, `gh` ouvre un prompt ; `--title`, `--body` (ou
`--body-file`), `--base`, `--head`, `--label`, `--draft` suppriment les prompts. En cas de succès,
**l'URL de la PR est imprimée sur stdout** [G1].

Commande type :

```bash
gh pr create \
  --repo COOPER2703/dev-auto \
  --base main \
  --head research/permissions-pr \
  --title "docs(research): cartographie permissions, push et PR" \
  --body-file /tmp/opencode/pr-body.md \
  --label "wayfinder:research" \
  --draft
```

- `--base` : branche cible. Si absent, `gh` utilise `gh-merge-base` puis la branche par défaut du
  dépôt [G1]. **Toujours passer `--base` explicitement** dans un harness vocal : le défaut peut
  changer.
- `--head` : branche source. Le passer explicitement évite le prompt « where to push » et le fork
  automatique si la branche n'est pas encore poussée [G1]. `--head` accepte `user:branch`.
- `--body-file` : chemin d'un fichier Markdown, ou `-` pour stdin. Le corps peut être généré par le
  modèle et écrit dans un fichier temporaire ; ne pas l'interpoler dans un shell.
- `--label` : répétable. Les labels doivent exister dans le dépôt ; sinon `gh` échoue. Vérifier avec
  `gh label list --repo …`.
- `--draft` : crée une PR draft. Vérifier ensuite `isDraft: true` [G2].
- `--dry-run` : présent dans le manuel `gh` 2.101.0 [G1] : « Print details instead of creating the
  PR. May still push git changes. » C'est parfait pour valider les arguments sans créer de PR, mais
  cela peut quand même pousser. À utiliser après le push, pas à sa place.

Sans édition interactive, on peut aussi utiliser `--fill` pour reprendre le titre/corps des commits
[G1], mais pour une PR générée par le harness, `--title` + `--body-file` donne un contrôle exact.

### 3.5 Variante API GitHub (si `gh` n'est pas disponible)

`gh api` sait appeler n'importe quel endpoint REST et gère l'authentification :

```bash
gh api --method POST \
  -H "Accept: application/vnd.github+json" \
  repos/COOPER2703/dev-auto/pulls \
  -f title="docs(research): cartographie permissions, push et PR" \
  -f head="research/permissions-pr" \
  -f base="main" \
  -f body="$(cat /tmp/opencode/pr-body.md)" \
  -F draft=true
```

Points importants :

- `POST /repos/{owner}/{repo}/pulls` exige `title`, `head`, `base` ; `body`, `draft` et
  `maintainer_can_modify` sont optionnels [G5].
- **Les labels ne font pas partie du corps de création de PR** dans l'API REST : ils s'ajoutent via
  l'API Issues (`POST /repos/{owner}/{repo}/issues/{issue_number}/labels`) après création. `gh pr
  create --label` fait cette gymnastique pour vous [G1][G5].
- `-f` transmet une chaîne, `-F` un champ typé (booléen, entier). Pour un corps complexe, préférer
  `--input fichier.json` afin d'éviter les problèmes de quoting.
- Codes : `201` créé, `403` interdit, `422` validation (par ex. pas de commits entre base et head,
  base inexistante) [G5].

### 3.6 Ce qu'il faut vérifier avant de créer la PR

```bash
# La branche est bien sur le remote
git ls-remote --heads origin research/permissions-pr

# La base existe
gh api repos/COOPER2703/dev-auto/branches/main --jq '.name'

# Il y a bien des commits entre base et head
gh api "repos/COOPER2703/dev-auto/compare/main...research/permissions-pr" \
  --jq '{status,ahead_by,behind_by,total_commits}'

# Les labels existent
gh label list --repo COOPER2703/dev-auto --limit 100

# Pas de PR déjà ouverte pour cette branche
gh pr list --repo COOPER2703/dev-auto --head research/permissions-pr --state open
```

Si `ahead_by` vaut 0, GitHub renverra `422 No commits between …` [G5]. Si la base n'existe pas,
même type d'erreur. Si une PR existe déjà, `gh pr create` peut échouer ou signaler la PR existante ;
le harness doit traiter ce cas comme « déjà fait », vérifier l'URL avec `gh pr list` et ne pas créer
de doublon.

---

## 4. Vérifier après coup, et gérer les échecs

### 4.1 Push

```bash
git ls-remote --heads origin research/permissions-pr
# -> <sha> refs/heads/research/permissions-pr

git rev-parse HEAD
# comparer avec le sha distant
```

Si le sha distant diffère, le push n'a pas publié le commit attendu (mauvais dépôt, mauvais remote,
hook qui réécrit, ou push partiel). Le harness doit comparer les deux et refuser de dire « poussé »
sinon.

### 4.2 PR

```bash
gh pr view <url> \
  --json url,number,state,isDraft,headRefName,baseRefName,mergeable,mergeStateStatus,statusCheckRollup
```

Exemple de lecture :

- `url` : URL canonique à annoncer à l'oral ;
- `state: OPEN` ;
- `isDraft` : doit correspondre à la demande ;
- `mergeable: true|false|null` ; `null` signifie que GitHub calcule encore, il faut re-interroger
  [G5] ;
- `mergeStateStatus` : `CLEAN`, `DIRTY` (conflit), `BLOCKED` (règle de protection), `BEHIND`,
  `UNSTABLE`, etc. ;
- `statusCheckRollup` : synthèse des checks.

```bash
gh pr checks <url>
# code 0 = tout passe ; code 8 = checks en attente ; code non nul = échec [G3]
gh pr checks <url> --watch --interval 10
gh pr checks <url> --json name,state,bucket,link
```

Le champ `bucket` classe `state` en `pass`, `fail`, `pending`, `skipping`, `cancel` [G3].

### 4.3 Matrice des échecs

| Symptôme | Cause probable | Détection | Réaction du harness |
|---|---|---|---|
| `Permission denied (publickey)` | SSH non configuré | `git push` stderr | basculer HTTPS + helper `gh`, ou demander `gh auth setup-git` |
| `gh: To use GitHub CLI …` / 401 | token absent/expiré | `gh auth status` (exit 1) | ne pas réessayer ; demander `gh auth login` |
| `403 Forbidden` | scope manquant, droit absent, ruleset | sortie `gh`/API | expliquer le scope/droit ; ne pas contourner |
| `remote: error: GH006: Protected branch update failed` | push direct sur branche protégée | stderr du push | pousser une branche de feature, ouvrir une PR ; ne jamais forcer |
| `! [rejected] … (non-fast-forward)` | branche distante divergente | stderr | s'arrêter ; demander merge/rebase explicite ; pas de `--force` |
| `422 No commits between …` | base == head ou aucun commit | API `compare` / erreur | corriger la base ou pousser des commits |
| `422 Validation Failed` | base inexistante, head invalide, PR déjà existante | API/gh | vérifier `branches`, `compare`, `pr list` |
| `mergeable: false`, `mergeStateStatus: DIRTY` | conflit avec la base | `gh pr view --json` | annoncer le conflit ; ne pas merger |
| `mergeStateStatus: BLOCKED` | review/check/règle requis | `gh pr view` | attendre ; ne pas contourner |
| `gh pr checks` exit 8 | checks en attente | code de sortie | attendre / `--watch` ; ne pas dire « vert » |
| `gh pr checks` exit 1, « no checks reported » | aucun workflow/check sur la branche | code de sortie + message | dire « aucun check configuré », pas « CI verte » |
| checks `fail` | CI rouge | `gh pr checks --json` | annoncer les checks en échec et leurs liens |
| rate limit 403/429 | trop d'appels | en-têtes/erreur | backoff ; ne pas spammer |

Les codes et comportements `gh pr checks` (dont le code 8) viennent du manuel [G3]. Les codes REST
`201/403/422` et la sémantique `mergeable` viennent de la doc REST [G5].

### 4.4 Ne pas confondre « PR créée » et « PR mergeable »

`gh pr create` peut réussir (201, URL imprimée) alors que la PR est en conflit ou bloquée. Le
harness doit donc annoncer deux choses distinctes :

1. « PR créée : <url> » (fait vérifié par `gh pr view`) ;
2. « État : draft/open, mergeable/dirty/blocked, checks pending/pass/fail ».

---

## 5. Risques de sécurité

### 5.1 Shell arbitraire dans OpenCode

- `shell` s'exécute avec **les droits fichiers, processus et réseau de l'utilisateur** [D1]. Ce
  n'est pas un sandbox.
- Le scanner de commande est **best effort** ; il peut produire plusieurs ressources pour une
  commande composée, ou échouer à analyser. La doc recommande explicitement une **allowlist étroite**
  plutôt que des motifs censés reconnaître toutes les commandes dangereuses [D1].
- Les alias, fonctions shell, `git -c`, scripts et binaires locaux peuvent changer le comportement
  réel d'une commande dont le texte semble bénin. Un `git push` refusé au niveau permission peut
  être contourné par un script exécuté via une autre commande autorisée. C'est pourquoi la couche B
  (exécuteur hors OpenCode) est préférable pour push/PR.
- `execute` (Code Mode) peut atteindre des outils imbriqués ; un `deny` sur `execute` retire Code
  Mode, mais les outils imbriqués gardent leurs propres règles [D1].

### 5.2 Approbations durables et auto-acceptation

- `always` crée une règle `allow` **projet-scopée** et durable ; elle peut couvrir plus large que
  l'action confirmée, selon le motif `save` proposé par l'outil [D1][S3]. Ne jamais l'utiliser pour
  push/PR.
- `session.permissions: "autoaccept"` (TUI) accepte automatiquement **toutes** les demandes
  [D12]. `opencode run --auto` fait de même [D11]. Ces modes annulent le bénéfice de la
  confirmation orale.
- Une policy `tool.use` `deny` est la seule barrière qui survit à un `always` et à une config
  projet complaisante [D4]. Utiliser les deux (permissions + policies) pour push/PR.

### 5.3 Serveur OpenCode local

- Par défaut, le serveur écoute sur **localhost** (port 49374 sur macOS/Linux/WSL) et est protégé
  par un mot de passe [D6]. La doc insiste : ne pas l'exposer sans nécessité.
- `opencode serve --hostname 0.0.0.0` ou `opencode service set hostname 0.0.0.0` l'expose sur le
  réseau. L'authentification est **Basic** [S7] : sur du HTTP en clair, le mot de passe et les
  données transitent sans chiffrement. Sur un réseau non fiable, utiliser un tunnel SSH ou rester
  en localhost.
- `opencode pair --remote` expose le serveur via OpenTunnel ; l'accès exige le mot de passe du
  serveur et les liens de pairing sont à usage unique / 5 minutes [D6]. C'est une surface
  d'attaque supplémentaire : ne pas l'activer pour le harness.
- Le fichier d'enregistrement du service contient l'URL, le PID et le mot de passe, avec des
  permissions **0600** [S7][S11]. Toute lecture par un autre processus du même utilisateur (ou
  root) donne le contrôle du serveur.
- `--cors` élargit les origines autorisées ; ne pas mettre `*` sans besoin [D5][D6].

### 5.4 Clés API et tokens

- Les credentials OpenCode (clés API, OAuth) sont stockés dans la base locale
  `~/.local/share/opencode/opencode.db` (et l'ancien `auth.json`), avec des permissions **0600**
  observées sur la machine de test [S12][V1]. Ce n'est pas du chiffrement au repos : toute lecture
  du fichier par l'utilisateur ou root expose les clés.
- `gh` stocke son token dans `~/.config/gh/hosts.yml`, permissions **0600** observées [V1]. Le token
  a les scopes du compte ; un token `repo` permet de pousser et de créer des PR sur tous les dépôts
  accessibles. Le principe du moindre privilège n'est pas respecté par un token classique large.
- Ne jamais passer un token en argument de commande (visible dans `ps`/l'historique) ni le logger.
  Préférer `gh` (qui lit son propre store) ou un `GITHUB_TOKEN` fourni par l'environnement, avec
  `--no-show-token`.
- `gh auth token` imprime le token en clair ; ne pas l'utiliser dans le harness.
- Les logs OpenCode peuvent contenir des prompts, du contenu de fichiers et des en-têtes ; la doc
  troubleshooting demande de les nettoyer avant partage [D13].

### 5.5 Confirmation orale et prompt injection

- Le contenu lu par le modèle (fichiers, pages web, résultats d'outils, issues) peut contenir des
  instructions malveillantes. Une demande de permission peut donc être déclenchée par une injection,
  pas par l'utilisateur. La confirmation orale doit porter sur **l'action exacte**, pas sur un
  résumé généré par le modèle.
- Le TTS peut être entendu par le micro (auto-confirmation). Utiliser un PIN non prononcé, le mute,
  l'echo cancellation, et un seuil ASR [conception harness, section 2.3].
- Un attaquant qui obtient le micro ou le serveur peut répondre à la place de l'utilisateur. Le PIN
  et le TTL limitent le rejeu, pas une compromission complète de la machine.
- Le `requestID` est un identifiant généré (`per_` + suffixe), mais ce n'est pas un secret
  d'autorisation : l'authentification du serveur reste le mot de passe Basic [S4][S7].

### 5.6 Branches et dépôts

- Une branche protégée peut interdire le push direct ; ne pas tenter de contourner avec `--force` ou
  un bypass de ruleset. Le harness doit pousser une branche de feature et ouvrir une PR.
- `gh pr create` peut créer une PR dans un dépôt que l'utilisateur ne contrôle pas si le token et
  les droits le permettent. Le harness doit **figer** `--repo` et `--base` (allowlist), et refuser
  toute valeur venue du modèle sans confirmation.
- Les PR draft sont visibles ; ne pas les confondre avec des PR prêtes à reviewer. Vérifier
  `isDraft`.

---

## 6. Journal des tests effectués

### 6.1 Serveur OpenCode isolé

But : ne pas toucher au service OpenCode de l'utilisateur, ni à sa base de sessions.

- Serveur lancé avec des répertoires XDG séparés sous `/tmp/opencode/ociso`, sur
  `127.0.0.1:4399`.
- Session créée avec `permissions: [{action:"shell", resource:"*", effect:"ask"}]`.
- `POST /api/session/:id/permission` avec
  `resources:["git push origin research/permissions-pr"]`, `save:["git push *"]`
  → `{"effect":"ask"}`, demande `per_...` visible dans `GET /api/session/:id/permission`.
- `GET /api/permission/request` → demande visible avec `location`.
- `POST .../reply {"decision":"once"}` → **204**, puis `GET .../permission` → `{"data":[]}`.
- Deuxième `POST .../reply` sur le même ID → **404 `PermissionNotFoundError`** (usage unique
  confirmé).
- `always` sur une deuxième demande → **204**, puis `GET /api/permission/saved` montre
  `{"action":"shell","resource":"git push *","projectID":...}`.
- Une demande suivante `git push origin foo` dans une **autre session** du même projet →
  `{"effect":"allow"}` : la règle sauvegardée est bien projet-scopée et inter-sessions.
- `reject` avec `message` sur une demande, avec une autre demande en attente dans la même session →
  les **deux** demandes disparaissent (`GET .../permission` → `[]`, `GET` de l'autre → 404).
  Comportement conforme au code [S3].
- Session avec `permissions:[{action:"shell",resource:"*",effect:"deny"}]` → `POST .../permission`
  renvoie `{"effect":"deny"}` et **aucune** demande en attente.
- Flux SSE `GET /api/event` : `permission.asked` puis `permission.replied` observés avec les bons
  `data`.
- `opencode api --server http://127.0.0.1:4399 session.permission.list --param sessionID=...` et
  `session.permission.reply` testés avec `OPENCODE_PASSWORD` ; la CLI sort en 1 et affiche
  `HTTP 404 Not Found` sur un rejeu.
- La règle sauvegardée de test a été supprimée via `DELETE /api/permission/saved/:id` (204) ;
  `GET` final → `{"data":[]}`.

Aucun de ces tests n'a touché la base OpenCode de l'utilisateur : les XDG étaient isolés.

### 6.2 Dépôt `COOPER2703/dev-auto`

- Le dépôt distant était **vide** au moment du test (`Git Repository is empty`, 409 sur
  `/git/refs`), alors que le worktree local avait déjà le commit d'init `d8cdb83` sur `main`.
  Après le premier push, GitHub a pris `research/permissions-pr` comme **branche par défaut** ; il
  n'y avait toujours pas de `main` distante.
- Le remote local était en SSH (`git@github.com:...`) et le push SSH a échoué faute de clé
  (`Permission denied (publickey)`). Le remote a été basculé en HTTPS avec
  `git config --local credential.helper '!gh auth git-credential'`.
- Push effectué :

  ```bash
  git push -u origin research/permissions-pr
  # -> * [new branch] research/permissions-pr -> research/permissions-pr
  git ls-remote --heads origin
  # -> 8eba5ece016445cc07ca936b1d6f8041c1a3c94d refs/heads/research/permissions-pr
  ```

- Une base temporaire `pr-test-base` a été créée au commit parent `d8cdb83` (puisque `main`
  n'existait pas sur le remote) :

  ```bash
  git push origin d8cdb83:refs/heads/pr-test-base
  ```

- `gh pr create --dry-run` a validé les arguments sans créer de PR :

  ```text
  Would have created a Pull Request with:
  title: [TEST] Recette push et création de PR
  draft: true
  base:  pr-test-base
  head:  research/permissions-pr
  ```

- PR réelle de test créée puis vérifiée :

  ```bash
  gh pr create --repo COOPER2703/dev-auto --base pr-test-base \
    --head research/permissions-pr \
    --title "[TEST] Recette push et création de PR" \
    --body-file /tmp/opencode/pr-body.md --draft
  # -> https://github.com/COOPER2703/dev-auto/pull/15

  gh pr view 15 --repo COOPER2703/dev-auto \
    --json url,number,state,isDraft,headRefName,baseRefName,mergeable,mergeStateStatus,statusCheckRollup
  # -> state OPEN, isDraft true, mergeable MERGEABLE, mergeStateStatus CLEAN, checks []
  ```

- `gh pr checks 15` a renvoyé **exit 1** avec « no checks reported on the
  'research/permissions-pr' branch » : il n'y avait aucun workflow sur ce dépôt. C'est différent
  d'un code 8 (checks en attente) et d'un échec de check.
- PR fermée immédiatement :

  ```bash
  gh pr close 15 --repo COOPER2703/dev-auto --delete-branch=false
  # -> state CLOSED, closedAt 2026-10-21T19:35:09Z
  ```

- Branche de base temporaire supprimée :

  ```bash
  git push origin --delete pr-test-base
  ```

- Vérification finale : `gh pr list --state open` → `[]` ; `git ls-remote --heads origin` ne montre
  plus que `research/permissions-pr`. **Aucune PR ouverte, aucune branche de test restante.**

> Détail à retenir pour le harness : un dépôt GitHub peut être **vide** alors que le clone local a
> des commits. `gh pr create` exige une base existante ; le harness doit vérifier `branches/base`
> avant de créer la PR, et sinon proposer de pousser la base ou de choisir une autre base. Ici la
> branche par défaut est devenue la branche poussée, ce qui est un piège : ne pas supposer que
> `main` existe parce que le dépôt a un `default_branch`.

---

## 7. Recommandation d'architecture pour le harness

```text
┌────────────────────┐        SSE /api/event          ┌────────────────────┐
│ OpenCode 2.1.2     │ ─────────────────────────────► │ Harness vocal      │
│ permissions:       │                                │ - STT/TTS          │
│  shell * ask       │ ◄──── reply once/reject ────── │ - grants TTL       │
│  git push * deny   │      /api/session/.../reply    │ - PIN + readback   │
│  gh pr * deny      │                                │ - executor gh/git  │
└────────────────────┘                                └─────────┬──────────┘
                                                                │
                                                                │ argv validé
                                                                ▼
                                                      git push / gh pr create
                                                                │
                                                                ▼
                                                      vérification gh/git
```

Principes :

1. **OpenCode ne pousse jamais.** `deny` + policy sur `shell:git push *` et `shell:gh pr *`.
2. **Le harness répond `once` uniquement**, avec TTL, digest, PIN, anti-rejeu.
3. **Le harness exécute `git push` et `gh pr create` lui-même**, avec des arguments figés/validés.
4. **Vérification obligatoire** avant d'annoncer le succès : `git ls-remote`, `gh pr view`,
   `gh pr checks`.
5. **Pas de `always` pour les actions hors OpenCode**, et pas de `--auto`/`autoaccept`.

---

## 8. Sources

### Documentation officielle OpenCode V2

- [D1] Permissions — <https://opencode.ai/v2/docs/permissions>
- [D2] Configuration — <https://opencode.ai/v2/docs/config>
- [D3] Agents — <https://opencode.ai/v2/docs/agents>
- [D4] Policies — <https://opencode.ai/v2/docs/policies>
- [D5] CLI commands (`serve`, `api`, `pair`, `service`) — <https://opencode.ai/v2/docs/cli/commands>
- [D6] Web / serveur local, pairing, remote access — <https://opencode.ai/v2/docs/cli/web>
- [D7] Plugins (permission, session.update, integration) — <https://opencode.ai/v2/docs/build/plugins>
- [D8] JavaScript client (`@opencode/client`) — <https://opencode.ai/v2/docs/build/client>
- [D11] `opencode run --help` : `--auto  Auto-approve permissions that are not explicitly denied`
  (CLI 2.1.2, vérifié localement)
- [D12] CLI config TUI : `session.permissions` = `prompt` ou `autoaccept` —
  <https://opencode.ai/v2/docs/cli/config>
- [D13] Troubleshooting / nettoyage des logs — <https://opencode.ai/v2/docs/troubleshooting>

### Source `anomalyco/opencode`, branche `dev`, commit `863ea41`

- [S1] `packages/protocol/src/groups/permission.ts` — routes, payloads, OpenAPI annotations.
- [S2] `packages/protocol/openapi.json` — chemins, schémas `Permission.*`, codes 204/404.
- [S3] `packages/core/src/permission.ts` — `evaluate`, `merge`, `ask`, `assert`, `reply`, `pending`,
  `close`, règles sauvegardées, absence de TTL.
- [S4] `packages/schema/src/permission.ts` — `Request`, `Reply`, `Effect`, `Rule`, `Ruleset`,
  événements `permission.asked` / `permission.replied`.
- [S5] `packages/core/src/permission/saved.ts` + `packages/schema/src/permission-saved.ts` — règles
  durables, projet-scopées, unicité, CRUD.
- [S6] `packages/server/src/handlers/permission.ts` — ownership `sessionID`/`requestID`, 404.
- [S7] `packages/server/src/auth.ts`, `packages/server/src/middleware/authorization.ts`,
  `packages/cli/src/services/service-registration.ts` — Basic auth, mot de passe, session cookie,
  fichier 0600.
- [S8] `packages/core/src/tool/plugin/shell.ts` — `permission.assert({action:"shell", resources,
  save})`, scanner, `access.authorizeExternal`.
- [S9] `packages/protocol/src/groups/integration.ts`, `packages/schema/src/integration.ts` — API
  `integration` (méthodes key/OAuth/command/external/env), pas d'opération Git/PR.
- [S10] `github/README.md`, `github/action.yml` — GitHub Action OpenCode, déclenchement `/oc`,
  création de branche/PR dans le runner.
- [S11] `packages/cli/src/commands/handlers/api.ts`,
  `packages/cli/src/services/server-connection.ts` — `opencode api`, `OPENCODE_PASSWORD`,
  `--server`.
- [S12] `packages/core/src/credential/sql.ts`, `packages/core/src/credential.ts`,
  `packages/core/src/database/sqlite.bun.ts` — credentials en base SQLite, permissions observées
  `0600` sur `opencode.db` et `auth.json`.

### `gh` CLI et GitHub REST

- [G1] `gh pr create` — <https://cli.github.com/manual/gh_pr_create>
- [G2] `gh pr view` — <https://cli.github.com/manual/gh_pr_view>
- [G3] `gh pr checks` — <https://cli.github.com/manual/gh_pr_checks>
- [G4] `gh auth status` — <https://cli.github.com/manual/gh_auth_status>
- [G5] REST — Pull requests (create/get, `mergeable`, 201/403/422) —
  <https://docs.github.com/en/rest/pulls/pulls?apiVersion=2022-11-28>
- [G6] REST — Check runs / checks —
  <https://docs.github.com/en/rest/checks/runs?apiVersion=2022-11-28>

### Vérifications locales

- [V1] Machine de test : `opencode --version` = `2.1.2`, `gh --version` = `2.101.0`,
  `gh auth status` compte `COOPER2703`, permissions `0600` sur
  `~/.local/share/opencode/opencode.db`, `~/.local/share/opencode/auth.json`,
  `~/.config/gh/hosts.yml`, et `~/.local/state/opencode/service.json`.
- [V2] Serveur isolé `127.0.0.1:4399` : tests permission list/create/reply/always/reject/deny,
  SSE, `opencode api`, décrits en section 6.1.
- [V3] `COOPER2703/dev-auto` : push HTTPS via helper `gh`, PR de test avec base temporaire puis
  fermeture, décrit en section 6.2.
