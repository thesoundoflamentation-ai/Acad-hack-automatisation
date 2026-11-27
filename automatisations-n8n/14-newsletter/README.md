# N°14 — Newsletter (envoi manuel au bureau → liste des abonnés → journal)

Branche : `vibe/newsletter-182b13` — à construire **à la main dans n8n**, export JSON envoyé pour push ici après validation.

Processus (board Excalidraw « newsletter », 24/11) : le bureau rédige le mail et clique « Envoyer » → Flow Directus appelle le webhook n8n → n8n récupère la **liste des adhérents actifs ayant coché « newsletter »** (e-mails perso et pro, dédupliqués) → envoi **individuel** (jamais de BCC de masse) → **journal de l'envoi** (date, liste, statut « envoyé ») → réponse 200.

RGPD : consentement = case « newsletter » cochée (fonctionnel §6.3) ; chaque mail contient un pied de désinscription ; le journal reste côté back-office (bureau seul).

## Contrat du webhook (placeholder)

`POST /webhook/envoi-newsletter` — **Header Auth** (règle 21, credential placeholder `MR - secret webhook newsletter`) — body :

```json
{ "sujet": "Les nouveautés du boulevard — novembre", "texte": "Bonjour, ..." }
```

Réponses : 200 `{"envoyes":12,"erreurs":0,"journal_id":"..."}` ou 400 `{"erreur":"sujet et texte requis"}`.

## Schéma

```
Webhook envoi-newsletter → Normaliser la demande → IF "Demande valide ?" ─ false → 400
  true → GET Liste des abonnés (Directus, adhesion active + newsletter=true)
       → Déplier les destinataires (Code, dédoublonne e-mails perso/pro)
       → IF "Au moins un abonné ?" ─ false → 200 - aucun abonné
            true → Envoyer la newsletter (e-mail, 1 par item, On Error Continue)
                 → Compter les envois (Code) → POST Journal de l'envoi (Directus) → 200 - envoi terminé
Error Trigger (racine séparée) → Alerte au bureau - erreur workflow
```

## Étapes

Nouveau workflow **`MR - Newsletter`**, OFF.

### 1. Webhook `Envoi newsletter demandé`

POST, Path `envoi-newsletter`, Respond = `Using 'Respond to Webhook' Node`, **Authentication = Header Auth** (credential placeholder `MR - secret webhook newsletter`).

### 2. Code `Normaliser la demande`

```javascript
let body = $json.body;
if (typeof body === "string") { try { body = JSON.parse(body); } catch (e) { body = null; } }
if (!body || typeof body !== "object" || Array.isArray(body)) body = $json;
const sujet = String(body.sujet || "").trim().slice(0, 200);
const texte = String(body.texte || "").trim().slice(0, 20000);
return { json: { sujet, texte, valide: !!(sujet && texte) } };
```

### 3. IF `Demande valide ?`

`={{ $json.valide === true }}` is true → étape 4. false → Respond `400 - demande invalide` (`={{ JSON.stringify({ erreur: "sujet et texte requis" }) }}`, code 400, règle 14).

### 4. HTTP GET `PLACEHOLDER - Liste des abonnés`

