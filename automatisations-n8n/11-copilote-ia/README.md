# N°11 — Copilote IA (correction du texte de la fiche)

Branche : `vibe/copilote-ia-182b13` — à construire **à la main dans n8n**, export JSON envoyé pour push ici après validation.

Fichiers versionnés dans ce dossier : **`skill-copilote.md`** (skill CP-09, à copier tel quel dans le nœud Skill) et **`mots-interdits-cp10.txt`** (liste du filtre CP-10, déjà intégrée dans le code ci-dessous).

## Objectif (CdC tech §5.3, §16.2–16.4, CP-01/CP-07/CP-09/CP-10)

Au clic sur « Valider » dans « Ma fiche », une Flow Directus appelle le webhook n8n avec le texte du commerçant. n8n :

1. envoie `texte_origine` + le **skill** à **Workers AI** (Llama 3.3 70B, offre gratuite, plafond dur 10 000 Neurons/jour) qui renvoie **2 versions** ≤ 300 caractères en JSON ;
2. applique le **filtre CP-10** (mots interdits, + complément Santé si catégorie Santé) : une version bloquée n'est pas montrée ;
3. écrit dans `proposition_fiche` (placeholder) : `version_ia_1`, `version_ia_2`, et `alerte_moderation` (message rouge si activité illégale détectée par le skill, ou « correction IA impossible » si les 2 versions sont bloquées) ;
4. répond 200 au site.

Le copilote **ne publie jamais rien** (CP-07) : le commerçant choisit `version_retenue`, puis modération habituelle (n°6).

## Contrat du webhook (placeholder)

`POST /webhook/copilote` — body : `{ "proposition_id": "P1", "texte_origine": "...", "categorie": "Boulangerie" }`
Réponse : 200 `{"recu":true,"nb_versions":2,"alerte":""}`.

## Schéma

```
Webhook copilote → Normaliser la demande → Skill copilote (Set) → Appel Workers AI (POST chat/completions)
→ IF "Réponse IA reçue ?" ─ false → PATCH alerte indisponible (7bis) → Respond 200 - IA indisponible (8bis)
   true → Parser + filtrer CP-10 (Code) → PATCH Écrire versions IA (7) → Respond 200 - versions IA (8)
Error Trigger (racine séparée) → Alerte au bureau - erreur workflow
```

## Étapes

Nouveau workflow **`MR - Copilote IA`**, OFF.

### 1. Webhook `Réception demande copilote`

POST, Path `copilote`, Respond = `Using 'Respond to Webhook' Node` (règle 10 : branche le Respond avant de tester).

### 2. Code `Normaliser la demande`

```javascript
let body = $json.body;
if (typeof body === "string") { try { body = JSON.parse(body); } catch (e) { body = null; } }
if (!body || typeof body !== "object" || Array.isArray(body)) body = $json;
return { json: {
  proposition_id: String(body.proposition_id || body.id || ""),
  texte_origine: String(body.texte_origine || body.texte || ""),
  categorie: String(body.categorie || "")
} };
```

### 3. Set `Skill copilote`

