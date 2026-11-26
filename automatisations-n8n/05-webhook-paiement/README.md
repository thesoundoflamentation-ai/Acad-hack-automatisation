# N°5 — Webhook paiement HelloAsso → activation de l'adhésion

Branche : `vibe/webhook-paiement-helloasso-182b13` — à construire **à la main dans n8n**, export JSON envoyé pour push ici après validation.

## Objectif (cahier tech v0.5 §5.1 étape 5, tests T-07 / T-08, e-mail M-05)

HelloAsso envoie une notification sur notre webhook à chaque événement. Quand un **paiement est confirmé**, le workflow :

1. extrait l'événement (`eventType` + `data`) — on ne traite que `eventType === "Payment"` ;
2. **vérifie l'idempotence** : l'événement a-t-il déjà été enregistré côté site ? (GET placeholder sur `evenement_paiement` filtré par `helloasso_event_id` — T-07 : un webhook rejoué ne doit rien refaire) ;
3. **relit le paiement côté API HelloAsso** (`GET /v5/payments/{id}` — T-08 : on ne se fie jamais au seul webhook) et ne traite que si `state === "Authorized"` ;
4. lit la commande (`GET /v5/orders/{orderId}`) pour récupérer l'e-mail du payeur ;
5. **enregistre l'événement** côté site (POST placeholder `items/evenement_paiement`, clé unique `helloasso_event_id` côté Directus — le Flow Directus déclenché à la création active alors le règlement, l'adhésion 12 mois à partir du paiement, et la création du compte commerçant) ;
6. envoie l'e-mail **M-05** (bienvenue, lien d'activation du compte) ;
7. répond **toujours 200** à HelloAsso (sinon HelloAsso réessaie) : `{"traite":true}` / `{"deja_traite":true}` / `{"ignore":true}`.

⚠️ Règle métier : le paiement n'est confirmé que par **webhook + relecture API**, jamais par la seule page de retour du site.

## Format de la notification HelloAsso (API v5)

```json
{
  "eventType": "Payment",
  "data": { "id": 987654, "orderId": 123456, "amount": 10000, "date": "2026-11-26T10:00:00+01:00", "state": "Authorized" },
  "metadata": { "adhesion_id": "D10", "payer_token": "<uuid>" }
}
```

- `eventType` : `Payment` (paiement autorisé), `Order` (commande créée), `Form`, etc. → on ne garde que `Payment`.
- `metadata` : renvoie ce qu'on avait mis dans le checkout (n°4) → `adhesion_id` / `payer_token`.
- En **prod**, l'URL de notification à renseigner dans le back-office HelloAsso (section Notifications) : `https://n8n.acadenice.com/webhook/helloasso-webhook` (chemin `/webhook/...`, pas `/webhook-test/...`).
- Sandbox API : `https://api.helloasso-sandbox.com/v5` · prod : `https://api.helloasso.com/v5`. OAuth2 idem n°4 (`POST {base}/oauth2/token`, form `grant_type=client_credentials`).

## Schéma des nœuds

```
Webhook HelloAsso → Extraire l'événement → IF "Paiement à traiter ?"
   ├─ false → Respond 200 ignoré
   └─ true → PLACEHOLDER - Événement déjà traité ? (GET) → IF "Déjà traité ?"
        ├─ true → Respond 200 déjà traité
        └─ false → Token OAuth2 HelloAsso → Relire le paiement HelloAsso (GET /v5/payments/{id})
             → IF "Paiement autorisé ?" (state === "Authorized")
                  ├─ false → Respond 200 ignoré
                  └─ true → Lire la commande HelloAsso (GET /v5/orders/{orderId})
                       → Assembler le contexte (Code) → PLACEHOLDER - Enregistrer l'événement (POST)
                       → M-05 Bienvenue (e-mail) → Respond 200 traité
```

## Étapes à faire dans n8n

Nouveau workflow **`MR - Webhook paiement HelloAsso`**, OFF :