URL fixe `https://api.madeleine-riviera.fr/items/adhesion`, header `Authorization` = `Bearer PLACEHOLDER`, **Send Query Parameters** (règle 19 — filtres en paramètres, pas dans l'URL) :

| Name | Value |
|---|---|
| `filter[statut][_eq]` | `active` |
| `filter[newsletter][_eq]` | `true` |
| `fields` | `id,email,email_pro,nom_commercial` |
| `limit` | `500` |

Options : Never Error ON + timeout 10000, On Error = Continue.

> ⚠️ Champs à aligner avec le modèle Directus réel : la case « newsletter » et l'e-mail professionnel s'appellent peut-être autrement (`newsletter`, `email_pro` sont des placeholders de nommage — à confirmer côté devs, comme `statut=active`).

### 5. Code `Déplier les destinataires` — mode **Run Once for All Items**

```javascript
const first = $input.all()[0].json;
if (!first || !Array.isArray(first.data)) throw new Error("Réponse inattendue de l'API (champ data absent) : " + JSON.stringify(first).slice(0, 300));
const vus = new Set();
const out = [];
for (const a of first.data) {
  for (const e of [a.email, a.email_pro]) {
    const mail = String(e || "").trim().toLowerCase();
    if (mail && mail.includes("@") && !vus.has(mail)) {
      vus.add(mail);
      out.push({ json: { email: mail, nom: String(a.nom_commercial || ""), adhesion_id: String(a.id || "") } });
    }
  }
}
return out;
```

### 6. IF `Au moins un abonné ?`

Après le Code, utiliser `$input.all().length` : `={{ $input.all().length > 0 }}` is true → étape 7. false → Respond `200 - aucun abonné` (`={{ JSON.stringify({ envoyes: 0, erreurs: 0, remarque: "aucun abonné newsletter" }) }}`, code 200).

### 7. E-mail `Envoyer la newsletter`

- From : `site@madeleine-riviera.fr` (fixe)
- To : `={{ $json.email }}` (un item = un destinataire)
- Subject : `={{ $('Normaliser la demande').item.json.sujet }}` (règle 16)
- Format : text. Texte :

```
={{ $('Normaliser la demande').item.json.texte + "\n\n---\nAssociation Madeleine Riviera — boulevard de la Madeleine, Nice.\nVous recevez ce message car vous avez coché « newsletter » lors de votre adhésion.\nPour ne plus recevoir la newsletter : répondez « STOP » à ce message ou connectez-vous à votre espace.\n" }}
```

Credential SMTP « SMTP account 4 » (placeholder), **On Error = Continue** (un échec n'arrête pas le lot).

### 8. Code `Compter les envois` — mode **Run Once for All Items**

```javascript
const tous = $input.all().map(i => i.json);
const envoyes = tous.filter(x => !(x && x.error)).length;
const erreurs = tous.length - envoyes;
return [{ json: {
  sujet: $('Normaliser la demande').item.json.sujet,
  envoyes, erreurs, total: tous.length,
  destinataires: tous.map(x => (x && x.email) || "").filter(Boolean)
} }];
```

### 9. HTTP POST `PLACEHOLDER - Journal de l'envoi`

URL `https://api.madeleine-riviera.fr/items/envoi_newsletter`, headers `Authorization` = `Bearer PLACEHOLDER` + `Content-Type: application/json`, body (règle 16) :

```
={{ JSON.stringify({ date: $now.toISO(), sujet: $('Compter les envois').item.json.sujet, nb_envoyes: $('Compter les envois').item.json.envoyes, nb_erreurs: $('Compter les envois').item.json.erreurs, statut: "envoye", destinataires: $('Compter les envois').item.json.destinataires }) }}
```

> Collection `envoi_newsletter` à créer côté Directus (champs : `date`, `sujet`, `nb_envoyes`, `nb_erreurs`, `statut`, `destinataires` JSON) — **à confirmer côté devs**.

Never Error ON + timeout 10000 + On Error Continue (le journal ne doit pas faire échouer l'envoi déjà fait).

### 10. Respond `200 - envoi terminé`

```
={{ JSON.stringify({ envoyes: $('Compter les envois').item.json.envoyes, erreurs: $('Compter les envois').item.json.erreurs, total: $('Compter les envois').item.json.total }) }}
```

Code 200 (règles 14 + 16).

### 11. Error Trigger `Erreur détectée` → `Alerte au bureau - erreur workflow`

Copier depuis la n°11/n°12 (racine séparée). Règle 18.

### 12. Save (Ctrl+S)

## Tests de recette

Webhook → « Listen for test event » (1 appel par clic), PowerShell une commande à la fois. Remplace **temporairement** le GET Directus par un Code du **même nom exact** (règle 13) :

- **N1 — envoi normal** (FAUX liste : 3 adhésions dont 1 avec `email` = `email_pro` identique → dédup) :

```javascript
return { json: { data: [
  { id: "A1", email: "a@test.fr", email_pro: "pro1@test.fr", nom_commercial: "Boulangerie Meme" },
  { id: "A2", email: "a@test.fr", email_pro: "", nom_commercial: "Cordonnerie" },
  { id: "A3", email: "", email_pro: "pro3@test.fr", nom_commercial: "Opticien" }
] } };
```

```powershell
Invoke-RestMethod -Uri "https://n8n.acadenice.com/webhook-test/envoi-newsletter" -Method POST -ContentType "application/json" -Body '{"sujet":"Test newsletter","texte":"Bonjour à tous"}'
```

→ attendu : 200 `{"envoyes":3,"erreurs":0,"total":3}` (dédup : a@test.fr une seule fois) ; dans Executions, 3 e-mails envoyés (SMTP bidon → erreur ENOTFOUND attendue, comptée en `erreurs` si On Error Continue la marque) + POST du journal.

- **N2 — aucun abonné** (FAUX liste `{data: []}`) → 200 `{"envoyes":0,...,"remarque":"aucun abonné newsletter"}`.
- **N3 — sujet manquant** → 400 `{"erreur":"sujet et texte requis"}`.

Puis remets le vrai GET, Save, aucun FAUX dans l'export final.

## Livraison

Quand N1–N3 sont OK : ⋯ → **Export** → envoie-moi le JSON (zéro `==`, `pinData` vide, pas de FAUX) → push dans `14-newsletter/workflow-newsletter.json` sur cette branche, merge vers `main` (PR #10).

## Checklist après import

1. Credential **Header Auth** `MR - secret webhook newsletter` sur le webhook (envoyé par la Flow Directus « Envoyer la newsletter », jamais dans le repo).
2. SMTP : remplacer « SMTP account 4 » par le vrai Scaleway TEM.
3. ⋯ → Settings → Error workflow → ce workflow lui-même.
4. Côté devs à confirmer : champ `newsletter` + `email_pro` sur `adhesion`, collection `envoi_newsletter`, et la Flow Directus qui appelle le webhook avec `{sujet, texte}`.

⚠️ Placeholders à remplacer à la livraison : `https://api.madeleine-riviera.fr` + `Bearer PLACEHOLDER`, secret Header Auth.
