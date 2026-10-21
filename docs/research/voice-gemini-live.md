# Gemini Live via l'API vocale d'OpenCode en forfait gratuit

- **Ticket :** [COOPER2703/dev-auto#2](https://github.com/COOPER2703/dev-auto/issues/2)
- **Carte parente :** #1 (wayfinder « harness vocal pour OpenCode »)
- **Date de la recherche :** 2026-10-21
- **Environnement :** OpenCode `v2.1.2`, binaire `/home/cooper/.opencode/bin/opencode`, Linux, Intel N100, 15 Go, pas de GPU.
- **Worktree :** `/tmp/opencode/research-voice-gemini-live`, branche `research/voice-gemini-live`.
- **Limite de cette recherche :** aucune clé Google n'est configurée sur la machine. Le chemin serveur, les routes, le payload et les erreurs sont vérifiés localement ; **la session Gemini Live réelle (audio, quota, latence) n'a pas pu être exécutée**. La section « Ce qui reste à tester » donne le protocole exact.

---

## Réponse directe

1. **Oui, OpenCode sait déjà ouvrir une session Gemini Live.** La version installée `v2.1.2` expose `POST /api/experimental/voice/session`, accepte le modèle `gemini-3.8-live` en transport `websocket`, et va chercher la clé Google côté serveur pour fabriquer un jeton éphémère Google ([preuve locale](#preuves-locales), [source `realtime.ts`](https://github.com/anomalyco/opencode/blob/863ea413649484b5761cd0c5c0e54c559a5c8b52/packages/ai/src/experimental/realtime.ts)).
2. **Le 404 n'est ni un flag expérimental, ni un problème de location, ni l'absence de clé.** C'est un **décalage de version** : `GET /api/experimental/voice/models` a été ajouté sur `dev` le 21 octobre 2026 à 06:13 UTC (commit [`79fd806`](https://github.com/anomalyco/opencode/commit/79fd806343db0de150f23b09e4b9199434d6779c), PR [#57194](https://github.com/anomalyco/opencode/pull/57194)), **après** le tag `v2.1.2` créé à 03:41 UTC. Aucune version publiée ne contient encore cette route.
3. **Une clé Google AI Studio gratuite suffit techniquement** : Gemini 3.8 Live est listé « Free of charge » en entrée et en sortie sur le free tier ([pricing](https://ai.google.dev/gemini-api/docs/pricing)). La France fait partie des régions servies ([régions](https://ai.google.dev/gemini-api/docs/available-regions)).
4. **Mais le free tier n'est pas dimensionné pour des sessions de conduite longues et quotidiennes** : l'API Live est en **Preview**, les quotas exacts RPM/TPM/RPD ne sont pas publiés et dépendent du projet, et le contenu du free tier est utilisé pour améliorer les produits Google ([pricing](https://ai.google.dev/gemini-api/docs/pricing), [rate limits](https://ai.google.dev/gemini-api/docs/rate-limits)).
5. **Pour le harness, le vrai travail restant est côté client audio** : OpenCode fournit le broker de jeton et les routes serveur, mais aucun client audio temps réel n'existe dans les packages publics inspectés. Il faudra capter le micro, rééchantillonner en PCM 16 kHz, envoyer les chunks, jouer le PCM 24 kHz et gérer reprise/compression.

---

## Preuves locales

### Version installée

```bash
opencode --version
# opencode v2.1.2
```

### Le 404, tel qu'observé

```bash
opencode api get /api/experimental/voice/models
# HTTP 404 Not Found
```

Le serveur local (confirmé sur un serveur isolé démarré sur le port 49401, avec des répertoires XDG dédiés) ne contient qu'**une** route `voice` dans son document OpenAPI : la route de liste n'est pas enregistrée.

```bash
opencode api get /openapi.json \
  | jq -r '.paths | keys[] | select(contains("voice"))'
# /api/experimental/voice/session
```

En revanche, la route de session existe bien et atteint le contrôleur de clé :

```bash
opencode api post /api/experimental/voice/session \
  -d '{"model":"gemini-3.8-live","transport":{"type":"websocket"}}'
# {"_tag":"ServiceUnavailableError",
#  "message":"Voice needs a Google API key on the server (a stored key, GEMINI_API_KEY,
#             GOOGLE_GENERATIVE_AI_API_KEY, or GOOGLE_API_KEY).",
#  "service":"google"}
# HTTP 503 Service Unavailable
```

Avec une clé **factice** configurée, on n'a plus le message « clé absente » mais une erreur amont de Google, ce qui prouve que le chemin de credential fonctionne et que le 404 n'a rien à voir avec la clé :

```bash
GEMINI_API_KEY=dummy-invalid-key opencode serve --port 49402
# puis POST /api/experimental/voice/session
# {"_tag":"ServiceUnavailableError",
#  "message":"API key not valid. Please pass a valid API key.",
#  "service":"google"}
# HTTP 503
```

La même chose fonctionne en stockant une clé via l'API d'intégration (testé avec une clé factice, dans le serveur isolé) :

```bash
opencode api post /api/integration/google/connect/key \
  -d '{"key":"dummy-invalid-key"}'
# HTTP 204

opencode api get /api/integration/google | jq '.data.connections'
# [{"type":"credential","id":"cred_...","label":"Google","method":"key"}]
```

> Conclusion locale : **la route de session est opérationnelle sur `v2.1.2` ; seule la route de liste des modèles manque.**

---

## 1. Cause exacte du 404

### Chronologie

| Date (UTC) | Événement | Preuve |
|---|---|---|
| 2026-10-19 20:04 | Ajout du support Gemini Live (jeton éphémère) | commit [`31dc762`](https://github.com/anomalyco/opencode/commit/31dc7621), PR [#56888](https://github.com/anomalyco/opencode/pull/56888) |
| 2026-10-21 03:41 | Tag de release `v2.1.2` | `git log -1 v2.1.2` → `a517687 release: v2.1.2` |
| 2026-10-21 06:13 | Ajout de `GET /api/experimental/voice/models` sur `dev` | commit [`79fd806`](https://github.com/anomalyco/opencode/commit/79fd806343db0de150f23b09e4b9199434d6779c), PR [#57194](https://github.com/anomalyco/opencode/pull/57194) |

À `v2.1.2`, le groupe `voice` ne déclare que `voice.session` ([`packages/protocol/src/groups/voice.ts` @ v2.1.2](https://github.com/anomalyco/opencode/blob/v2.1.2/packages/protocol/src/groups/voice.ts)) :

```bash
git show v2.1.2:packages/protocol/src/groups/voice.ts | grep HttpApiEndpoint
#     HttpApiEndpoint.post("voice.session", "/api/experimental/voice/session", {
```

Sur `dev`, le même fichier déclare trois routes ([permalink](https://github.com/anomalyco/opencode/blob/863ea413649484b5761cd0c5c0e54c559a5c8b52/packages/protocol/src/groups/voice.ts)) :

- `GET  /api/experimental/voice/models`
- `POST /api/experimental/voice/session`
- `POST /api/experimental/voice/session/:sessionID/instructions`

### Ce que ce n'est pas

- **Pas un flag expérimental.** Le groupe est ajouté inconditionnellement à l'API ([`packages/protocol/src/api.ts#L170`](https://github.com/anomalyco/opencode/blob/863ea413649484b5761cd0c5c0e54c559a5c8b52/packages/protocol/src/api.ts#L170)) et le handler est enregistré sans condition ([`packages/server/src/handlers.ts#L46`](https://github.com/anomalyco/opencode/blob/863ea413649484b5761cd0c5c0e54c559a5c8b52/packages/server/src/handlers.ts#L46)). Le préfixe `/api/experimental/` est un nom d'URL, pas une option de configuration.
- **Pas un problème de location.** Une location inconnue produirait une `LocationNotFoundError` en JSON, pas un 404 vide ; surtout, `/openapi.json` ne liste pas la route du tout.
- **Pas l'absence de clé.** Sans clé, la route existante répond **503** avec un message explicite, jamais 404. Avec une clé factice, elle répond encore **503** (clé invalide). La différence 404/503 est donc purement routage.
- **Pas un problème de version de protocole Gemini.** Le support Gemini Live est déjà dans `v2.1.2` : le moteur `gemini-3.8-live`, le POST `auth_tokens` et l'URL `BidiGenerateContentConstrained` y sont présents.

### Commande de vérification exacte

```bash
# 1. Version
opencode --version

# 2. Routes réellement servies par l'installation
opencode api get /openapi.json \
  | jq -r '.paths | keys[] | select(contains("voice"))'

# 3. La route manquante
opencode api get /api/experimental/voice/models          # 404

# 4. La route présente, jusqu'au contrôle de clé
opencode api post /api/experimental/voice/session \
  -d '{"model":"gemini-3.8-live","transport":{"type":"websocket"}}'   # 503 clé absente
```

**Correction attendue :** aucune version publiée ne corrige le 404 pour l'instant. Il faut soit attendre une release contenant #57194, soit utiliser un build `dev`, soit **appeler directement `POST /session`** avec le modèle connu (`gemini-3.8-live`) sans passer par `/models`. Pour le harness, cette dernière option est la plus simple et fonctionne dès `v2.1.2`.

> Attention : le site de docs et `https://opencode.ai/v2/openapi.json` suivent `dev` et listent déjà les trois routes ([OpenAPI public](https://opencode.ai/v2/openapi.json), [référence API](https://opencode.ai/v2/docs/api/)). C'est normal qu'ils soient en avance sur le binaire installé.

---

## 2. Ce que fait l'API vocale d'OpenCode

### Routes

| Route | `v2.1.2` installé | `dev` (2026-10-21) | Rôle |
|---|---:|---:|---|
| `GET /api/experimental/voice/models` | ❌ 404 | ✅ | Liste les moteurs, transports et disponibilité de clé |
| `POST /api/experimental/voice/session` | ✅ | ✅ | Crée une session et renvoie SDP WebRTC ou jeton WebSocket |
| `POST /api/experimental/voice/session/:id/instructions` | ❌ 404 | ✅ | Remplace les instructions d'une session **hébergée** (OpenCode Console), pas Gemini |

Sources : [`voice.ts` du protocole](https://github.com/anomalyco/opencode/blob/863ea413649484b5761cd0c5c0e54c559a5c8b52/packages/protocol/src/groups/voice.ts), [`handlers/voice.ts`](https://github.com/anomalyco/opencode/blob/863ea413649484b5761cd0c5c0e54c559a5c8b52/packages/server/src/handlers/voice.ts), [OpenAPI local vérifié](#preuves-locales).

### Moteurs connus

`Realtime.engines` déclare quatre moteurs ([`packages/ai/src/experimental/realtime.ts#L30-L47`](https://github.com/anomalyco/opencode/blob/863ea413649484b5761cd0c5c0e54c559a5c8b52/packages/ai/src/experimental/realtime.ts#L30-L47)) :

| Modèle | Provider | Transports |
|---|---|---|
| `gpt-live-1` | `openai` | WebRTC |
| `gpt-realtime-2.1` | `openai` | WebRTC, WebSocket |
| `grok-voice-3.0` | `xai` | WebSocket |
| **`gemini-3.8-live`** | **`google`** | **WebSocket** |

Le matcher accepte le nom de famille et les variantes qui commencent par `gemini-3.8-live-` ([`engine()`](https://github.com/anomalyco/opencode/blob/863ea413649484b5761cd0c5c0e54c559a5c8b52/packages/ai/src/experimental/realtime.ts#L49-L51)). En pratique, seul `gemini-3.8-live` est listé et testé ; les autres modèles Live (`gemini-3.1-flash-live-preview`, `gemini-3.8-live-extended-thinking`) ne sont pas des moteurs OpenCode de premier rang. **Utiliser `gemini-3.8-live`.**

### Flux Gemini Live côté OpenCode

Pour `gemini-3.8-live`, `Realtime.connect` fait ceci ([`realtime.ts#L94-L116`](https://github.com/anomalyco/opencode/blob/863ea413649484b5761cd0c5c0e54c559a5c8b52/packages/ai/src/experimental/realtime.ts#L94-L116)) :

1. Calcule `expiresAt = maintenant + 60 s`.
2. `POST https://generativelanguage.googleapis.com/v1beta/auth_tokens` avec l'en-tête `x-goog-api-key`.
3. Corps : `uses: 1`, `expireTime: maintenant + 30 min`, `newSessionExpireTime: maintenant + 60 s`, et **`bidiGenerateContentSetup`** contenant le `session` fourni par l'appelant plus `model: models/gemini-3.8-live`.
4. Renvoie au client :
   - `url = wss://generativelanguage.googleapis.com/ws/google.ai.generativelanguage.v1beta.GenerativeService.BidiGenerateContentConstrained`
   - `token = data.name` (le jeton éphémère)
   - `expiresAt = (maintenant + 60 s) / 1000`

La clé API longue durée n'est jamais exposée au client : c'est OpenCode qui la détient et qui fabrique le jeton éphémère. C'est le modèle recommandé par Google ([jetons éphémères](https://ai.google.dev/gemini-api/docs/live-api/ephemeral-tokens)).

### Détail important : `expiresAt` n'est pas l'expiration du jeton

OpenCode renvoie `expiresAt = now + 60 s`, qui correspond à `newSessionExpireTime` (fenêtre pour **démarrer** une session). Le jeton lui-même a `expireTime = now + 30 min` pour **envoyer des messages** ([`realtime.ts#L95-L104`](https://github.com/anomalyco/opencode/blob/863ea413649484b5761cd0c5c0e54c559a5c8b52/packages/ai/src/experimental/realtime.ts#L95-L104), [référence AuthToken](https://ai.google.dev/api/live#authtoken)). Un client qui traite `expiresAt` comme « fin de session dans 60 s » se trompe ; il doit se connecter dans la minute, puis peut converser jusqu'à 30 min (et reprendre avec le même jeton, une reprise ne comptant pas comme un usage).

### Clé Google côté serveur

`Voice.session` cherche une connexion d'intégration de type `env` ou `key` pour le provider `google` ([`packages/core/src/voice.ts#L92-L142`](https://github.com/anomalyco/opencode/blob/863ea413649484b5761cd0c5c0e54c559a5c8b52/packages/core/src/voice.ts#L92-L142)). Les sources acceptées sont :

- une clé stockée dans l'intégration `google` ;
- `GEMINI_API_KEY` ;
- `GOOGLE_GENERATIVE_AI_API_KEY` ;
- `GOOGLE_API_KEY`.

Le message d'erreur exact le confirme ([`voice.ts#L179-L185`](https://github.com/anomalyco/opencode/blob/863ea413649484b5761cd0c5c0e54c559a5c8b52/packages/core/src/voice.ts#L179-L185)). Sur un service lancé par le desktop, une variable exportée dans un shell n'est pas forcément visible : **stocker la clé via l'intégration est plus fiable** que l'environnement.

### `voice.models` sur `dev`

La route liste les quatre moteurs avec `available: true/false` selon qu'une connexion existe ([`voice.ts#L98-L122`](https://github.com/anomalyco/opencode/blob/863ea413649484b5761cd0c5c0e54c559a5c8b52/packages/core/src/voice.ts#L98-L122)). Sans clé Google, `gemini-3.8-live` apparaît avec `available: false` et il n'y a pas de `default` si aucun moteur n'est disponible. C'est un endpoint de **découverte**, pas un prérequis pour ouvrir une session.

---

## 3. Gemini Live : protocole et configuration de session

### 3.1 WebSocket et authentification

- L'API Live est un WebSocket avec état ([vue d'ensemble](https://ai.google.dev/gemini-api/docs/live)).
- Avec une clé API classique : `wss://generativelanguage.googleapis.com/ws/google.ai.generativelanguage.v1beta.GenerativeService.BidiGenerateContent?key=...` ([guide WebSocket](https://ai.google.dev/gemini-api/docs/live-api/get-started-websocket)).
- Avec un jeton éphémère : `.../BidiGenerateContentConstrained?access_token=<token>` ([guide WebSocket](https://ai.google.dev/gemini-api/docs/live-api/get-started-websocket), [jetons éphémères](https://ai.google.dev/gemini-api/docs/live-api/ephemeral-tokens)).
- Le premier message doit être un `BidiGenerateContentSetup` ([guide WebSocket](https://ai.google.dev/gemini-api/docs/live-api/get-started-websocket)).
- Quand un jeton contient `bidiGenerateContentSetup` et que `fieldMask` est vide, **la configuration effective est entièrement celle du jeton** ; le setup envoyé sur la connexion est ignoré ([référence AuthToken](https://ai.google.dev/api/live#authtoken)). C'est exactement ce que fait OpenCode : le client n'a pas besoin de rejouer les instructions ni la voix, mais doit quand même envoyer un premier message `setup` conforme au protocole.

### 3.2 Audio

- Entrée : PCM brut 16 bits, little-endian, **16 kHz** ; l'API rééchantillonne si besoin ([vue d'ensemble](https://ai.google.dev/gemini-api/docs/live), [guide WebSocket](https://ai.google.dev/gemini-api/docs/live-api/get-started-websocket)).
- Sortie : PCM brut 16 bits, **24 kHz** ([vue d'ensemble](https://ai.google.dev/gemini-api/docs/live)).
- MIME d'envoi : `audio/pcm;rate=16000` ([guide WebSocket](https://ai.google.dev/gemini-api/docs/live-api/get-started-websocket)).
- Chunks recommandés : **20 à 100 ms** ; ne pas bufferiser une seconde ([bonnes pratiques](https://ai.google.dev/gemini-api/docs/live-api/best-practices)).
- Rééchantillonner le micro (souvent 44,1/48 kHz) vers 16 kHz avant envoi ([bonnes pratiques](https://ai.google.dev/gemini-api/docs/live-api/best-practices)).

### 3.3 Champs de configuration utiles

D'après le document de découverte de l'API et la référence ([découverte v1beta](https://generativelanguage.googleapis.com/$discovery/rest?version=v1beta), [référence Live](https://ai.google.dev/api/live)) :

| Champ | Où | Usage |
|---|---|---|
| `model` | racine du setup | `models/gemini-3.8-live` |
| `generationConfig.responseModalities` | `generationConfig` | `["AUDIO"]` (seule modalité de sortie des modèles audio natifs) |
| `generationConfig.speechConfig.voiceConfig.prebuiltVoiceConfig.voiceName` | `generationConfig` | voix, par exemple `Kore` |
| `systemInstruction` | racine | persona, règles, langue (français) |
| `realtimeInputConfig.automaticActivityDetection` | racine | VAD automatique, seuils, silence |
| `realtimeInputConfig.activityHandling` | racine | `START_OF_ACTIVITY_INTERRUPTS` pour le barge-in |
| `inputAudioTranscription` | racine | transcription de l'utilisateur |
| `outputAudioTranscription` | racine | transcription de Gemini (texte pour l'UI/les logs) |
| `contextWindowCompression.slidingWindow` | racine | sessions longues |
| `sessionResumption` | racine | reprise après reset de connexion |
| `tools` | racine | function calling |

> **Incohérence de doc à connaître :** le tutoriel WebSocket montre `responseModalities` à la racine du setup, alors que la référence API et le document de découverte le placent dans `generationConfig` ([tutoriel](https://ai.google.dev/gemini-api/docs/live-api/get-started-websocket), [référence](https://ai.google.dev/api/live), [découverte](https://generativelanguage.googleapis.com/$discovery/rest?version=v1beta)). **Utiliser la forme `generationConfig`**, qui est celle du schéma machine.

### 3.4 Voix et langue

- Les modèles à audio natif choisissent automatiquement la langue ; le code de langue n'est pas censé être forcé ([capacités](https://ai.google.dev/gemini-api/docs/live-api/capabilities)).
- Pour imposer le français, le plus robuste est l'instruction système : « Réponds toujours en français, sans exception » ([bonnes pratiques](https://ai.google.dev/gemini-api/docs/live-api/best-practices)).
- La voix se choisit avec `speechConfig.voiceConfig.prebuiltVoiceConfig.voiceName`, par exemple `Kore` ([capacités](https://ai.google.dev/gemini-api/docs/live-api/capabilities)).
- La liste des voix disponibles est celle des modèles TTS, visible dans AI Studio ([capacités](https://ai.google.dev/gemini-api/docs/live-api/capabilities)).
- Incohérence de doc : la vue d'ensemble annonce « 70 langues » alors que la page capacités en liste **99** ([vue d'ensemble](https://ai.google.dev/gemini-api/docs/live), [capacités](https://ai.google.dev/gemini-api/docs/live-api/capabilities)). Le français (`fr`) est présent dans les deux cas.

### 3.5 VAD

- Le VAD automatique est actif par défaut ([capacités](https://ai.google.dev/gemini-api/docs/live-api/capabilities)).
- `silenceDurationMs` recommandé : **500 à 800 ms** ; la valeur interne par défaut est d'environ 800 ms. Trop bas, la parole est coupée ; trop haut, la latence augmente ([capacités](https://ai.google.dev/gemini-api/docs/live-api/capabilities)).
- `prefixPaddingMs` évite de couper la première syllabe ([capacités](https://ai.google.dev/gemini-api/docs/live-api/capabilities)).
- En cas d'interruption, le serveur envoie `serverContent.interrupted: true` ; le client doit jeter son buffer audio de lecture ([bonnes pratiques](https://ai.google.dev/gemini-api/docs/live-api/best-practices)).
- Si le micro est coupé plus d'une seconde, envoyer `audioStreamEnd` pour vider l'audio en cache ([capacités](https://ai.google.dev/gemini-api/docs/live-api/capabilities)).

### 3.6 Payload OpenCode prêt à l'emploi

Le champ `session` est un `Record<string, unknown>` transmis tel quel dans `bidiGenerateContentSetup` ([protocole](https://github.com/anomalyco/opencode/blob/863ea413649484b5761cd0c5c0e54c559a5c8b52/packages/protocol/src/groups/voice.ts#L72-L84)). Ne pas y mettre `model` : OpenCode l'écrase par `models/gemini-3.8-live` ([`realtime.ts#L105`](https://github.com/anomalyco/opencode/blob/863ea413649484b5761cd0c5c0e54c559a5c8b52/packages/ai/src/experimental/realtime.ts#L105)).

```json
{
  "model": "gemini-3.8-live",
  "transport": { "type": "websocket" },
  "session": {
    "generationConfig": {
      "responseModalities": ["AUDIO"],
      "speechConfig": {
        "voiceConfig": {
          "prebuiltVoiceConfig": { "voiceName": "Kore" }
        }
      }
    },
    "systemInstruction": {
      "parts": [
        {
          "text": "Tu es le conductor vocal d'un développeur. Tu parles toujours français, de façon concise et naturelle. Tu confirmes brièvement les actions risquées avant de les déléguer à une session OpenCode. Tu ne prétends jamais avoir fait une action que tu n'as pas déléguée."
        }
      ]
    },
    "realtimeInputConfig": {
      "automaticActivityDetection": {
        "disabled": false,
        "startOfSpeechSensitivity": "START_SENSITIVITY_HIGH",
        "endOfSpeechSensitivity": "END_SENSITIVITY_HIGH",
        "prefixPaddingMs": 100,
        "silenceDurationMs": 700
      },
      "activityHandling": "START_OF_ACTIVITY_INTERRUPTS"
    },
    "inputAudioTranscription": { "languageCodes": ["fr-FR"] },
    "outputAudioTranscription": {},
    "contextWindowCompression": { "slidingWindow": {} },
    "sessionResumption": {}
  }
}
```

Après l'appel, la réponse a cette forme (schéma `VoiceSessionResponse`, transport `websocket`) :

```json
{
  "data": {
    "transport": {
      "type": "websocket",
      "url": "wss://generativelanguage.googleapis.com/ws/google.ai.generativelanguage.v1beta.GenerativeService.BidiGenerateContentConstrained",
      "token": "<jeton éphémère>",
      "expiresAt": 1790000060
    }
  }
}
```

Le client se connecte ensuite à :

```
${url}?access_token=${encodeURIComponent(token)}
```

puis envoie un premier message de setup. Comme la configuration est verrouillée dans le jeton, un setup minimal suffit :

```json
{ "setup": { "model": "models/gemini-3.8-live" } }
```

Puis il envoie l'audio :

```json
{
  "realtimeInput": {
    "audio": {
      "data": "<base64 PCM 16 kHz>",
      "mimeType": "audio/pcm;rate=16000"
    }
  }
}
```

> **Point non testé en réel :** que Google accepte un setup minimal quand `bidiGenerateContentSetup` est verrouillé. La référence dit que le setup de connexion est ignoré, mais le protocole exige quand même un premier message `setup` ([AuthToken](https://ai.google.dev/api/live#authtoken), [guide WebSocket](https://ai.google.dev/gemini-api/docs/live-api/get-started-websocket)). À valider avec une vraie clé ; en cas d'erreur, renvoyer le setup complet.

---

## 4. Forfait gratuit : suffisant ou pas ?

### Ce que le free tier couvre

Le tableau de prix liste explicitement **Gemini 3.8 Live** avec, en free tier :

- **Input : Free of charge**
- **Output : Free of charge**
- Grounding with Google Search : supporté
- **« Used to improve our products » : Yes** en free tier, **No** en paid

Source : [Gemini Developer API pricing](https://ai.google.dev/gemini-api/docs/pricing).

La clé se crée dans Google AI Studio, sans facturation ; la France est dans la liste des régions servies ([clés API](https://ai.google.dev/gemini-api/docs/api-key), [régions](https://ai.google.dev/gemini-api/docs/available-regions)).

### Limites structurelles (indépendantes du quota)

| Limite | Valeur | Source |
|---|---|---|
| Session audio seule, sans compression | **15 minutes** | [Session management](https://ai.google.dev/gemini-api/docs/live-api/session-management) |
| Session audio + vidéo, sans compression | **2 minutes** | [Session management](https://ai.google.dev/gemini-api/docs/live-api/session-management) |
| Durée d'une connexion WebSocket | **~10 minutes**, puis `GoAway` | [Session management](https://ai.google.dev/gemini-api/docs/live-api/session-management) |
| Session avec compression de contexte | **illimitée** (fenêtre glissante) | [Session management](https://ai.google.dev/gemini-api/docs/live-api/session-management) |
| Contexte | **128 000 tokens** (modèles audio natifs) | [Capacités](https://ai.google.dev/gemini-api/docs/live-api/capabilities) |
| Sortie | **65 536 tokens** | [Fiche Gemini 3.8 Live](https://ai.google.dev/gemini-api/docs/models/gemini-3.8-live) |
| Audio | **~25 tokens par seconde** (~1 500/min) | [Bonnes pratiques](https://ai.google.dev/gemini-api/docs/live-api/best-practices) |
| Jeton éphémère | 1 usage, démarrage dans **1 min**, messages pendant **30 min**, max **20 h** | [Jeton éphémère](https://ai.google.dev/gemini-api/docs/live-api/ephemeral-tokens), [AuthToken](https://ai.google.dev/api/live#authtoken) |
| Reprise de session | handle valide **2 h** après la fin de la session | [Session management](https://ai.google.dev/gemini-api/docs/live-api/session-management) |

### Requêtes par minute : la réponse honnête

**Les chiffres RPM / TPM / RPD du free tier ne sont pas publiés dans la doc.** Google indique que :

- les limites sont mesurées en **RPM**, **TPM** et **RPD** ;
- elles s'appliquent **par projet**, pas par clé ;
- les RPD se réinitialisent à minuit heure du Pacifique ;
- les modèles **experimental/preview** ont des limites plus restrictives ;
- les valeurs exactes sont visibles dans **AI Studio, page Rate limits**, après connexion au compte.

Sources : [Rate limits](https://ai.google.dev/gemini-api/docs/rate-limits). La page AI Studio exige un compte Google (testé : redirection vers `accounts.google.com`), donc **impossible de relever les valeurs exactes sans la clé/compte**.

### Verdict

- **Pour un POC et des sessions courtes : oui, le free tier suffit.** Une clé AI Studio gratuite permet d'ouvrir une session Gemini 3.8 Live, de parler français, et de valider le conductor.
- **Pour l'usage cible (conduite, sessions longues, quotidiennes) : non, pas en l'état.** Il faut mesurer le quota réel, et probablement prévoir un fallback payant. À titre d'ordre de grandeur, en paid tier, l'audio revient à **$0.005/min en entrée + $0.018/min en sortie**, soit environ **$0.023/min (~$1.38/h)** ([pricing](https://ai.google.dev/gemini-api/docs/pricing)).
- **Le vrai plafond technique n'est pas le free tier mais la session de 15 min** : sans `contextWindowCompression`, la session meurt au bout de 15 minutes, même avec un quota infini. OpenCode transmet la config mais ne l'ajoute pas lui-même ; **c'est au harness de fournir `contextWindowCompression` et `sessionResumption`**.

---

## 5. Risques

1. **Confidentialité.** En free tier, Google indique que le contenu est utilisé pour améliorer ses produits ; en paid tier, non ([pricing](https://ai.google.dev/gemini-api/docs/pricing)). Pour un développeur qui dicte du code propriétaire, c'est un risque à arbitrer explicitement. Le paid tier est le seul mode compatible avec du code sensible.
2. **Preview.** L'API Live est marquée **Preview** ; les modèles, champs et limites peuvent changer ([capacités](https://ai.google.dev/gemini-api/docs/live-api/capabilities)). `gemini-3.8-live` est récent (septembre 2026) et `proactive_audio` y est permanent : envoyer `proactiveAudio: false` provoque une erreur ([fiche modèle](https://ai.google.dev/gemini-api/docs/models/gemini-3.8-live)). `thinkingLevel` n'est pas supporté sur `gemini-3.8-live` ([fiche modèle](https://ai.google.dev/gemini-api/docs/models/gemini-3.8-live)).
3. **Coupure à 15 min / reset à 10 min.** Sans compression + reprise, le conductor perd le contexte en pleine conversation. Il faut écouter `GoAway`, conserver le handle de reprise et reconnecter ([session management](https://ai.google.dev/gemini-api/docs/live-api/session-management)).
4. **Quota free tier inconnu.** Un `429 RESOURCE_EXHAUSTED` peut arriver sans prévenir ; le harness doit avoir un fallback (texte, autre provider, message vocal d'excuse) et ne pas bloquer la session OpenCode en cours ([rate limits](https://ai.google.dev/gemini-api/docs/rate-limits)).
5. **`expiresAt` trompeur.** OpenCode renvoie 60 s alors que le jeton vit 30 min. Une logique naïve de reconnexion toutes les 60 s gaspille des jetons et peut échouer ; il faut distinguer « fenêtre de démarrage » et « durée des messages » ([code](https://github.com/anomalyco/opencode/blob/863ea413649484b5761cd0c5c0e54c559a5c8b52/packages/ai/src/experimental/realtime.ts#L94-L115), [AuthToken](https://ai.google.dev/api/live#authtoken)).
6. **Pas de client audio dans OpenCode.** Les packages publics inspectés contiennent le broker, les routes et le format de délégation vocale, mais pas d'implémentation de capture/lecture audio temps réel ([arbre `dev`](https://github.com/anomalyco/opencode/tree/dev/packages), [`voice-delegation.ts`](https://github.com/anomalyco/opencode/blob/dev/packages/app/src/session/timeline/voice-delegation.ts)). Le harness devra l'écrire.
7. **Pas de session réelle testée.** La clé absente empêche de vérifier : acceptation du payload par Google, latence sur N100, qualité audio en français, quota réel, comportement de `bidiGenerateContentSetup` verrouillé, et reprise après `GoAway`.
8. **Clés AI Studio « auth keys ».** Depuis mai 2026, les nouvelles clés sont des auth keys liées à un compte de service ; une clé non restreinte peut être rejetée ([clés API](https://ai.google.dev/gemini-api/docs/api-key)). Restreindre la clé à la Generative Language API et la stocker via l'intégration, pas dans un fichier de config versionné.

---

## 6. Ce qui reste à tester (protocole exact)

### Étape 1 — Créer et stocker la clé

Créer une clé dans [AI Studio](https://aistudio.google.com/apikey), la restreindre à la Generative Language API, puis :

```bash
export GEMINI_API_KEY='...'

opencode api post /api/integration/google/connect/key \
  -d "{\"key\":\"$GEMINI_API_KEY\"}"

opencode api get /api/integration/google | jq '.data.connections'
# doit contenir {"type":"credential","method":"key",...}
```

> Sur un serveur qui vient de démarrer, le premier appel peut répondre `404 IntegrationNotFoundError` le temps que la location se charge ; relancer l'appel. La commande met la clé dans l'historique du shell : préférer `/connect` dans le TUI quand c'est possible.

Alternative pour un test isolé : lancer un serveur avec `GEMINI_API_KEY` dans l'environnement.

### Étape 2 — Vérifier la disponibilité

Sur `v2.1.2`, `/models` répondra toujours 404 : c'est attendu. Vérifier plutôt la session :

```bash
cat > /tmp/voice-session.json <<'JSON'
{
  "model": "gemini-3.8-live",
  "transport": { "type": "websocket" },
  "session": {
    "generationConfig": {
      "responseModalities": ["AUDIO"],
      "speechConfig": {
        "voiceConfig": { "prebuiltVoiceConfig": { "voiceName": "Kore" } }
      }
    },
    "systemInstruction": {
      "parts": [{ "text": "Tu parles toujours français, de façon concise." }]
    },
    "contextWindowCompression": { "slidingWindow": {} },
    "sessionResumption": {},
    "outputAudioTranscription": {}
  }
}
JSON

opencode api post /api/experimental/voice/session \
  -d "$(cat /tmp/voice-session.json)" | tee /tmp/voice-session-response.json
```

Attendu : `200` avec `data.transport.token`, `url`, `expiresAt`. Si `503 API key not valid`, la clé est mauvaise ou mal restreinte. Si `400`, le payload est refusé (commencer par retirer les champs optionnels).

### Étape 3 — Ouvrir le WebSocket et envoyer de l'audio

Node 26 fournit `WebSocket` nativement. Exemple de harnais minimal :

```bash
TOKEN=$(jq -r '.data.transport.token' /tmp/voice-session-response.json)
URL=$(jq -r '.data.transport.url' /tmp/voice-session-response.json)

TOKEN="$TOKEN" URL="$URL" node --input-type=module <<'JS'
const ws = new WebSocket(`${process.env.URL}?access_token=${encodeURIComponent(process.env.TOKEN)}`)
ws.onopen = () => {
  console.log('open')
  ws.send(JSON.stringify({ setup: { model: 'models/gemini-3.8-live' } }))
}
ws.onmessage = (event) => console.log('message', event.data.toString().slice(0, 500))
ws.onerror = (event) => console.error('error', event.message)
ws.onclose = (event) => console.log('close', event.code, event.reason)
setTimeout(() => ws.close(), 30_000)
JS
```

Attendu : un message `setupComplete`, puis les réponses audio/transcriptions. Ensuite seulement, brancher un fichier PCM 16 kHz ou le micro.

### Étape 4 — Mesurer les limites réelles

1. **Quota** : avant/après une session, ouvrir [AI Studio → Rate limits](https://aistudio.google.com/rate-limit) et relever RPM/TPM/RPD pour `gemini-3.8-live`. Noter aussi `usageMetadata` renvoyé par la session ([capacités](https://ai.google.dev/gemini-api/docs/live-api/capabilities)).
2. **Durée** : lancer une session volontairement longue **sans** compression pour confirmer la coupure à 15 min ; puis **avec** `contextWindowCompression.slidingWindow` pour vérifier la prolongation ([session management](https://ai.google.dev/gemini-api/docs/live-api/session-management)).
3. **Reset de connexion** : attendre ou provoquer un `GoAway`, conserver `sessionResumptionUpdate.newHandle`, reconnecter avec `sessionResumption.handle` et vérifier que le contexte survit ([session management](https://ai.google.dev/gemini-api/docs/live-api/session-management)).
4. **Français** : vérifier que la voix répond en français sans accent anglais et que la transcription d'entrée est correcte ; sinon renforcer l'instruction système.
5. **Latence N100** : mesurer le temps entre la fin de parole (VAD) et le premier chunk audio de réponse. Le GPU n'est pas nécessaire (l'inférence est distante), mais le rééchantillonnage et la lecture PCM doivent tenir 4 cœurs.
6. **`/models`** : refaire le test après une release contenant #57194, ou sur un build `dev`. Attendu : `200` avec `gemini-3.8-live.available: true`.

### Étape 5 — Test direct du jeton Google (sans OpenCode)

Pour isoler un problème OpenCode d'un problème Google :

```bash
EXP=$(date -u -d '+30 minutes' +%Y-%m-%dT%H:%M:%SZ)
NEW=$(date -u -d '+1 minute' +%Y-%m-%dT%H:%M:%SZ)

cat > /tmp/auth-token.json <<JSON
{
  "uses": 1,
  "expireTime": "$EXP",
  "newSessionExpireTime": "$NEW",
  "bidiGenerateContentSetup": {
    "model": "models/gemini-3.8-live",
    "generationConfig": { "responseModalities": ["AUDIO"] }
  }
}
JSON

curl -sS -X POST \
  'https://generativelanguage.googleapis.com/v1beta/auth_tokens' \
  -H "x-goog-api-key: $GEMINI_API_KEY" \
  -H 'Content-Type: application/json' \
  --data @/tmp/auth-token.json | jq .
```

Attendu : `{"name":"..."}`. Puis se connecter à `BidiGenerateContentConstrained?access_token=<name>`.

---

## 7. Incertitudes restantes

- **Quotas free tier exacts (RPM/TPM/RPD)** : non publiés, non mesurables sans compte Google connecté. C'est la principale inconnue pour décider si le free tier suffit à l'usage cible.
- **Session Gemini réelle** : aucun test audio de bout en bout, faute de clé. Le payload est conforme aux schémas, mais son acceptation par le modèle n'est pas prouvée.
- **Setup minimal avec jeton verrouillé** : la doc dit que le setup de connexion est ignoré, mais le protocole exige un premier `setup`. Comportement exact à vérifier.
- **`responseModalities` racine vs `generationConfig`** : la doc se contredit. Le schéma machine donne `generationConfig` ; c'est la forme recommandée ici, mais un test réel trancherait.
- **`gemini-3.8-live-extended-thinking` via OpenCode** : le matcher par préfixe pourrait l'accepter, mais il n'est ni listé ni testé. Ne pas s'appuyer dessus pour la spec.
- **Prochaine release OpenCode** : la route `/models` est sur `dev` depuis le 21 octobre 2026, mais aucune version publiée ne la contient à la date de cette note. La version qui l'embarque n'est pas connue.
- **Latence et charge N100** : non mesurées ; l'absence de GPU n'est pas bloquante en théorie (audio I/O local, inférence distante), mais rien ne le prouve ici.
- **Clé « auth key » vs clé standard** : le chemin de stockage a été validé avec une clé factice, pas avec une vraie auth key liée à un compte de service.

---

## Sources

### OpenCode

- **S1.** Dépôt public, branche `dev`, commit `863ea413649484b5761cd0c5c0e54c559a5c8b52` (2026-10-21) : <https://github.com/anomalyco/opencode/tree/863ea413649484b5761cd0c5c0e54c559a5c8b52>
- **S2.** `packages/ai/src/experimental/realtime.ts` : <https://github.com/anomalyco/opencode/blob/863ea413649484b5761cd0c5c0e54c559a5c8b52/packages/ai/src/experimental/realtime.ts>
- **S3.** `packages/core/src/voice.ts` : <https://github.com/anomalyco/opencode/blob/863ea413649484b5761cd0c5c0e54c559a5c8b52/packages/core/src/voice.ts>
- **S4.** `packages/protocol/src/groups/voice.ts` : <https://github.com/anomalyco/opencode/blob/863ea413649484b5761cd0c5c0e54c559a5c8b52/packages/protocol/src/groups/voice.ts>
- **S5.** `packages/server/src/handlers/voice.ts` : <https://github.com/anomalyco/opencode/blob/863ea413649484b5761cd0c5c0e54c559a5c8b52/packages/server/src/handlers/voice.ts>
- **S6.** Tag `v2.1.2` : <https://github.com/anomalyco/opencode/blob/v2.1.2/packages/protocol/src/groups/voice.ts>
- **S7.** Commit d'ajout de `/models`, PR #57194 : <https://github.com/anomalyco/opencode/commit/79fd806343db0de150f23b09e4b9199434d6779c>
- **S8.** OpenAPI public (suit `dev`) : <https://opencode.ai/v2/openapi.json>
- **S9.** Référence API OpenCode, groupe `voice` : <https://opencode.ai/v2/docs/api/>
- **S10.** Format de délégation vocale côté app : <https://github.com/anomalyco/opencode/blob/dev/packages/app/src/session/timeline/voice-delegation.ts>

### Google Gemini Live

- **G1.** Vue d'ensemble Live API : <https://ai.google.dev/gemini-api/docs/live>
- **G2.** Guide WebSocket : <https://ai.google.dev/gemini-api/docs/live-api/get-started-websocket>
- **G3.** Capacités Live API (VAD, voix, langues, limites) : <https://ai.google.dev/gemini-api/docs/live-api/capabilities>
- **G4.** Gestion de session (15 min, 10 min, compression, reprise) : <https://ai.google.dev/gemini-api/docs/live-api/session-management>
- **G5.** Jetons éphémères : <https://ai.google.dev/gemini-api/docs/live-api/ephemeral-tokens>
- **G6.** Pricing (free tier Live) : <https://ai.google.dev/gemini-api/docs/pricing>
- **G7.** Rate limits : <https://ai.google.dev/gemini-api/docs/rate-limits>
- **G8.** Fiche Gemini 3.8 Live : <https://ai.google.dev/gemini-api/docs/models/gemini-3.8-live>
- **G9.** Référence API Live et AuthToken : <https://ai.google.dev/api/live>
- **G10.** Document de découverte v1beta : <https://generativelanguage.googleapis.com/$discovery/rest?version=v1beta>
- **G11.** Bonnes pratiques Live : <https://ai.google.dev/gemini-api/docs/live-api/best-practices>
- **G12.** Clés API Gemini : <https://ai.google.dev/gemini-api/docs/api-key>
- **G13.** Régions disponibles : <https://ai.google.dev/gemini-api/docs/available-regions>

### Preuves locales (reproductibles)

- **L1.** `opencode --version` → `opencode v2.1.2`.
- **L2.** `opencode api get /api/experimental/voice/models` → `HTTP 404 Not Found`.
- **L3.** `opencode api get /openapi.json | jq ...` → seule route voice : `/api/experimental/voice/session`.
- **L4.** `opencode api post /api/experimental/voice/session` sans clé → `503` « Voice needs a Google API key… ».
- **L5.** Même appel avec `GEMINI_API_KEY=dummy-invalid-key` → `503` « API key not valid ».
- **L6.** `opencode api post /api/integration/google/connect/key` avec clé factice → `204`, puis la connexion apparaît dans `/api/integration/google`.
