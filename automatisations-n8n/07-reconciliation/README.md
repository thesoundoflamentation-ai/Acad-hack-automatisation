# N°7 — Réconciliation quotidienne HelloAsso (rattrapage des webhooks perdus)

> **Mise à jour 27/11** : le fichier `workflow-*.json` de ce dossier a été re-pushé avec l'API réelle livrée en clair (`https://site.g3.madeleine-riviera.fr/api/n8n/...`) et les e-mails via `POST /api/n8n/email`. Les secrets restent dans les credentials n8n (id `PLACEHOLDER` dans le repo). Cloudflare / HelloAsso restent en placeholder tant que les vrais compte/jetons ne sont pas fournis.

Branche : `vibe/reconciliation-helloasso-182b13` — à construire **à la main dans n8n**, export JSON envoyé pour push ici après validation.

## Objectif (CdC tech v0.5 §5.1 « Webhook perdu : la réconciliation quotidienne de n8n le rattrape », §7.3 tâche quotidienne)

Tous les jours à 6h (Europe/Paris), le workflow :

1. récupère un token OAuth2 HelloAsso (sandbox en recette) ;
2. liste les **paiements de la veille** côté API HelloAsso (`GET /v5/organizations/{slug}/payments?from=...&to=...&pageSize=100`, réponse paginée `{data: [...], pagination: {...}}`) ;
3. pour chaque paiement `state === "Authorized"` : vérifie l'**idempotence** côté site (GET placeholder `evenement_paiement` par `helloasso_event_id` — même clé unique que la n°5, T-07) ;
4. si l'événement n'existe pas encore → enregistre le paiement côté site (POST placeholder `items/evenement_paiement`, `source: "reconciliation"` — le Flow Directus active règlement + adhésion 12 mois + compte, comme la n°5) et envoie **M-05** au payeur (e-mail lu dans la commande `GET /v5/orders/{orderId}`).

Un webhook déjà traité par la n°5 est simplement ignoré ici (idempotence) ; un webhook perdu est rattrapé. **Pas de webhook entrant** : déclencheur = cron, pas de nœud Respond.

## Schéma des nœuds

```
Schedule (0 6 * * *, Europe/Paris) → Token OAuth2 HelloAsso → Lister les paiements (GET .../payments?from&to)
→ Déplier les paiements (Code, 1 item par paiement)
→ IF "Paiement autorisé ?" (state === "Authorized") ─ false → NoOp
   true → PLACEHOLDER - Événement déjà traité ? (GET) → IF "Déjà traité ?" ─ true → NoOp
      false → Lire la commande HelloAsso (GET /v5/orders/{id}) → PLACEHOLDER - Enregistrer (POST, source=reconciliation)
      → M-05 Bienvenue (rattrapage) → NoOp
```

## Étapes à faire dans n8n

Nouveau workflow **`MR - Réconciliation HelloAsso (quotidienne)`**, OFF :

1. **Schedule Trigger** `Tous les jours à 6h` : Mode `Cron`, Cron Expression `0 6 * * *`, **Timezone `Europe/Paris`** (dans le trigger, à côté de l'expression cron — sinon l'heure est en UTC).
2. **HTTP Request** `Token OAuth2 HelloAsso` : POST `https://api.helloasso-sandbox.com/oauth2/token` (prod : `api.helloasso.com`), Send Body ON, Body Content Type `Form URL Encoded`, champ texte : Body = `grant_type=client_credentials&client_id=PLACEHOLDER&client_secret=PLACEHOLDER` — ⚠️ **sans `=` devant** (un `=` en tête ferait croire à une expression et casserait le nœud, règle 3). Options : Never Error ON + timeout 10000. Settings → On Error = `Continue (regular output)`.
3. **HTTP Request** `Lister les paiements de la veille` : GET, URL (expression, un seul `=`) :
   ```
   =https://api.helloasso-sandbox.com/v5/organizations/madeleine-riviera/payments?from={{ $now.minus(1, "days").toISO() }}&to={{ $now.toISO() }}&pageSize=100
   ```
   Header `Authorization` = `=Bearer {{ $('Token OAuth2 HelloAsso').item.json.access_token }}` (liste déroulante `$('`). Never Error + timeout 15000 + On Error Continue.