1. **Webhook** `Webhook HelloAsso` : Method `POST`, Path `helloasso-webhook`, Respond = `Using 'Respond to Webhook' Node`.
   - ⚠️ Tant qu'**aucun** nœud `Respond to Webhook` n'est connecté, tout test échoue avec `No Respond to Webhook node found in the workflow` → **c'est normal en cours de construction**. Construis d'abord tous les nœuds y compris les 3 `Respond to Webhook` (étapes 13–15), branche-les, Save — l'erreur disparaît. Alternative temporaire : mets le webhook en Respond = `Immediately`, teste, puis remets `Using 'Respond to Webhook' Node` **avant la recette** (plusieurs nœuds Respond to Webhook sont possibles, un par branche).
2. **Code** `Extraire l'événement` (JavaScript, Run Once for Each Item) :
```javascript
let body = $json.body;
if (typeof body === "string") { try { body = JSON.parse(body); } catch (e) { body = null; } }
if (!body || typeof body !== "object" || Array.isArray(body)) body = $json;
const data = (body.data && typeof body.data === "object" && !Array.isArray(body.data)) ? body.data : {};
const meta = (body.metadata && typeof body.metadata === "object" && !Array.isArray(body.metadata)) ? body.metadata : {};
return { json: {
  eventType: String(body.eventType || ""),
  payment_id: data.id != null ? String(data.id) : "",
  order_id: data.orderId != null ? String(data.orderId) : "",
  montant_centimes: (data.amount != null) ? data.amount : null,
  date_paiement: data.date || "",
  adhesion_id: meta.adhesion_id || "",
  payer_token: meta.payer_token || ""
} };
```
3. **IF** `Paiement à traiter ?` : condition boolean, Value 1 (expression, un seul `=`) : `={{ $json.eventType === "Payment" && $json.payment_id !== "" }}`, opération `is true` → true = à traiter, false = ignoré.
4. **HTTP Request** `PLACEHOLDER - Événement déjà traité ?` : GET, URL (expression, un seul `=`) :
   ```
   =https://api.madeleine-riviera.fr/items/evenement_paiement?filter[helloasso_event_id][_eq]={{ $json.payment_id }}&fields=id,statut
   ```
   Headers (Using Fields Below, mode fixe, sans `=`) : `Authorization` → `Bearer PLACEHOLDER`. Options : timeout 10000 + **Never Error ON**. Settings → On Error = `Continue (regular output)`.
5. **IF** `Déjà traité ?` : `={{ $json.data && $json.data.length > 0 }}` ⚠️ bien `length` (pas `lenght`) `is true` → true = déjà traité, false = continuer.
   - ⚠️ Si le GET placeholder échoue (continue-on-fail), `$json` = `{error:...}` → `data` absent → branche **false** → on continue (comportement sûr : on relira le paiement chez HelloAsso).
6. **HTTP Request** `Token OAuth2 HelloAsso` : POST `https://api.helloasso-sandbox.com/oauth2/token` (prod : `api.helloasso.com`), Send Body ON → **Body Content Type = `Form URL Encoded`** → 3 champs : `grant_type` = `client_credentials`, `client_id` = `PLACEHOLDER`, `client_secret` = `PLACEHOLDER`. ⚠️ Ne laisse PAS le body en mode « String » avec du texte libre (`grant_type = ...`) : ce n'est pas du form-urlencoded valide, HelloAsso répondrait 400. Never Error + On Error Continue + timeout 10000.
7. **HTTP Request** `Relire le paiement HelloAsso` : GET, URL (expression) :
   ```
   =https://api.helloasso-sandbox.com/v5/payments/{{ $json.payment_id }}
   ```
   Header (Using Fields Below) : `Authorization` → Value (expression, un seul `=`) : `=Bearer {{ $('Token OAuth2 HelloAsso').item.json.access_token }}` ← référence via la liste déroulante `$('`. Never Error + On Error Continue + timeout 10000.
8. **IF** `Paiement autorisé ?` : `={{ $json.state === "Authorized" }}` is true → true = traiter, false = ignoré.
9. **HTTP Request** `Lire la commande HelloAsso` : GET, URL (expression) :
   ```
   =https://api.helloasso-sandbox.com/v5/orders/{{ $('Extraire l'événement').item.json.order_id }}
   ```
   Header `Authorization` = `=Bearer {{ $('Token OAuth2 HelloAsso').item.json.access_token }}`. Never Error + On Error Continue + timeout 10000. → fournit `payer.email`, `payer.firstName`, `payer.lastName`.
