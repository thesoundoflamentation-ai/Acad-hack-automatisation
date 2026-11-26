# N°2 — Notifications e-mail de base (M-01 / M-03)

Branche : `vibe/notifications-base-182b13` — à construire **à la main dans n8n**, puis export JSON envoyé pour push ici après validation.

## Objectif (cahier tech v0.5 §7.2)

| # | E-mail | Déclencheur | Destinataire |
|---|---|---|---|
| M-01 | « Votre demande d'adhésion est bien reçue » | création de la demande | le commerçant (son e-mail) |
| M-03 | « Votre demande n'a pas été retenue » + **motif** | refus par le bureau | le commerçant |

(M-02 « alerte au bureau » est déjà dans la n°1. M-04 lien de paiement = n°3.)

## Déclenchement (placeholder)

Un **Flow Directus** côté site appellera le webhook n8n quand une demande est créée ou refusée. Tant que l'API du site n'existe pas : on teste à la main.

- Webhook : `POST /webhook/evenement-adhesion`
- Body attendu :
```json
{
  "evenement": "demande_creee" | "demande_refusee",
  "demande": { "id": "D1", "email": "commercant@test.fr", "nom_commercial": "Boulangerie Meme" },
  "motif": "SIRET du siège : demander le SIRET du commerce du boulevard"
}
```
(`motif` présent seulement si `demande_refusee`.)

## Étapes à faire dans n8n

1. **Nouveau workflow** : nom `MR - Notifications adhésion (M-01/M-03)`, toggle **OFF**.
2. **Nœud 1 — Webhook** `Réception événement adhésion` : Method `POST`, Path `evenement-adhesion`, Respond = `Using 'Respond to Webhook' Node`.
3. **Nœud 2 — Code** `Normaliser l'événement` (Language JavaScript, Mode Run Once for Each Item) — colle ce bloc :

```javascript
let body = $json.body;
if (typeof body === "string") { try { body = JSON.parse(body); } catch (e) { body = null; } }
if (!body || typeof body !== "object" || Array.isArray(body)) body = $json;
return {
  json: {
    evenement: body.evenement || "",
    demande: (body.demande && typeof body.demande === "object") ? body.demande : {},
    motif: body.motif || ""
  }
};
```

