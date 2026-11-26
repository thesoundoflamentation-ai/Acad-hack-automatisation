# N°6 — Modération des fiches : alerte bureau, M-06 (publiée), M-07 (refusée avec motif)

Branche : `vibe/moderation-fiches-182b13` — à construire **à la main dans n8n**, export JSON envoyé pour push ici après validation.

## Objectif (CdC tech v0.5 §5.3 et §7.2, CdC fonctionnel §5.3–5.4)

Tout ce que publie un commerçant passe par le bureau. n8n ne fait **que notifier par e-mail** (les changements de statut et la copie dans `commerce` sont faits par les Flows Directus) :

| Événement (reçu par webhook) | Action n8n |
|---|---|
| `proposition_creee` (le commerçant clique « Envoyer en modération » ; statut « Fiche à valider ») | **Alerte e-mail au bureau** (M-02) : nom du commerce, id de la proposition, texte d'origine, version retenue, et l'alerte rouge éventuelle. Le commerçant n'est **pas** averti de l'alerte. |
| `fiche_validee` (le bureau a validé ; la fiche est publiée) | **M-06** au commerçant : « votre fiche est publiée » |
| `fiche_refusee` (le bureau a refusé, avec motif ; la fiche revient en brouillon) | **M-07** au commerçant : refus + motif |

## Contrat du webhook (placeholder — à ajuster quand les devs livrent l'API)

`POST /webhook/evenement-fiche` (prod) — body :

```json
{
  "evenement": "proposition_creee | fiche_validee | fiche_refusee",
  "proposition": {
    "id": "P1",
    "commerce_id": "C1",
    "nom_commercial": "Boulangerie Meme",
    "email": "commercant@test.fr",
    "texte_origine": "texte saisi par le commerçant",
    "version_retenue": "version IA choisie",
    "alerte_moderation": ""
  },
  "motif": ""
}
```

Réponse attendue : toujours **200** `{"recu":true,"evenement":"..."}` (même pour un événement inconnu).

## Schéma des nœuds

```
Webhook → Normaliser l'événement → Switch "Quel événement ?"
   ├─ 0 proposition_creee → Alerte bureau (M-02) ─┐
   ├─ 1 fiche_validee     → M-06 Fiche publiée  ────┼─→ Respond 200 - reçu
   ├─ 2 fiche_refusee     → M-07 Fiche refusée  ────┤
   └─ fallback (extra) ─────────────────────────────┘
```

## Étapes à faire dans n8n

Nouveau workflow **`MR - Modération fiches`**, OFF :

