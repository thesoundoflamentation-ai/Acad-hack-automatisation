# N°3 — Validation par le bureau → lien de paiement (M-04)

> **Mise à jour 27/11** : le fichier `workflow-*.json` de ce dossier a été re-pushé avec l'API réelle livrée en clair (`https://site.g3.madeleine-riviera.fr/api/n8n/...`) et les e-mails via `POST /api/n8n/email`. Les secrets restent dans les credentials n8n (id `PLACEHOLDER` dans le repo). HelloAsso reste en sandbox tant que les vrais client_id/secret prod ne sont pas fournis.

Branche : `vibe/validation-paiement-m04-182b13` — à construire **à la main dans n8n**, export JSON envoyé pour push ici après validation.

## Objectif (cahier tech v0.5 §5.1 étape 3)

Quand le bureau valide une demande d'adhésion, n8n :
1. crée l'adhésion au statut **« En attente de paiement »** (écrit via l'API du site — **placeholder**),
2. génère un **jeton** et un lien stable `https://madeleine-riviera.fr/payer/{jeton}` (un lien HelloAsso direct expire en 15 min — d'où le lien intermédiaire, §5.1),
3. envoie l'e-mail **M-04** au commerçant avec ce lien de paiement.

Montant : **100 €/an** (hypothèse H5, à confirmer par le bureau vendredi 27).

## Déclenchement (placeholder)

Flow Directus côté site appellera le webhook quand le bureau clique « Valider ».

- Webhook : `POST /webhook/demande-validee`
- Body attendu :
```json
{ "demande": { "id": "D10", "email": "commercant@test.fr", "nom_commercial": "Boulangerie Meme" }, "montant_eur": 100 }
```

## Étapes à faire dans n8n

1. **Nouveau workflow** : `MR - Validation → lien de paiement (M-04)`, toggle **OFF**.
2. **Nœud 1 — Webhook** `Réception validation bureau` : Method `POST`, Path `demande-validee`, Respond = `Using 'Respond to Webhook' Node`.
3. **Nœud 2 — Code** `Préparer le paiement` (JavaScript, Run Once for Each Item) — colle ce bloc :

```javascript
let body = $json.body;
if (typeof body === "string") { try { body = JSON.parse(body); } catch (e) { body = null; } }
if (!body || typeof body !== "object" || Array.isArray(body)) body = $json;
const { randomUUID } = require('crypto');
const SITE_URL = "https://madeleine-riviera.fr"; // PLACEHOLDER : nom de domaine (H9), à remplacer par le vrai domaine
const demande = (body.demande && typeof body.demande === "object") ? body.demande : {};
const montant = Number(body.montant_eur) > 0 ? Number(body.montant_eur) : 100; // cotisation 100 €/an (H5)
const payer_token = randomUUID();
return {
  json: {
    demande: demande,
    montant_eur: montant,
    payer_token: payer_token,
    payer_url: SITE_URL + "/payer/" + payer_token
  }
};
```

4. **Nœud 3 — HTTP Request** `PLACEHOLDER - Enregistrer l'adhésion en attente de paiement` :
   - Method `PATCH`
   - URL (expression, un seul `=`) : `=https://api.madeleine-riviera.fr/items/demande_adhesion/{{ $json.demande.id }}` ← **PLACEHOLDER API des devs**
   - Headers (Send Headers ON → Specify Headers → **Using Fields Below**, 2 lignes, champs en mode **fixe** fx gris, **sans `=`** devant les valeurs) :
     - Name `Authorization` → Value `Bearer PLACEHOLDER`
     - Name `Content-Type` → Value `application/json`
   - Send Body ON, Body Content Type `JSON`, Specify Body `JSON`, JSON Body (expression, un seul `=`) :
     ```
     ={ "statut_adhesion": "en_attente_paiement", "payer_token": $json.payer_token, "payer_url": $json.payer_url, "montant_eur": $json.montant_eur }
     ```
   - Options : Timeout `10000`, Never Error ON
   - Settings → **On Error = Continue (regular output)**
   - Note sur le nœud (champ Notes) : `PLACEHOLDER — remplacer l'URL et le token quand l'API Directus est livrée`
5. **Nœud 4 — Send Email** `M-04 Lien de paiement` :
   - Credentials : même SMTP que les autres (« SMTP account 4 »)
   - From Email (expression) : `=site@madeleine-riviera.fr`
   - To Email (expression) : `={{ $json.demande.email }}`
   - Subject (expression) : `=Votre adhésion est validée — lien de paiement de votre cotisation`
   - Email Format : `Text`
   - Text (expression) :
```
=Bonjour {{ $json.demande.nom_commercial || "cher commerçant" }},

Bonne nouvelle : le bureau a validé votre demande d'adhésion à Madeleine Riviera (n° {{ $json.demande.id }}).

Pour activer votre adhésion ({{ $json.montant_eur }} € pour 12 mois), réglez votre cotisation sur ce lien :
{{ $json.payer_url }}

Ce lien reste valable tant que le paiement n'est pas effectué. Dès la confirmation du paiement, votre compte sera activé et vous recevrez un e-mail pour choisir votre mot de passe.

L'association Madeleine Riviera
```
   - Settings → On Error = `Continue (regular output)`
6. **Nœud 5 — Respond to Webhook** `200 reçu` :
   - Respond With `JSON`
   - Response Body (expression, un seul `=`) — ⚠️ piège n°5 : après le PATCH en continue-on-fail, `$json` = l'erreur, donc on référence le nœud 2 explicitement :
     ```
     ={ "recu": true, "payer_url": "{{ $('Préparer le paiement').item.json.payer_url }}" }
     ```
   - ⚠️ **Erreur « Referenced node doesn't exist »** = le nom entre `$('…')` ne correspond pas exactement au nœud 2. Manip sûre : dans le champ expression, efface et tape `$('` → une **liste déroulante** apparaît → clique sur le nœud dans la liste (le bon nom est inséré automatiquement). Vérifie aussi que le nœud 2 s'appelle bien exactement `Préparer le paiement` (double-clic sur son titre).
   - Options → Response Code `200`
7. **Save**.

## Règles anti-erreurs

Voir `../REGLES-ERREURS-n8n.md`. Ici en particulier : parse défensif (bloc ci-dessus), un seul `=` par expression, pas de `.first()` dans le nœud Code, On Error Continue sur HTTP/SMTP, et `$('Préparer le paiement').item.json…` (pas `$json…`) dans le Respond à cause du PATCH en continue.

## Tests de recette

Webhook → **Listen for test event** → PowerShell (`Invoke-RestMethod`, une commande à la fois) :

```powershell
# V1 — cas nominal : lien de paiement généré
Invoke-RestMethod -Uri "https://n8n.acadenice.com/webhook-test/demande-validee" -Method POST -ContentType "application/json" -Body '{"demande":{"id":"D10","email":"commercant@test.fr","nom_commercial":"Boulangerie Meme"},"montant_eur":100}'

# V2 — montant absent -> forcé à 100
Invoke-RestMethod -Uri "https://n8n.acadenice.com/webhook-test/demande-validee" -Method POST -ContentType "application/json" -Body '{"demande":{"id":"D11","email":"x@test.fr"}}'
```

Vérification dans **Executions** :
- réponse `200 {"recu":true,"payer_url":"https://madeleine-riviera.fr/payer/<uuid>"}`
- nœud `Préparer le paiement` → Output : `payer_token` = UUID, `montant_eur` = 100, `payer_url` contient `/payer/`
- nœud `M-04 Lien de paiement` → Output : `subject`/`text` contiennent le lien `payer_url` et « 100 € » (l'envoi SMTP échoue ENOTFOUND, normal, On Error Continue)
- nœud PATCH → erreur DNS/placeholder attendue, flux continue

## Livraison

Quand V1/V2 sont OK : ⋯ → **Export** → copie le JSON → envoie-le moi → je le pousse dans `03-validation-paiement/workflow-validation-paiement.json` sur cette branche, puis merge vers `main`.