4. **Nœud 3 — Switch** `Quel événement ?` : Mode `Rules`, 2 règles :
   - règle 1 : **Value 1** en mode **expression** (fx BLEU activé) → tape `{{ $json.evenement }}` **SANS `=`** ; Operation `equals` (string) ; **Value 2** en mode **fixe** (gris) → tape `demande_creee` (texte, sans `=`) → cette règle sort sur **output 0**
   - règle 2 : idem avec `demande_refusee` → **output 1**
   - **Fallback output : crée une 3e sortie (output 2)** et branche-la **directement sur le nœud Respond to Webhook** (étape 7) — comme ça même un événement inconnu reçoit un 200 et l'appelant (Flow Directus) ne bloque pas. Sans fallback, un événement inconnu s'arrête au Switch et la requête HTTP reste en attente (c'est le « blocage » constaté sur N3 — comportement normal).
   - ⚠️ **Piège constaté** : si tu vois `=={{ ... }}` dans l'export, la règle ne matche jamais → aucune branche → le Respond n'est jamais atteint → la requête HTTP « bloque » (timeout). Correction : un seul `=` max. Vérif : ⋯ → Export → chercher `"=="` → aucun.
5. **Branche output 0 → Nœud 4 — Send Email** `M-01 Confirmation au commerçant` :
   - Credentials : le même SMTP que la n°1 (« SMTP account 4 »)
   - From Email (expression, un seul `=`) : `=site@madeleine-riviera.fr`
   - To Email (expression) : `={{ $json.demande.email }}`
   - Subject (expression) : `=Votre demande d'adhésion Madeleine Riviera est bien reçue`
   - Email Format : `Text`
   - Text (expression) :
```
=Bonjour {{ $json.demande.nom_commercial || "cher commerçant" }},

Votre demande d'adhésion à l'association Madeleine Riviera a bien été enregistrée (demande n° {{ $json.demande.id }}).
Elle sera examinée par le bureau : vous recevrez un e-mail avec la décision (validation avec lien de paiement, ou refus avec le motif).

À bientôt,
L'association Madeleine Riviera
```
   - Onglet Settings → **On Error** = `Continue (regular output)`
6. **Branche output 1 → Nœud 5 — Send Email** `M-03 Refus avec motif` :
   - Mêmes réglages, avec :
   - To Email : `={{ $json.demande.email }}`
   - Subject : `=Votre demande d'adhésion n'a pas été retenue`
   - Text (expression) :
```
=Bonjour {{ $json.demande.nom_commercial || "cher commerçant" }},

Votre demande d'adhésion (n° {{ $json.demande.id }}) n'a pas été retenue par le bureau.

Motif : {{ $json.motif }}

Vous pouvez corriger votre demande et la renvoyer depuis le site.

L'association Madeleine Riviera
```
   - Settings → On Error = `Continue (regular output)`
7. **Brancher les deux sorties (nœuds 4 et 5) → Nœud 6 — Respond to Webhook** `200 reçu` :
   - Respond With : `JSON`
   - Response Body (expression, un seul `=`) : `={ "recu": true, "evenement": "{{ $json.evenement }}" }`
   - Options → Response Code : `200`
8. **Save**.

## Note sur le SMTP en placeholder

Tant que le credential SMTP n'est pas un vrai compte Scaleway, l'envoi échoue avec une erreur du type `getaddrinfo ENOTFOUND smtp.scaleway.com` — **c'est attendu**. Ce qui compte :
1. Les 2 nœuds Send Email doivent avoir **Settings → On Error = Continue (regular output)** → sinon l'exécution s'arrête à l'e-mail et aucun 200 n'est renvoyé.
2. La recette porte sur le **contenu généré** (subject/text) visible dans l'Output du nœud e-mail, pas sur l'envoi réel.

## Règles anti-erreurs à respecter (rétro n°1)

Lire `../REGLES-ERREURS-n8n.md` avant. Points clés ici : parse défensif du body (étape 3), un seul `=` par expression (vérifier via ⋯ → Export, chercher `"=="`), pas de `.first()` dans le nœud Code, On Error = Continue sur les 2 e-mails (le SMTP est bidon pour l'instant).

## Tests de recette

Webhook → **Listen for test event** → PowerShell (`Invoke-RestMethod`, une commande à la fois) :

```powershell
# N1 — M-01 attendu
Invoke-RestMethod -Uri "https://n8n.acadenice.com/webhook-test/evenement-adhesion" -Method POST -ContentType "application/json" -Body '{"evenement":"demande_creee","demande":{"id":"D1","email":"commercant@test.fr","nom_commercial":"Boulangerie Meme"}}'

# N2 — M-03 attendu, avec motif
Invoke-RestMethod -Uri "https://n8n.acadenice.com/webhook-test/evenement-adhesion" -Method POST -ContentType "application/json" -Body '{"evenement":"demande_refusee","demande":{"id":"D2","email":"commercant@test.fr","nom_commercial":"Boulangerie Meme"},"motif":"SIRET du siège : demander le SIRET du commerce du boulevard"}'

# N3 — événement inconnu attendu : aucune branche, juste le 200
Invoke-RestMethod -Uri "https://n8n.acadenice.com/webhook-test/evenement-adhesion" -Method POST -ContentType "application/json" -Body '{"evenement":"bidule","demande":{"id":"D3","email":"x@test.fr"}}'
```

Vérification dans **Executions** :
- N1 : exécution verte → nœud `M-01 Confirmation au commerçant` → Output → le champ `text`/`subject` doit contenir « bien reçue » et « Boulangerie Meme » et « D1 ». (L'envoi SMTP lui-même échouera avec le credential bidon → nœud en erreur mais flux continue → réponse 200.)
- N2 : nœud `M-03 Refus avec motif` → Output → le texte contient bien le motif « SIRET du siège… ».
- N3 : événement inconnu → **aucun nœud e-mail exécuté**, mais le Respond répond quand même `200 {"recu":true,"evenement":"bidule"}` (grâce au fallback output 2). Dans Executions : l'exécution N3 ne montre que Webhook → Code → Switch → Respond.

## Livraison

Quand N1/N2/N3 sont OK : ⋯ → **Export** → copie le JSON du workflow → envoie-le moi ici → après validation je le pousse dans ce dossier (`workflow-notifications-base.json`) et on merge la branche vers `main`.