1. **Webhook** `Webhook événement fiche` : Method `POST`, Path `evenement-fiche`, Respond = `Using 'Respond to Webhook' Node`. ⚠️ Règle 10 : tant qu'aucun nœud Respond n'est branché, un test échoue avec « No Respond to Webhook node found » — construis d'abord tout le flux, branche le Respond, Save.
2. **Code** `Normaliser l'événement` (JavaScript, Run Once for Each Item) :
```javascript
let body = $json.body;
if (typeof body === "string") { try { body = JSON.parse(body); } catch (e) { body = null; } }
if (!body || typeof body !== "object" || Array.isArray(body)) body = $json;
const p = (body.proposition && typeof body.proposition === "object" && !Array.isArray(body.proposition)) ? body.proposition : {};
return { json: {
  evenement: String(body.evenement || ""),
  proposition: {
    id: String(p.id || ""),
    commerce_id: String(p.commerce_id || p.commerce || ""),
    nom_commercial: String(p.nom_commercial || ""),
    email: String(p.email || ""),
    texte_origine: String(p.texte_origine || ""),
    version_retenue: String(p.version_retenue || ""),
    alerte_moderation: String(p.alerte_moderation || "")
  },
  motif: String(body.motif || "")
} };
```
3. **Switch** `Quel événement ?` (typeVersion 3.5) : 3 règles, `={{ $json.evenement }}` **equals** `proposition_creee` / `fiche_validee` / `fiche_refusee` (une règle par sortie, dans l'ordre), Fallback Output = `extra` (sortie 4). ⚠️ Règle 3 : un seul `=`.
4. **Send Email** `Alerte bureau - nouvelle proposition (M-02)` (sortie 0) — credential SMTP existant, Settings → On Error = `Continue (regular output)` :
   - From Email : `=site@madeleine-riviera.fr`
   - To Email : `=bureau@madeleine-riviera.fr`
   - Subject (expression) : `=[PROPOSITION] Nouvelle fiche à valider — {{ $json.proposition.nom_commercial }} (n° {{ $json.proposition.id }})`
   - Text (expression, emailFormat `text`) :
   ```
   =Nouvelle proposition de fiche en attente de modération.

Commerce : {{ $json.proposition.nom_commercial }} (proposition n° {{ $json.proposition.id }}, commerce {{ $json.proposition.commerce_id }})

Texte d'origine :
{{ $json.proposition.texte_origine }}

Version retenue (IA) :
{{ $json.proposition.version_retenue }}
{{ $json.proposition.alerte_moderation ? "ALERTE ROUGE : " + $json.proposition.alerte_moderation : "" }}

À traiter dans le back-office (liste des propositions à valider).

L'alerte rouge éventuelle ne bloque rien : la décision reste humaine, et le commerçant n'en est pas averti.
   ```
5. **Send Email** `M-06 Fiche publiée` (sortie 1) — On Error Continue :
   - From `=site@madeleine-riviera.fr`, To `={{ $json.proposition.email }}`
   - Subject : `=Votre fiche est publiée sur madeleine-riviera.fr`
   - Text :
   ```
   =Bonjour {{ $json.proposition.nom_commercial || "cher commerçant" }},

Votre fiche a été validée par le bureau et est maintenant publiée sur madeleine-riviera.fr (proposition n° {{ $json.proposition.id }}).

Merci pour votre adhésion,
L'association Madeleine Riviera
   ```
6. **Send Email** `M-07 Fiche refusée` (sortie 2) — On Error Continue :
   - From `=site@madeleine-riviera.fr`, To `={{ $json.proposition.email }}`
   - Subject : `=Votre proposition de fiche n'a pas été retenue`
   - Text :
   ```
   =Bonjour {{ $json.proposition.nom_commercial || "cher commerçant" }},

Votre proposition de fiche (n° {{ $json.proposition.id }}) n'a pas été retenue par le bureau. Votre fiche revient en brouillon.

Motif : {{ $json.motif }}

Vous pouvez corriger et renvoyer une nouvelle proposition depuis votre espace.

L'association Madeleine Riviera
   ```
7. **Respond to Webhook** `200 - reçu` (unique, toutes les branches y convergent : les 3 e-mails + la sortie fallback) : Respond With `JSON`, Response Body (expression) — ⚠️ **règle 14 : chaîne JSON via `JSON.stringify`**, et règle 5 (après un e-mail en continue-on-fail, référencer le nœud amont) + règle 12 (apostrophe échappée dans le nom du nœud) :
   ```
   ={{ JSON.stringify({ recu: true, evenement: $('Normaliser l\'événement').item.json.evenement }) }}
   ```
   Response Code `200`.
8. Branche les fils : Webhook → Normaliser → Switch ; Switch sorties 0/1/2 → les 3 e-mails ; chaque e-mail → `200 - reçu` ; Switch fallback (extra) → `200 - reçu`. **Save**.

## Règles anti-erreurs

Voir `../REGLES-ERREURS-n8n.md`. Ici : parse défensif (règle 1), un seul `=` (règle 3), après un nœud On Error Continue référencer le nœud amont `$('...').item.json` (règle 5), apostrophe échappée `\'` dans les noms de nœuds (règle 12 — `Normaliser l'événement`), Respond to Webhook en `JSON.stringify` (règle 14), références de nœuds via la liste déroulante `$('`. E-mails en placeholder : On Error Continue (l'envoi SMTP échouera `ENOTFOUND smtp.scaleway.com` tant que le vrai TEM n'est pas branché → continue, normal).

## Tests de recette

⚠️ Mode test : `/webhook-test/...` = **1 seul appel par clic** sur « Listen for test event » → **recliquer entre chaque test**. Vrai statut dans **Executions**. PowerShell, **une commande à la fois**.

Webhook → « Listen for test event » → :

```powershell
# F1 — nouvelle proposition : attendu 200 {"recu":true,"evenement":"proposition_creee"} + alerte bureau dans Executions (To bureau@madeleine-riviera.fr, texte_origine + version_retenue présents)
Invoke-RestMethod -Uri "https://n8n.acadenice.com/webhook-test/evenement-fiche" -Method POST -ContentType "application/json" -Body '{"evenement":"proposition_creee","proposition":{"id":"P1","commerce_id":"C1","nom_commercial":"Boulangerie Meme","email":"commercant@test.fr","texte_origine":"Boulangerie depuis 1998, pain au levain, ouvert 7h.","version_retenue":"Depuis 1998, la Boulangerie Meme propose pains au levain et viennoiseries, ouverte dès 7h.","alerte_moderation":""}}'

# F2 — fiche validée : attendu 200 + M-06 vers commercant@test.fr
Invoke-RestMethod -Uri "https://n8n.acadenice.com/webhook-test/evenement-fiche" -Method POST -ContentType "application/json" -Body '{"evenement":"fiche_validee","proposition":{"id":"P1","commerce_id":"C1","nom_commercial":"Boulangerie Meme","email":"commercant@test.fr"}}'

# F3 — fiche refusée avec motif : attendu 200 + M-07 vers commercant@test.fr, motif dans le texte
Invoke-RestMethod -Uri "https://n8n.acadenice.com/webhook-test/evenement-fiche" -Method POST -ContentType "application/json" -Body '{"evenement":"fiche_refusee","proposition":{"id":"P1","commerce_id":"C1","nom_commercial":"Boulangerie Meme","email":"commercant@test.fr"},"motif":"Texte trop long (300 caractères max)."}'

# F4 — événement inconnu : attendu 200 {"recu":true,"evenement":"bidule"}, aucun e-mail
Invoke-RestMethod -Uri "https://n8n.acadenice.com/webhook-test/evenement-fiche" -Method POST -ContentType "application/json" -Body '{"evenement":"bidule","proposition":{"id":"P9","email":"x@test.fr"}}'
```

Vérification : dans **Executions**, pour F1/F2/F3 ouvrir le nœud e-mail correspondant → onglet Input : destinataire, sujet, texte (motif présent pour F3). L'envoi SMTP échoue (placeholder) → erreur continue → le Respond 200 répond quand même.

## Livraison

Quand F1–F4 sont OK : ⋯ → **Export** → copie le JSON → envoie-le moi → push dans `06-moderation-fiches/workflow-moderation-fiches.json` sur cette branche, merge vers `main` (PR #5). Vérifie avant export : zéro `==` (⋯ → Export → chercher `"=="`), aucun nœud de test, `pinData` vide.