Nœud de type **Set** (« Edit Fields » / « Définir »). Mode **Manual Mapping** → « Add Field » → Name `skill`, Type **String**, Value = **contenu complet de `skill-copilote.md`** collé en texte brut (copier depuis le repo, ce fichier est la version de référence). ⚠️ le champ Value doit rester en mode **fixe** (pas d'icône fx / pas de `=` en tête — le skill commence par `#`, donc c'est bon) : une valeur commençant par `=` serait interprétée comme une expression. Ne mets PAS le skill dans un nœud Code avec des backticks (le skill contient des blocs ``` qui casseraient le template JS) — le Set en texte brut est plus sûr. En aval, on le référence par `$('Skill copilote').item.json.skill`. Astuce : le skill peut aussi être lu en live via un HTTP GET sur le raw GitHub — mais en MVP, en dur dans le Set, en restant synchronisé avec le fichier.

### 4. HTTP `Appel Workers AI`

POST, URL (fixe) :

```
https://api.cloudflare.com/client/v4/accounts/PLACEHOLDER_ACCOUNT_ID/ai/v1/chat/completions
```

Headers : `Authorization` = `Bearer PLACEHOLDER_API_TOKEN` (mode fixe), `Content-Type` = `application/json`. Body JSON (expression) — ⚠️ règle 16 : références amont, pas `$json` :

```
={{ JSON.stringify({ model: "@cf/meta/llama-3.3-70b-instruct-fp8-fast", messages: [ { role: "system", content: $('Skill copilote').item.json.skill }, { role: "user", content: $('Normaliser la demande').item.json.texte_origine } ], max_tokens: 1024, temperature: 0.7 }) }}
```

Options : Never Error ON + timeout 30000. On Error = Continue. (Quota épuisé → 429/3036 → branche false du IF.)

### 5. IF `Réponse IA reçue ?`

```
={{ !!($json.choices && $json.choices[0] && $json.choices[0].message && $json.choices[0].message.content) }}
```

is true → étape 6. false → étape 7bis (PATCH alerte indisponible **dédié**, pas celui de la branche true) → étape 8bis (Respond dédié).

### 6. Code `Parser + filtrer CP-10`

```javascript
const rep = $json;
const content = (((rep.choices || [])[0] || {}).message || {}).content || "";
const m = content.match(/\{[\s\S]*\}/);
let parsed = {};
try { parsed = m ? JSON.parse(m[0]) : {}; } catch (e) { parsed = {}; }
const ev = $('Normaliser la demande').item.json;
function norm(s) {
  return (s || "").toString().normalize("NFD").replace(/[̀-ͯ]/g, "")
    .toLowerCase().replace(/[^a-z0-9° ]/g, " ").replace(/\s+/g, " ").trim();
}
const INTERDITS = ["meilleur","le plus","la plus","les plus","n°1","no 1","imbattable","exceptionnel","moins cher","prix bas","petit prix","pas cher","bio","biologique","label","certifié","certifie","garanti","une seule séance","une seule seance","gueri","guerison"];
const SANTE = ["soulage","retrouver votre forme","retrouve ta forme","resultat"];
const liste = /sant/i.test(ev.categorie || "") ? INTERDITS.concat(SANTE) : INTERDITS;
function interditDans(texte) {
  const n = norm(texte);
  return liste.find(w => n.includes(norm(w))) || null;
}
function nettoyer(v) {
  if (!v || typeof v !== "string") return "";
  let t = v.trim();
  if (t.length > 300) t = t.slice(0, 300);
  return t;
}
let v1 = nettoyer(parsed.v1), v2 = nettoyer(parsed.v2);
const b1 = interditDans(v1), b2 = interditDans(v2);
if (b1) v1 = "";
if (b2) v2 = "";
const versions = [v1, v2].filter(Boolean);
return { json: {
  proposition_id: ev.proposition_id,
  version_ia_1: v1,
  version_ia_2: v2,
  nb_versions: versions.length,
  alerte_moderation: (parsed.alerte || "").toString().trim() || (versions.length === 0 ? "correction IA impossible (mots interdits CP-10)" : ""),
  bloque_v1: b1, bloque_v2: b2
} };
```

(La liste = `mots-interdits-cp10.txt` du repo.)

### 7. PATCH `PLACEHOLDER - Écrire les versions IA`

Method PATCH. URL (expression, règle 16) :

```
=https://api.madeleine-riviera.fr/items/proposition_fiche/{{ $('Parser + filtrer CP-10').item.json.proposition_id }}
```

Headers `Authorization` = `Bearer PLACEHOLDER`, `Content-Type` = `application/json`. Body (expression) :

```
={{ JSON.stringify({ version_ia_1: $('Parser + filtrer CP-10').item.json.version_ia_1, version_ia_2: $('Parser + filtrer CP-10').item.json.version_ia_2, alerte_moderation: $('Parser + filtrer CP-10').item.json.alerte_moderation }) }}
```

Never Error ON + timeout 10000 + On Error Continue → étape 8.

### 7bis. Branche false du IF — PATCH `PLACEHOLDER - Alerte IA indisponible`

⚠️ nœud **séparé** (le PATCH de l'étape 7 référence `Parser + filtrer CP-10`, qui n'a pas tourné ici — règle 16). Method PATCH, URL (expression) :

```
=https://api.madeleine-riviera.fr/items/proposition_fiche/{{ $('Normaliser la demande').item.json.proposition_id }}
```

Headers `Authorization` = `Bearer PLACEHOLDER`, `Content-Type` = `application/json`. Body (expression, valeurs en dur — aucune référence à un nœud qui n'a pas tourné) :

```
={{ JSON.stringify({ version_ia_1: "", version_ia_2: "", alerte_moderation: "correction IA indisponible (API indisponible)" }) }}
```

Never Error ON + timeout 10000 + On Error Continue → étape 8bis.

### 8. Respond `200 - versions IA`

Respond With JSON, Response Body (règle 14 + règle 16 — références amont) :

```
={{ JSON.stringify({ recu: true, nb_versions: $('Parser + filtrer CP-10').item.json.nb_versions, alerte: $('Parser + filtrer CP-10').item.json.alerte_moderation }) }}
```

Response Code 200. (Branche true uniquement.)

### 8bis. Branche false — Respond `200 - IA indisponible`

Respond With JSON, Response Body (valeurs en dur, règle 14) :

```
={{ JSON.stringify({ recu: true, nb_versions: 0, alerte: "correction IA indisponible (API indisponible)" }) }}
```

Response Code 200.

### 9. Error Trigger `Erreur détectée` → `Alerte au bureau - erreur workflow`

Copier depuis la n°7/n°8 (racine séparée, On Error Continue sur l'e-mail). Règle 18.

### 10. Save (Ctrl+S)

## Règles anti-erreurs

Voir `../REGLES-ERREURS-n8n.md` (20 règles). Ici : règles 1 (parse défensif), 3 (un seul `=`), 5/16 (`$json` après HTTP = réponse du dernier nœud → références `$('...').item.json` amont), 12 (apostrophes échappées dans les noms de nœuds — aucun ici), 14 (Respond en `JSON.stringify`), 10 (Respond branché avant test). HTTP externes : Never Error + On Error Continue + timeout.

## Tests de recette

Webhook → « Listen for test event » (1 appel par clic, recliquer entre chaque test), PowerShell une commande à la fois. Pour C1–C3, remplace **temporairement** le nœud HTTP `Appel Workers AI` par un Code du **même nom exact** (règle 13) :

**C1 — texte normal** (FAUX IA) :

```javascript
return { json: { choices: [{ message: { content: "{\"v1\":\"Depuis 1998, pains au levain et viennoiseries, ouverts dès 7h.\",\"v2\":\"Ouverte dès 7h, la boulangerie propose depuis 1998 pains au levain et viennoiseries.\",\"alerte\":\"\"}" } }] } };
```

```powershell
Invoke-RestMethod -Uri "https://n8n.acadenice.com/webhook-test/copilote" -Method POST -ContentType "application/json" -Body '{"proposition_id":"P1","texte_origine":"boulangerie depuis 1998 pain au levain ouvert 7h","categorie":"Boulangerie"}'
```

→ attendu : 200 `{"recu":true,"nb_versions":2,"alerte":""}` ; dans Executions, le PATCH a `version_ia_1` et `version_ia_2` renseignés, URL contient `/proposition_fiche/P1`.

**C2 — version avec mot interdit** (FAUX IA, v1 contient « meilleur ») → `nb_versions: 1` (v1 écartée) ; si les deux bloquées → `nb_versions: 0` + `alerte: "correction IA impossible (mots interdits CP-10)"`.

**C3 — activité illégale** (FAUX IA avec `alerte` non vide, ex. texte « je vends du tabac ») → PATCH avec `alerte_moderation` renseigné (message rouge modérateur).

**C4 — vraie IA sans clé** (remets le vrai HTTP `Appel Workers AI`, placeholders) → 401/403 → IF false → PATCH `alerte_moderation: "correction IA indisponible (API indisponible)"` + 200 `{"recu":true,"nb_versions":0,...}`.

Puis remets le vrai nœud HTTP, Save, aucun FAUX IA dans l'export final.

## Livraison

Quand C1–C4 sont OK : ⋯ → **Export** → envoie-moi le JSON (zéro `==`, `pinData` vide, pas de FAUX) → push dans `11-copilote-ia/workflow-copilote-ia.json` sur cette branche, merge vers `main` (PR #8).

⚠️ Placeholders à remplacer à la livraison des devs/Cloudflare : `PLACEHOLDER_ACCOUNT_ID`, `PLACEHOLDER_API_TOKEN` (compte Cloudflare de l'association, jeton au droit « Workers AI »), et l'URL Directus `https://api.madeleine-riviera.fr` + `Bearer PLACEHOLDER`.