4. **Code** `Déplier les paiements` (Run Once for Each Item) — transforme la liste en 1 item par paiement :
```javascript
const d = ($json.data && Array.isArray($json.data)) ? $json.data : [];
return d.map(p => ({ json: {
  payment_id: String(p.id),
  order_id: String(p.orderId != null ? p.orderId : ""),
  montant_centimes: (p.amount != null) ? p.amount : null,
  date_paiement: p.date || "",
  state: p.state || ""
} }));
```
   (liste vide → 0 item → le flux s'arrête là, c'est normal)
5. **IF** `Paiement autorisé ?` : `={{ $json.state === "Authorized" }}` is true → false : nœud **No Operation** `Fin - non autorisé`.
6. **HTTP Request** `PLACEHOLDER - Événement déjà traité ?` : GET, URL (expression) :
   ```
   =https://api.madeleine-riviera.fr/items/evenement_paiement?filter[helloasso_event_id][_eq]={{ $json.payment_id }}&fields=id,statut
   ```
   Header `Authorization` → `Bearer PLACEHOLDER` (mode fixe). Never Error ON + timeout 10000 + On Error Continue. ⚠️ URL sans retour à la ligne final.
7. **IF** `Déjà traité ?` : `={{ $json.data && $json.data.length > 0 }}` ⚠️ `length` is true → true : **No Operation** `Fin - déjà traité` ; false → étape 8.
8. **HTTP Request** `Lire la commande HelloAsso` : GET, URL (expression, un seul `=`) :

   `=https://api.helloasso-sandbox.com/v5/orders/{{ $('Déplier les paiements').item.json.order_id }}`

   ⚠️ **pas `$json.order_id`** : à ce point `$json` = la réponse du GET idempotence (`{data:...}`), pas le paiement → règle 5, on référence `Déplier les paiements`. Header `Authorization` = `=Bearer {{ $('Token OAuth2 HelloAsso').item.json.access_token }}`. Never Error ON + timeout 10000 + On Error Continue.
9. **HTTP Request** `PLACEHOLDER - Enregistrer l'événement de paiement (rattrapage)` : POST `https://api.madeleine-riviera.fr/items/evenement_paiement`, headers `Authorization` → `Bearer PLACEHOLDER`, `Content-Type` → `application/json`. Send Body ON → Specify Body `JSON` → JSON Body (expression, un seul `=`) :
   ```
   ={ "helloasso_event_id": $('Déplier les paiements').item.json.payment_id, "helloasso_order_id": $('Déplier les paiements').item.json.order_id, "montant_centimes": $('Déplier les paiements').item.json.montant_centimes, "date_paiement": $('Déplier les paiements').item.json.date_paiement, "statut": "traite", "source": "reconciliation" }
   ```
   Never Error + timeout 10000 + On Error Continue.
10. **Send Email** `M-05 Bienvenue (rattrapage) - PLACEHOLDER SMTP` (credential SMTP existant, Settings → On Error = `Continue (regular output)`) : ⚠️ le champ Text commence par **un seul** `=` (`=Bonjour ...`) — un `==` en tête = expression invalide (règle 3) :
    - From `=site@madeleine-riviera.fr`, To (expression) : `={{ ($('Lire la commande HelloAsso').item.json.payer || {}).email || "" }}` ← règle 5 : après des nœuds en continue-on-fail, on référence le nœud amont
    - Subject : `=Bienvenue chez Madeleine Riviera — votre adhésion est active`
    - Text (expression, un seul `=` en tête) :
    ```
    =Bonjour {{ ($('Lire la commande HelloAsso').item.json.payer || {}).firstName || "" }},

Votre paiement de {{ ($('Déplier les paiements').item.json.montant_centimes || 0) / 100 }} € a bien été confirmé.

Votre adhésion à l'association Madeleine Riviera est active pour 12 mois.

Activez votre espace commerçant pour publier votre fiche :
https://madeleine-riviera.fr/activer-compte?jeton=PLACEHOLDER

L'association Madeleine Riviera
    ```