10. **Code** `Assembler le contexte` (après `Lire la commande`) — à partir de là, `$json` n'est plus l'événement : on reconstruit un objet plat en référençant les nœuds amont :
```javascript
const ev = $('Extraire l'événement').item.json;
const paiement = $('Relire le paiement HelloAsso').item.json || {};
const commande = $json || {};
const payer = (commande.payer && typeof commande.payer === "object") ? commande.payer : {};
return { json: {
  eventType: ev.eventType,
  payment_id: ev.payment_id,
  order_id: ev.order_id,
  montant_centimes: (ev.montant_centimes != null) ? ev.montant_centimes : ((paiement.amount != null) ? paiement.amount : null),
  date_paiement: ev.date_paiement || paiement.date || "",
  state_paiement: paiement.state || "",
  adhesion_id: ev.adhesion_id || "",
  payer_token: ev.payer_token || "",
  payer_email: payer.email || "",
  payer_prenom: payer.firstName || "",
  payer_nom: payer.lastName || ""
} };
```
11. **HTTP Request** `PLACEHOLDER - Enregistrer l'événement de paiement` : POST `https://api.madeleine-riviera.fr/items/evenement_paiement`, Headers (Using Fields Below) : `Authorization` → `Bearer PLACEHOLDER`, `Content-Type` → `application/json`. Send Body ON → Specify Body `JSON` → JSON Body (expression, un seul `=`, pas de guillemets autour de l'objet) :
    ```
    ={ "helloasso_event_id": $json.payment_id, "helloasso_order_id": $json.order_id, "montant_centimes": $json.montant_centimes, "date_paiement": $json.date_paiement, "adhesion_id": $json.adhesion_id || null, "payer_token": $json.payer_token || null, "statut": "traite", "source": "webhook_helloasso" }
    ```
    Never Error + On Error Continue + timeout 10000. Note sur le nœud : `PLACEHOLDER — clé unique helloasso_event_id côté Directus (T-07) ; le Flow Directus active règlement + adhésion 12 mois + compte`.
12. **Send Email** `M-05 Bienvenue - PLACEHOLDER SMTP` (même credential SMTP que les n°2–4, On Error = Continue) :
    - From Email : `=site@madeleine-riviera.fr`
    - To Email (expression) : `={{ $('Assembler le contexte').item.json.payer_email }}` ← après le POST (continue-on-fail) on référence le nœud amont, pas `$json`
    - Subject : `=Bienvenue chez Madeleine Riviera — votre adhésion est active`
    - Text (expression) :
    ```
    =Bonjour {{ $('Assembler le contexte').item.json.payer_prenom }},

Votre paiement de {{ ($('Assembler le contexte').item.json.montant_centimes || 0) / 100 }} € a bien été confirmé.

Votre adhésion à l'association Madeleine Riviera est active pour 12 mois.

Activez votre espace commerçant pour publier votre fiche :
https://madeleine-riviera.fr/activer-compte?jeton=PLACEHOLDER

L'association Madeleine Riviera
    ```
13. **Respond to Webhook** `200 - paiement traité` (fin de branche true) : Respond With `JSON`, Response Body (expression) : `={ "traite": true, "helloasso_event_id": $('Assembler le contexte').item.json.payment_id }`, Response Code `200`.
14. **Respond to Webhook** `200 - déjà traité` (branche true de `Déjà traité ?`) : `={ "deja_traite": true, "helloasso_event_id": $('Extraire l'événement').item.json.payment_id }`, code `200`.
15. **Respond to Webhook** `200 - ignoré` (branche false de `Paiement à traiter ?` ET branche false de `Paiement autorisé ?`) : `={ "ignore": true, "eventType": $('Extraire l'événement').item.json.eventType }`, code `200`.
16. **Save**.

## Règles anti-erreurs

Voir `../REGLES-ERREURS-n8n.md`. Ici en particulier : parse défensif du body (règle 1), jamais `.first()` dans un Code (règle 2), un seul `=` par expression (règle 3), IF boolean → `is true` (règle 4), **après un nœud en On Error Continue référencer le nœud amont `$('...').item.json`** (règle 5 — c'est pour ça que le nœud `Assembler le contexte` existe), body JSON `={ ... }` (règle 6), `length` pas `lenght` (règle 9), références de nœuds via la liste déroulante `$('`. Tous les HTTP externes : Never Error + On Error Continue + timeout 10 s.

## Tests de recette

⚠️ **Mode test n8n** : `/webhook-test/...` n'accepte **qu'un seul appel** par clic sur « Listen for test event » → **reclique « Listen for test event » entre chaque test**. Le webhook répond 200 dès réception : le vrai résultat est dans **Executions** (vert/rouge) et dans la réponse du terminal.

Webhook → **Listen for test event** → PowerShell (**une commande à la fois**) :

```powershell
# P3 — eventType "Order" : attendu 200 {"ignore":true,"eventType":"Order"} (aucun GET)
Invoke-RestMethod -Uri "https://n8n.acadenice.com/webhook-test/helloasso-webhook" -Method POST -ContentType "application/json" -Body '{"eventType":"Order","data":{"id":123456,"amount":10000},"metadata":{"adhesion_id":"D10","payer_token":"abc"}}'

# P2 — Payment déjà traité : attendu 200 {"deja_traite":true,"helloasso_event_id":"987654"} et AUCUN appel à "Relire le paiement"
Invoke-RestMethod -Uri "https://n8n.acadenice.com/webhook-test/helloasso-webhook" -Method POST -ContentType "application/json" -Body '{"eventType":"Payment","data":{"id":987654,"orderId":123456,"amount":10000,"date":"2026-11-26T10:00:00+01:00"},"metadata":{"adhesion_id":"D10","payer_token":"abc"}}'

# P1 — Payment neuf autorisé : attendu 200 {"traite":true,"helloasso_event_id":"987654"}
Invoke-RestMethod -Uri "https://n8n.acadenice.com/webhook-test/helloasso-webhook" -Method POST -ContentType "application/json" -Body '{"eventType":"Payment","data":{"id":987654,"orderId":123456,"amount":10000,"date":"2026-11-26T10:00:00+01:00"},"metadata":{"adhesion_id":"D10","payer_token":"abc"}}'
```

**Pins pour P1 et P2** (tant que l'API du site est en placeholder et sans vraies clés HelloAsso, les GET externes échouent → branche « ignoré » ; on épingle des réponses fictives) :

- Nœud `PLACEHOLDER - Événement déjà traité ?` → onglet **Pin** :
  - P1 : `{"data":[]}`
  - P2 : `{"data":[{"id":"E1","statut":"traite"}]}`
- Nœud `Relire le paiement HelloAsso` → Pin (P1 uniquement) :
```json
{"id":987654,"orderId":123456,"amount":10000,"initialAmount":10000,"state":"Authorized","date":"2026-11-26T10:00:00+01:00"}
```
- Nœud `Lire la commande HelloAsso` → Pin (P1 uniquement) :
```json
{"id":123456,"payer":{"email":"commercant@test.fr","firstName":"Youssef","lastName":"Abdelli"}}
```

→ **Save** → reclique **Listen for test event** → lance le test → vérifie dans **Executions** :

- **P1** : nœud `PLACEHOLDER - Enregistrer l'événement de paiement` → onglet **Input** : `helloasso_event_id: "987654"`, `helloasso_order_id: "123456"`, `montant_centimes: 10000`, `statut: "traite"`, `adhesion_id: "D10"` (le POST lui-même échouera → erreur continue, normal en placeholder) ; nœud `M-05 Bienvenue` → To `commercant@test.fr` (envoi SMTP en échec `ENOTFOUND smtp.scaleway.com` → continue, normal) ; réponse terminal `{"traite":true,...}`.
- **P2** : réponse `{"deja_traite":true,...}` et dans Executions le nœud `Relire le paiement HelloAsso` **n'a pas été exécuté**.
- **P3** : réponse `{"ignore":true,"eventType":"Order"}` et seul le début du flux a tourné.

→ **Dé-épingle** (Pin → Clear) après les tests.

**Sans l'onglet Pin** : remplace temporairement chaque GET à simuler par un nœud **Code** `FAUX GET` qui `return { json: <réponse fictive> };` (ex. `return { json: { data: [] } };`), teste, puis remets les vrais nœuds. Ne laisse aucun nœud de test dans l'export final.

## Livraison

Quand P1/P2/P3 sont OK : ⋯ → **Export** → copie le JSON → envoie-le moi → push dans `05-webhook-paiement/workflow-webhook-paiement.json` sur cette branche, merge vers `main` (PR #4).
