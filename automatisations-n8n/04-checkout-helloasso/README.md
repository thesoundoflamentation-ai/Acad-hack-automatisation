# N°4 — Checkout HelloAsso au clic sur `/payer/{jeton}`

Branche : `vibe/checkout-helloasso-182b13` — à construire **à la main dans n8n**, export JSON envoyé pour push ici après validation.

## Objectif (cahier tech v0.5 §5.1 étape 4, §7.1)

Quand le commerçant clique sur `/payer/{jeton}` (page Astro), le site appelle le webhook n8n, qui :
1. retrouve la demande par `payer_token` via l'API du site (**placeholder**),
2. crée un **checkout HelloAsso neuf** (API v5, montant en centimes, `backRef` = id de la demande, métadonnées) — car un lien HelloAsso direct expire en 15 min,
3. répond au site l'`redirectUrl` HelloAsso → Astro redirige le commerçant vers la page de paiement.

## API HelloAsso v5 (rappels)

- Sandbox (recette) : `https://api.helloasso-sandbox.com/v5` · Prod : `https://api.helloasso.com/v5`
- OAuth2 : `POST {base}/oauth2/token`, body `application/x-www-form-urlencoded` : `grant_type=client_credentials`, `client_id`, `client_secret` → réponse `{ "access_token": "..." }`
- Créer le checkout : `POST {base}/organizations/{organizationSlug}/checkout-intents`, header `Authorization: Bearer {access_token}`
- Body du checkout :
```json
{
  "totalAmount": 10000,
  "initialAmount": 10000,
  "itemName": "Cotisation Madeleine Riviera — 12 mois",
  "backRef": "<id demande/adhesion>",
  "containsDonation": false,
  "payer": { "email": "...", "firstName": "...", "lastName": "..." },
  "metadata": { "adhesion_id": "<id>", "payer_token": "<jeton>" }
}
```
- Réponse 201 : contient `redirectUrl` (et `id` du checkout)
- `organizationSlug` : **placeholder** `madeleine-riviera` (à lire dans le back-office HelloAsso de l'association)
- Credentials : **placeholders** (client_id / client_secret / slug), le bac à sable HelloAsso fonctionne sans RNA

## Étapes à faire dans n8n

Nouveau workflow **`MR - Checkout HelloAsso (/payer/{jeton})`**, OFF :

1. **Webhook** `Réception /payer/{jeton}` : Method `POST`, Path `payer`, Respond = `Using 'Respond to Webhook' Node`. Body attendu : `{ "payer_token": "<uuid>" }` (envoyé par la page Astro `/payer/{jeton}`).
2. **Code** `Extraire le jeton` (JavaScript, Run Once for Each Item) :
```javascript
let body = $json.body;
if (typeof body === "string") { try { body = JSON.parse(body); } catch (e) { body = null; } }
if (!body || typeof body !== "object" || Array.isArray(body)) body = $json;
return { json: { payer_token: String(body.payer_token || body.token || "") } };
```
3. **HTTP Request** `PLACEHOLDER - Lire la demande par jeton` : GET, URL (expression, un seul `=`) :
   ```
   =https://api.madeleine-riviera.fr/items/demande_adhesion?filter[payer_token][_eq]={{ $json.payer_token }}&fields=id,email,nom_commercial,montant_eur,statut_adhesion
   ```
   Headers (Using Fields Below, mode fixe, sans `=`) : `Authorization` → `Bearer PLACEHOLDER`. Options : timeout 10000, Never Error ON. Settings → On Error = `Continue (regular output)`. Note sur le nœud : `PLACEHOLDER — remplacer par l'API Directus réelle`.
4. **IF** `Demande trouvée ?` : condition `={{ $json.data && $json.data.length > 0 }}` is true → true = oui, false = non.
   - ⚠️ Si le GET placeholder échoue (continue-on-fail), `$json` = `{error:...}` → `data` absent → branche **false** → lien invalide (comportement attendu tant que l'API n'existe pas).
5. **Branche true → HTTP Request** `Token OAuth2 HelloAsso` : POST `https://api.helloasso-sandbox.com/oauth2/token` (sandbox ; prod = `api.helloasso.com`), Body Content Type `Form URL Encoded`, body : `grant_type` = `client_credentials`, `client_id` = `PLACEHOLDER`, `client_secret` = `PLACEHOLDER`. Never Error + On Error Continue. (En prod : remplacer par un vrai credential OAuth ou mettre le token en cache.)
6. **Branche true (suite) → HTTP Request** `Créer le checkout HelloAsso` : POST, URL (expression) :
   ```
   =https://api.helloasso-sandbox.com/v5/organizations/madeleine-riviera/checkout-intents
   ```
   Headers (Using Fields Below) : `Authorization` → Value (expression, un seul `=`) : `=Bearer {{ $('Token OAuth2 HelloAsso').item.json.access_token }}` ← ⚠️ piège n°5 + référence via liste déroulante `$('`. `Content-Type` → `application/json`.
   Send Body ON, JSON, Specify Body `JSON`, JSON Body (expression, un seul `=`) :
   ```
   ={ "totalAmount": ($('PLACEHOLDER - Lire la demande par jeton').item.json.data[0].montant_eur || 100) * 100, "initialAmount": ($('PLACEHOLDER - Lire la demande par jeton').item.json.data[0].montant_eur || 100) * 100, "itemName": "Cotisation Madeleine Riviera — 12 mois", "backRef": String($('PLACEHOLDER - Lire la demande par jeton').item.json.data[0].id), "containsDonation": false, "payer": { "email": $('PLACEHOLDER - Lire la demande par jeton').item.json.data[0].email || "" }, "metadata": { "adhesion_id": String($('PLACEHOLDER - Lire la demande par jeton').item.json.data[0].id), "payer_token": $json.payer_token } }
   ```
   Options : timeout 15000, Never Error ON. Settings → On Error = `Continue (regular output)`.
7. **Code** `Extraire redirectUrl` (après le checkout) :
```javascript
const cur = $json;
if (cur && cur.error) return { json: { erreur: true, message: String(cur.error && cur.error.message || cur.error) } };
return { json: { erreur: false, checkout_id: cur.id || null, redirectUrl: cur.redirectUrl || cur.checkoutIntentUrl || null } };
```
8. **Respond to Webhook** `200 redirectUrl` (branche true) : Respond With `JSON`, Response Body (expression) — références via `$('…')` :
   ```
   ={ "redirectUrl": "{{ $('Extraire redirectUrl').item.json.redirectUrl }}" }
   ```
   Response Code `200`.
9. **Branche false du IF → Respond to Webhook** `404 lien invalide` : Respond With `JSON`, Response Body `={ "error": "lien de paiement invalide ou expiré" }`, Options Response Code `404`.
10. **Save**.

## Règles anti-erreurs

Voir `../REGLES-ERREURS-n8n.md`. Ici : références de nœuds via la liste déroulante `$('` (sinon « Referenced node doesn't exist »), un seul `=` par expression, On Error Continue sur tous les HTTP, Never Error + timeout.

## Tests de recette

Webhook → **Listen for test event** → PowerShell (une commande à la fois) :

```powershell
# H1 — jeton bidon : attendu 404 {"error":"lien de paiement invalide ou expiré"} (le GET placeholder échoue -> branche false)
Invoke-RestMethod -Uri "https://n8n.acadenice.com/webhook-test/payer" -Method POST -ContentType "application/json" -Body '{"payer_token":"bidon"}'

# H2 — chemin complet : épingler (Pin) sur le nœud GET placeholder une réponse fictive :
#   {"data":[{"id":"D10","email":"commercant@test.fr","nom_commercial":"Boulangerie Meme","montant_eur":100,"statut_adhesion":"en_attente_paiement"}]}
#   puis lancer H2 avec n'importe quel token ; vérifier dans Executions que le nœud checkout a bien envoyé
#   totalAmount=10000, backRef="D10", metadata.payer_token (le POST HelloAsso échouera sans vraies clés -> erreur continue)
Invoke-RestMethod -Uri "https://n8n.acadenice.com/webhook-test/payer" -Method POST -ContentType "application/json" -Body '{"payer_token":"591fa12b-abdf-47f7-af1e-dad72feef69b"}'
```

Vérification H2 dans **Executions** → nœud `Créer le checkout HelloAsso` → onglet **Input** (le body envoyé) : `totalAmount: 10000`, `backRef: "D10"`, `metadata.adhesion_id: "D10"`.

## Livraison

Quand H1/H2 sont OK : ⋯ → **Export** → copie le JSON → envoie-le moi → push dans `04-checkout-helloasso/workflow-checkout-helloasso.json` sur cette branche, merge vers `main` (PR #3).
