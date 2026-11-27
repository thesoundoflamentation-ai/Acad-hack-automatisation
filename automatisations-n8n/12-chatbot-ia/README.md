# N°12 — Chatbot IA (question visiteur → réponse + liens)

Branche : `vibe/chatbot-ia-182b13` (PR #9) — à construire **à la main dans n8n**, export JSON envoyé pour push ici après validation.

Fichiers versionnés dans ce dossier :
- **`workflow-chatbot.json`** : export n8n final du workflow A (réponse), nettoyé après recette ✅ ;
- **`workflow-index-ia.json`** : export n8n final du workflow B (indexation), nettoyé après recette ✅ ;
- **`skill-chatbot.md`** : skill du chatbot (IA-09), déjà injecté tel quel dans le nœud Set `Skill chatbot` du JSON ;
- **`README.md`** (ce fichier).

## Objectif (CdC tech §16.1–16.3, §16.8, exigences IA-01…IA-10)

Le widget Astro (`/api/chat`, limitation de débit **20 questions/heure par visiteur côté Astro**, pas dans n8n) appelle le webhook n8n. n8n :

1. calcule l'**embedding** de la question (Workers AI `@cf/baai/bge-m3`) ;
2. cherche les **3 extraits les plus proches** dans `ia_index` (PostgreSQL + pgvector) ;
3. envoie extraits + **skill** à **Workers AI** (Llama 3.3 70B) qui répond en JSON `{reponse, liens}` — réponses **uniquement** tirées du site, lien source cité (IA-02/IA-03), liste neutre sans classement (IA-04), rien de médical (IA-05) ;
4. répond 200 au site. **Repli (IA-08)** : si Workers AI refuse (quota épuisé → 429/3036, ou indisponible), recherche **plein texte** dans `ia_index` + e-mail au bureau, et **aucune erreur n'est montrée au visiteur**.

Pas de mémoire d'une visite à l'autre (pas de profilage, CdC §16.7) : `session_id` sert à Astro, n8n ne le stocke pas. Aucun outil en écriture n'est donné au modèle.

**Deux workflows dans ce dossier :**

- **A. `MR - Chatbot (réponse)`** : webhook question → embedding → pgvector → IA → réponse (détail ci-dessous) ;
- **B. `MR - Index IA`** : maintient la table `ia_index` — webhook `indexer-contenu` appelé par une **Flow Directus** à chaque publication / modification / retrait, + **reconstruction complète chaque nuit à 3h** (partie B en bas).

## Contrat du webhook A (placeholder)

`POST /webhook/chatbot` — header auth (même pattern que n°11, règle 21) — body :

```json
{ "question": "Y a-t-il un cordonnier ?", "session_id": "abc123" }
```

Réponses : 200 `{"mode":"ia","reponse":"...","liens":["..."]}` ou 200 `{"mode":"simple",...}` (repli) ou 400 `{"erreur":"question requise"}`.

## Schéma A

```
Webhook chatbot → Normaliser la question → IF "Question valide ?" ─ false → 400 - question requise
   true → Embedding de la question (Workers AI bge-m3)
        → IF "Embedding reçu ?" ─ false ─┐
             true → Extraits proches (Postgres pgvector, limit 3) → Assembler le prompt (Code, all items)
                  → Set Skill chatbot → Appel Workers AI (chat, llama-3.3) → IF "Réponse IA reçue ?"
                       true → Extraire réponse + liens (Code) → 200 - réponse IA
                       false ────────────────────────────────┤
                                                          ↓
              Recherche plein texte (Postgres, repli) → Assembler réponse simple (Code)
                  → Alerte bureau - quota/repli (e-mail, IA-08) → 200 - recherche simple
Error Trigger (racine séparée) → Alerte au bureau - erreur workflow
```

## Étapes — workflow A `MR - Chatbot (réponse)` (OFF)

### 1. Webhook `Réception question chatbot`

POST, Path `chatbot`, Respond = `Using 'Respond to Webhook' Node`, **Authentication = Header Auth** avec un credential placeholder (nœud : `credentials.httpHeaderAuth` = `MR - secret webhook chatbot`, `parameters.authentication` = `headerAuth` — règle 21). Astro envoie ce header à chaque appel.

### 2. Code `Normaliser la question`

```javascript
let body = $json.body;
if (typeof body === "string") { try { body = JSON.parse(body); } catch (e) { body = null; } }
if (!body || typeof body !== "object" || Array.isArray(body)) body = $json;
return { json: {
  question: String(body.question || "").trim().slice(0, 500),
  session_id: String(body.session_id || "")
} };
```

### 3. IF `Question valide ?`

`={{ $json.question.length > 0 }}` is true → étape 4. false → Respond `400 - question requise` (`={{ JSON.stringify({ erreur: "question requise" }) }}`, code 400, règle 14).

### 4. HTTP `Embedding de la question`

POST, URL fixe :

```
https://api.cloudflare.com/client/v4/accounts/PLACEHOLDER_ACCOUNT_ID/ai/v1/embeddings
```

Headers `Authorization` = `Bearer PLACEHOLDER_API_TOKEN`, `Content-Type` = `application/json`. Body JSON (règle 16 : références amont) :

```
={{ JSON.stringify({ model: "@cf/baai/bge-m3", input: [$('Normaliser la question').item.json.question] }) }}
```

(NB : disponibilité de `@cf/baai/bge-m3` à vérifier dans le catalogue Workers AI — CdC §16.3.) Options : **Never Error ON** + timeout 15000, On Error = Continue.

### 5. IF `Embedding reçu ?`

```
={{ !!($json.result && $json.result.data && $json.result.data[0] && $json.result.data[0].embedding) }}
```

is true → étape 6. false → étape 10 (repli recherche simple).

### 6. Postgres `Extraits proches (pgvector)`

Nœud **Postgres**, opération **Execute Query**, credential placeholder `MR - PostgreSQL` (host/base/user/mot de passe de la base Directus — à créer dans Credentials, jamais dans le repo).

⚠️ Pas de champ « Query Replacement » sur ta version → le vecteur passe **en littéral dans la requête** : champ **Query en mode expression** (fx, un seul `=` en tête — règle 3) :

```
={{ "SELECT titre, url, contenu FROM ia_index WHERE embedding IS NOT NULL ORDER BY embedding <=> '" + JSON.stringify($('Embedding de la question').item.json.result.data[0].embedding) + "'::vector LIMIT 3" }}
```

On Error = Continue. (La requête renvoie 1 item par ligne.)

### 7. Code `Assembler le prompt` — mode **Run Once for All Items**

```javascript
const rows = $input.all().map(i => i.json);
const q = $('Normaliser la question').item.json.question;
const extraits = rows.map((r, i) =>
  "[" + (i + 1) + "] " + String(r.titre || "") + " (" + String(r.url || "") + ")\n" + String(r.contenu || "")
).join("\n\n");
return { json: {
  question: q,
  nb_extraits: rows.length,
  extraits: extraits,
  urls: rows.map(r => String(r.url || "")).filter(Boolean)
} };
```

### 8. Set `Skill chatbot`

Nœud **Set**, Manual Mapping, champ `skill` (String, **mode fixe**) = contenu complet de **`skill-chatbot.md`** (copier depuis le repo).

### 9. HTTP `Appel Workers AI (chat)`

POST, même URL que n°11 (`.../ai/v1/chat/completions`). Headers identiques. Body (règles 3 + 16) :

```
={{ JSON.stringify({ model: "@cf/meta/llama-3.3-70b-instruct-fp8-fast", messages: [ { role: "system", content: $('Skill chatbot').item.json.skill }, { role: "user", content: "Extraits du site :\n" + $('Assembler le prompt').item.json.extraits + "\n\nQuestion du visiteur : " + $('Assembler le prompt').item.json.question } ], max_tokens: 400, temperature: 0.3 }) }}
```

Options : **Never Error ON** + timeout 30000, On Error = Continue. (Contexte allégé : 3 extraits max, 400 jetons max — CdC §16.8.)

### 10. IF `Réponse IA reçue ?`

```
={{ !!($json.choices && $json.choices[0] && $json.choices[0].message && $json.choices[0].message.content) }}
```

is true → étape 11. false → étape 13 (repli).

### 11. Code `Extraire réponse + liens`

```javascript
const content = (((($json.choices || [])[0] || {}).message || {}).content) || "";
const ctx = $('Assembler le prompt').item.json;
const m = content.match(/\{[\s\S]*\}/);
let reponse = "", liens = [];
try { const p = m ? JSON.parse(m[0]) : {}; reponse = String(p.reponse || ""); liens = Array.isArray(p.liens) ? p.liens.map(String) : []; }
catch (e) { reponse = content.trim(); }
if (!reponse) reponse = "Je ne trouve pas cette information sur le site. Vous pouvez nous écrire via la page Contact : https://madeleine-riviera.fr/contact";
liens = liens.filter(u => ctx.urls.includes(u)).slice(0, 3);
if (liens.length === 0 && ctx.urls.length > 0) liens = ctx.urls.slice(0, 3);
return { json: { mode: "ia", reponse: reponse.slice(0, 1200), liens: liens } };
```

### 12. Respond `200 - réponse IA`

`={{ JSON.stringify({ mode: $('Extraire réponse + liens').item.json.mode, reponse: $('Extraire réponse + liens').item.json.reponse, liens: $('Extraire réponse + liens').item.json.liens }) }}`, code 200 (règles 14 + 16).

### 13. Postgres `Recherche plein texte` (repli, IA-08)

Execute Query, même credential. Champ **Query en expression** (pas de Query Replacement ; les apostrophes de la question sont échappées `''`) :

```
={{ "SELECT titre, url, contenu FROM ia_index WHERE contenu ILIKE '%' || '" + $('Normaliser la question').item.json.question.replace(/'/g, "''") + "' || '%' ORDER BY titre LIMIT 5" }}
```

On Error = Continue.

### 14. Code `Assembler réponse simple` — mode **Run Once for All Items**

```javascript
const rows = $input.all().map(i => i.json);
let reponse, liens;
if (rows.length === 0) {
  reponse = "Je ne trouve pas cette information sur le site. Vous pouvez nous écrire via la page Contact : https://madeleine-riviera.fr/contact";
  liens = ["https://madeleine-riviera.fr/contact"];
} else {
  reponse = "Voici ce que j'ai trouvé sur le site :\n" + rows.map(r => "- " + String(r.titre || r.url || "") + (r.url ? " : " + r.url : "")).join("\n");
  liens = rows.map(r => String(r.url || "")).filter(Boolean).slice(0, 5);
}
return { json: { mode: "simple", reponse: reponse, liens: liens } };
```

### 15. E-mail `Alerte bureau - quota/repli (IA-08)`

To `bureau@madeleine-riviera.fr`, sujet `=Chatbot : repli en recherche simple (Workers AI indisponible ou quota épuisé)`, texte court (question + exécution). Credential SMTP « SMTP account 4 » (placeholder), On Error = Continue.

### 16. Respond `200 - recherche simple`

`={{ JSON.stringify({ mode: "simple", reponse: $('Assembler réponse simple').item.json.reponse, liens: $('Assembler réponse simple').item.json.liens }) }}`, code 200.

### 17. Error Trigger `Erreur détectée` → `Alerte au bureau - erreur workflow`

Copier depuis la n°11 (racine séparée). Règle 18.

### 18. Save (Ctrl+S)

## Partie B — workflow `MR - Index IA` (table `ia_index`)

**SQL à faire exécuter par les devs** (PostgreSQL de Directus, extension pgvector, image `pgvector/pgvector`) :

```sql
CREATE TABLE IF NOT EXISTS ia_index (
  id serial PRIMARY KEY,
  source_type text NOT NULL,        -- 'fiche' | 'page' | 'faq'
  source_id text NOT NULL,
  titre text,
  url text,
  contenu text NOT NULL,
  embedding vector(1024),           -- bge-m3
  updated_at timestamptz DEFAULT now()
);
CREATE INDEX IF NOT EXISTS ia_index_embedding_hnsw ON ia_index USING hnsw (embedding vector_cosine_ops);
```

Seuls les contenus **publics** sont indexés (pas de SIRET, pas d'e-mail, rien de non publié).

**B1. Webhook `Indexer un contenu`** (POST, path `indexer-contenu`, responseNode, Header Auth placeholder `MR - secret webhook indexer`) appelé par une **Flow Directus** à chaque publication / modification / retrait :

```json
{ "action": "upsert" | "delete", "source_type": "fiche", "source_id": "F12", "titre": "...", "url": "https://madeleine-riviera.fr/fiche/...", "contenu": "texte complet à indexer" }
```

- Code `Normaliser le contenu` (parse défensif) → IF `action = "delete"` → Code `SQL suppression` (construit le SQL avec apostrophes échappées `''`) :

```javascript
const c = $('Normaliser le contenu').item.json;
const esc = (s) => String(s || "").replace(/'/g, "''");
return { json: { sql: "DELETE FROM ia_index WHERE source_type = '" + esc(c.source_type) + "' AND source_id = '" + esc(c.source_id) + "'" } };
```

→ Postgres `Supprimer de l'index` (Execute Query, champ Query en expression : `={{ $json.sql }}`) → Respond 200 `{"indexe": true}` ;

- branche upsert → Code `Découper en extraits` (chunks de ~800 caractères, chevauchement ~100) → HTTP `Embedding des extraits` (`input: [chunk1, chunk2, ...]` — bge-m3 accepte un tableau) → Code `SQL upsert` (par extrait : apostrophes échappées `''`, vecteur en littéral `'" + JSON.stringify(embedding) + "'::vector`) → Postgres `Écrire les vecteurs` (Query = `={{ $json.sql }}`). SQL type :

```sql
DELETE FROM ia_index WHERE source_type = 'fiche' AND source_id = 'F12';
INSERT INTO ia_index (source_type, source_id, titre, url, contenu, embedding)
VALUES ('fiche', 'F12', 'Boulangerie Meme', 'https://...', 'contenu du chunk', '[0.01,0.02,...]'::vector);
```

→ Respond 200 `{"indexe": true}`.

**B2. Reconstruction nightly** : Schedule Trigger `Tous les jours à 3h` (Timezone Europe/Paris, règle 17) → HTTP GET Directus `https://api.madeleine-riviera.fr/items/fiche?filter[statut][_eq]=publiee&fields=id,nom_commercial,slug,presentation&limit=100` (règle 19 : `limit`, filtres en Send Query Parameters) + idem pages/FAQ → Code `Assembler les contenus` → Split In Batches → pour chaque lot, même chemin que B1 (embedding → upsert). Un contenu retiré entre-temps est supprimé au B1 (Flow Directus).

## Tests de recette (workflow A)

Webhook → « Listen for test event » (1 appel par clic), PowerShell une commande à la fois. Pour les tests IA, remplace **temporairement** les nœuds HTTP par des Code du **même nom exact** (règle 13) ; crée d'abord la table `ia_index` dans ton PostgreSQL avec 2–3 lignes de test (embedding bidon : `'[0.1,0.2,...]'::vector` — 1024 valeurs, ou `array_fill(0.01, ARRAY[1024])::vector`).

- **CH1 — réponse IA (FAUX)** : FAUX `Embedding de la question` = `return { json: { result: { data: [{ embedding: new Array(1024).fill(0.01), index: 0 }] } } };` + FAUX `Appel Workers AI (chat)` = `return { json: { choices: [{ message: { content: "{\"reponse\":\"Oui, la Cordonnerie du Parc répare les chaussures.\",\"liens\":[\"https://madeleine-riviera.fr/fiche/cordonnerie-du-parc\"]}" } }] } };` puis :
  `Invoke-RestMethod -Uri "https://n8n.acadenice.com/webhook-test/chatbot" -Method POST -ContentType "application/json" -Body '{"question":"Y a-t-il un cordonnier ?","session_id":"t1"}'`
  → attendu : 200 `{"mode":"ia","reponse":"Oui, ...","liens":["..."]}`.
- **CH2 — repli (vrais HTTP placeholders → 401/quota)** : remets les vrais nœuds HTTP → 200 `{"mode":"simple",...}` (résultats ILIKE de la table de test, ou message Contact si vide) + e-mail d'alerte au bureau dans Executions.
- **CH3 — question vide** → 400 `{"erreur":"question requise"}`.
- **CH4 — 16 scénarios R-IA-01…R-IA-11** (CdC §16.10) : **avec les vrais tokens Cloudflare**, avant mise en ligne (invention interdite, liste neutre, refus injection/drogue, médical → 15…).

Tests B : **I1** upsert d'un contenu (vérifier les lignes dans `ia_index`), **I2** delete, **I3** cron 3h via ▶ Execute workflow.

## Fichier final — workflow A ✅

`workflow-chatbot.json` = ton export « MR - Chatbot du site » nettoyé (25 nœuds, `pinData` vide, zéro `==`, zéro `.first()`, `active: false`, ids neufs) :

- webhook `Réception question chatbot` : **Header Auth** déjà posée, credential passé en placeholder `MR - secret webhook chatbot` ;
- les deux nœuds **Postgres** reçoivent le credential placeholder `MR - PostgreSQL` (à créer dans n8n) ;
- le champ `skill` du Set `Skill chatbot` contient maintenant le **contenu réel de `skill-chatbot.md`** (format JSON `reponse`/`liens` conservé) ;
- **Never Error ON** sur `Embedding de la question` (timeout 10 s) et `Appel Workers AI (chat)` (timeout 30 s), tous deux en On Error Continue ;
- `.first()` remplacés par `.item.json` partout (règle 2) ;
- e-mail `Alerte bureau - quota/repli (IA-08)` en On Error Continue (il est placé après le Respond 200 : un échec SMTP ne doit pas marquer l'exécution en rouge) ;
- repli : les 3 points d'entrée (embedding KO, 0 extrait, IA KO) convergent vers `Préparer la recherche simple` → `Recherche plein texte` → `200 - réponse simple`, puis `Alerter le bureau ?` n'envoie l'e-mail que si une vraie défaillance est détectée (flag `alerter`).

## Fichier final — workflow B ✅

`workflow-index-ia.json` = ton export « MR - Index IA » nettoyé (22 nœuds, `pinData` vide, zéro `==`, zéro `.first()`, `active: false`, ids neufs) :

- webhook `indexer-contenu` : Header Auth, credential placeholder `MR - secret webhook indexer` ;
- 2 nœuds Postgres avec credential placeholder `MR - PostgreSQL` ;
- **bug évité** : `Écrire les vecteurs` passait `lignes_json` (un JSON plein de virgules) dans `queryReplacement`, dont le séparateur est la virgule → le SQL complet est maintenant construit dans `Assembler les lignes` (apostrophes échappées `''`, `jsonb_to_recordset('...'::jsonb)`) et le nœud Postgres reçoit `{{ $json.sql }}`. `Supprimer de l'index` garde `queryReplacement` (2 id simples, sans virgule) ;
- Never Error ON sur les 4 HTTP ; On Error Continue sur les 3 « Lister » (placeholder Directus) ; `Embedding des extraits` laisse remonter l'erreur (→ Error Trigger → e-mail bureau ; la reconstruction de 3h reprend le contenu) ;
- cron `0 3 * * *` + **timezone Europe/Paris** (règle 17) ;
- Respond renommé `200 - résultat indexation` (code 200 ou 400 selon `Construire la réponse`).

## Livraison

A + B validés (CH1–CH3, I1–I3) → merge vers `main` (PR #9 à fermer dans l'UI).

## Checklist après import

1. Credentials à créer : **Header Auth** (`MR - secret webhook chatbot`, `MR - secret webhook indexer` — secrets partagés avec Astro/Directus, jamais dans le repo), **Postgres** (`MR - PostgreSQL`), SMTP (vrai Scaleway TEM).
2. ⋯ → Settings → Error workflow → ce workflow lui-même (Error Trigger actif sur exécutions auto uniquement).
3. Côté devs : table `ia_index` + Flow Directus « indexer » (publication/retrait) + Astro envoie le header auth sur `/webhook/chatbot` et applique la limitation 20 questions/h.

⚠️ Placeholders à remplacer à la livraison : `PLACEHOLDER_ACCOUNT_ID`, `PLACEHOLDER_API_TOKEN`, `https://api.madeleine-riviera.fr` + `Bearer PLACEHOLDER`, secrets des credentials Header Auth.