11. **No Operation** `Fin - rattrapé` en sortie de M-05.
12. **Save**.

⚠️ **Tous les HTTP externes (5 nœuds) : Never Error ON + timeout + On Error = Continue** ; M-05 : On Error = Continue. Sans ça, un appel qui échoue (pas de vraies clés en placeholder) plante l'exécution.

## Règles anti-erreurs

Voir `../REGLES-ERREURS-n8n.md`. Ici : un seul `=` (règle 3), parse défensif inutile ici (pas de webhook), `length` pas `lenght` (règle 9), références de nœuds via la liste déroulante `$('` (règle 12 : pas d'apostrophe dans les noms de nœuds ici), après un nœud On Error Continue référencer le nœud amont (règle 5), tous les HTTP externes : Never Error + On Error Continue + timeout. Le body OAuth2 en **une seule chaîne urlencoded** (`grant_type=...&client_id=...`, pas de texte libre avec `=` et `,`).

## Tests de recette

Pas de webhook : teste avec le bouton **▶ Execute workflow** (le cron reste programmé, le clic lance une exécution manuelle immédiate). Pas de Pin (version gratuite) → méthode **FAUX GET** (nœuds Code temporaires, **même nom exact** que le HTTP remplacé — supprimer le HTTP d'abord, règle 13) :

- `Lister les paiements de la veille` (R1) :
```javascript
const d = { data: [
  { id: 987654, orderId: 123456, amount: 10000, state: "Authorized", date: "2026-11-26T10:00:00+01:00" },
  { id: 111222, orderId: 222333, amount: 10000, state: "Authorized", date: "2026-11-26T11:00:00+01:00" }
] };
return { json: d };
```
- `PLACEHOLDER - Événement déjà traité ?` (R1) — 987654 déjà traité, 111222 nouveau :
```javascript
const deja = $json.payment_id === "987654";
return { json: deja ? { data: [{ id: "E1", statut: "traite" }] } : { data: [] } };
```
- `Lire la commande HelloAsso` (R1) :
```javascript
return { json: { id: Number($json.order_id), payer: { email: "commercant@test.fr", firstName: "Youssef", lastName: "Abdelli" } } };
```

**R1** → attendu dans Executions : le POST `PLACEHOLDER - Enregistrer...` exécuté **1 seule fois**, pour `payment_id "111222"` (Input : `helloasso_event_id:"111222"`, `source:"reconciliation"`), jamais pour 987654 ; M-05 vers `commercant@test.fr`.

- **R2** : FAUX GET liste = `return { json: { data: [] } };` → exécution OK, aucun POST, aucun e-mail.
- **R3** : liste avec un paiement `state: "Refused"` → ignoré (branche NoOp), aucun POST.

→ puis **remets les vrais nœuds HTTP** (ou supprime les FAUX GET) et Save. Aucun FAUX GET dans l'export final.

## Fichier final

`workflow-reconciliation.json` = export validé et livré (recette R1–R3 OK) : 16 nœuds dont le **Error Trigger** (voir ci-dessous), cron `0 6 * * *` Europe/Paris, ids neufs, zéro `==`, `pinData` vide. Nettoyages appliqués à l'export : URL du GET idempotence restaurée avec le filtre `filter[helloasso_event_id][_eq]={{ $json.payment_id }}`, `jsonBody` du POST en forme objet `={ ... }` avec références `$('Déplier les paiements').item.json.*` (règle 16), `$input.first()` remplacé par `$json` (règle 2), Never Error + On Error Continue sur les 5 HTTP et les 2 e-mails.

## Alerte erreur workflow (Error Trigger — bonus)

Le workflow embarque un second déclencheur racine : **Error Trigger** `Erreur détectée` → e-mail `Alerte au bureau - erreur workflow` (To `bureau@madeleine-riviera.fr`, sujet `⚠️ Échec du workflow n8n : {{ $json.workflow.name }}`, texte avec `lastNodeExecuted`, `error.message`, id et lien de l'exécution). Règle 18. À répliquer sur les autres workflows critiques (n°5 webhook paiement notamment).

## Historique de construction

Voir les étapes ci-dessus (construit à la main dans n8n) et les tests R1–R3.
