# N°8 — Rappels et échéances (cron quotidien : M-08, M-09 ×2, M-10)

Branche : `vibe/rappels-echeances-182b13` — à construire **à la main dans n8n**, export JSON envoyé pour push ici après validation.

## Objectif (CdC fonctionnel B-10 / §6.4, CdC tech §7.2 M-08/M-09/M-10, §7.3 tâche quotidienne)

Tous les jours à 7h (Europe/Paris), trois traitements en parallèle depuis le même cron :

| Branche | Déclencheur | Action |
|---|---|---|
| A — M-08 | demande `en_attente_paiement` depuis **plus de 3 jours**, pas relancée depuis 24h | e-mail M-08 avec le lien `/payer/{payer_token}` + PATCH `m08_dernier_envoi` |
| B — M-09 | adhésion `active` dont la `date_fin` est dans **90 jours ou moins** | si `m09_dernier_rappel` ≠ la cible : e-mail M-09 (J-90 à 3 mois, J-30 à 30 jours) + PATCH `m09_dernier_rappel` (pas de doublon si la tâche repasse, R-46) |
| B — M-10 | adhésion `active` dont la `date_fin` est **dépassée** | PATCH `statut=echue` (la fiche passe en « inactive » via le Flow Directus, statut calculé §5.2) + e-mail M-10 (données gardées 6 mois) |

Plus un **Error Trigger** → e-mail au bureau (même pattern que la n°7, règle 18).

Toutes les lectures/écritures passent par l'**API du site en placeholder** (`https://api.madeleine-riviera.fr`, `Bearer PLACEHOLDER`).

## Schéma

```
Schedule 0 7 * * * (Europe/Paris) ─┬─→ A: GET demandes → Code Déplier demandes → IF À relancer ? → M-08 → PATCH m08 → NoOp
                                   └─→ B: GET adhésions → Code Déplier adhésions → Switch Quelle action ?
                                          ├─ "rappel" → M-09 → PATCH m09 → NoOp
                                          ├─ "echue"  → PATCH statut=echue → M-10 → NoOp
                                          └─ fallback → NoOp
Error Trigger (racine séparée) → Alerte au bureau - erreur workflow
```

Astuce : branche le cron sur **les deux premiers GET en parallèle** (3 fils depuis le trigger : GET A, GET B — et l'Error Trigger est une racine à part, non reliée au cron).

## Étapes

Nouveau workflow **`MR - Rappels et échéances (quotidien)`**, OFF.

### 1. Schedule Trigger `Tous les jours à 7h`

Mode Cron, expression `0 7 * * *`, **Timezone `Europe/Paris`** (règle 17).

### 2. Branche A — GET `PLACEHOLDER - Demandes en attente de paiement`

GET, URL (expression, un seul `=`) :

```
=https://api.madeleine-riviera.fr/items/demande_adhesion?filter[statut][_eq]=en_attente_paiement&fields=id,email,nom_commercial,payer_token,date_creation,m08_dernier_envoi&pageSize=100
```

Header `Authorization` = `Bearer PLACEHOLDER` (mode fixe). Never Error ON + timeout 10000 + On Error Continue.

### 3. Code `Déplier demandes`

```javascript
const d = ($json.data && Array.isArray($json.data)) ? $json.data : [];
const maintenant = new Date();
return d.map(p => {
  const cree = p.date_creation ? new Date(p.date_creation) : null;
  const dernier = p.m08_dernier_envoi ? new Date(p.m08_dernier_envoi) : null;
  const aRelancer = !!cree
    && (maintenant - cree) > 3 * 86400000
    && (!dernier || (maintenant - dernier) > 86400000);
  return { json: {
    id: String(p.id), email: p.email || "", nom_commercial: p.nom_commercial || "",
    payer_token: p.payer_token || "", a_relancer: aRelancer
  } };
});
```

### 4. IF `À relancer ?`

`={{ $json.a_relancer === true }}` is true → true : M-08 ; false : NoOp `Fin - non relancé`.

### 5. Send Email `M-08 Paiement à finir - PLACEHOLDER SMTP` (On Error Continue)

From `=site@madeleine-riviera.fr`, To `={{ $json.email }}` (à ce point `$json` = l'item du Déplier, l'IF le passe tel quel).

Subject :

```
=Relance — votre cotisation Madeleine Riviera est en attente
```

Text :

```
=Bonjour {{ $json.nom_commercial || "cher commerçant" }},

Votre demande d'adhésion (n° {{ $json.id }}) a été validée, mais le paiement n'a pas été effectué.

Pour régler votre cotisation (100 €, 12 mois) :
https://madeleine-riviera.fr/payer/{{ $json.payer_token }}

L'association Madeleine Riviera
```

### 6. PATCH `PLACEHOLDER - Marquer M-08 envoyée`

Method PATCH. URL (expression) — ⚠️ règle 16 : après l'e-mail, `$json` = item d'erreur, on référence le nœud amont :

```
=https://api.madeleine-riviera.fr/items/demande_adhesion/{{ $('Déplier demandes').item.json.id }}
```

Headers `Authorization` = `Bearer PLACEHOLDER`, `Content-Type` = `application/json`. Body JSON (un seul `=`) :

```
={ "m08_dernier_envoi": $now.toISO() }
```

Never Error + timeout 10000 + On Error Continue → NoOp `Fin - M-08 envoyée`.

### 7. Branche B — GET `PLACEHOLDER - Adhésions actives`

```
=https://api.madeleine-riviera.fr/items/adhesion?filter[statut][_eq]=active&fields=id,email,nom_commercial,date_fin,m09_dernier_rappel&pageSize=100
```

Idem : header Bearer PLACEHOLDER, Never Error + timeout 10000 + On Error Continue.

### 8. Code `Déplier adhésions`

```javascript
const d = ($json.data && Array.isArray($json.data)) ? $json.data : [];
const maintenant = new Date();
return d.map(a => {
  const fin = a.date_fin ? new Date(a.date_fin) : null;
  let action = "rien", cible = "", jours = null;
  if (fin && fin < maintenant) {
    action = "echue";
  } else if (fin) {
    jours = Math.ceil((fin - maintenant) / 86400000);
    if (jours <= 30) { action = "rappel"; cible = "J-30"; }
    else if (jours <= 90) { action = "rappel"; cible = "J-90"; }
    if (action === "rappel" && a.m09_dernier_rappel === cible) action = "rien";
  }
  return { json: {
    id: String(a.id), email: a.email || "", nom_commercial: a.nom_commercial || "",
    date_fin: a.date_fin || "", action, cible, jours_restants: jours
  } };
});
```

### 9. Switch `Quelle action ?`

Règle 1 : `={{ $json.action }}` equals `rappel` → sortie 0. Règle 2 : equals `echue` → sortie 1. Fallback Output = `extra` → sortie 2.

### 10. Sortie « rappel » — Send Email `M-09 Rappel renouvellement - PLACEHOLDER SMTP` (On Error Continue)

From `=site@madeleine-riviera.fr`, To `={{ $json.email }}`.

Subject :

```
=Rappel — votre adhésion Madeleine Riviera expire dans {{ $json.jours_restants }} jours
```

Text :

```
=Bonjour {{ $json.nom_commercial || "cher commerçant" }},

Votre adhésion à l'association Madeleine Riviera (adhésion n° {{ $json.id }}) arrive à échéance le {{ $json.date_fin }} — dans {{ $json.jours_restants }} jours ({{ $json.cible }}).

Sans renouvellement, votre fiche passera en « inactive » et sortira de l'annuaire à cette date.

Pour renouveler (12 mois) : connectez-vous à votre espace commerçant sur madeleine-riviera.fr.

L'association Madeleine Riviera
```

Puis PATCH `PLACEHOLDER - Marquer M-09 envoyée` : PATCH `=https://api.madeleine-riviera.fr/items/adhesion/{{ $('Déplier adhésions').item.json.id }}` (règle 16), body :

```
={ "m09_dernier_rappel": $('Déplier adhésions').item.json.cible }
```

Never Error + On Error Continue → NoOp `Fin - M-09 envoyée`.

### 11. Sortie « echue » — PATCH puis M-10

PATCH `PLACEHOLDER - Adhésion échue` : PATCH `=https://api.madeleine-riviera.fr/items/adhesion/{{ $('Déplier adhésions').item.json.id }}`, body :

```
={ "statut": "echue" }
```

> ⚠️ La fiche passe en « inactive » côté Directus (Flow sur mise à jour de l'adhésion, statut calculé §5.2) — n8n ne touche qu'à `adhesion.statut`.

Puis Send Email `M-10 Adhésion échue - PLACEHOLDER SMTP` (On Error Continue) — après le PATCH, référencer le nœud amont (règle 16) :

- To : `={{ $('Déplier adhésions').item.json.email }}`
- Subject : `=Votre adhésion Madeleine Riviera est arrivée à échéance`
- Text :

```
=Bonjour {{ $('Déplier adhésions').item.json.nom_commercial || "cher commerçant" }},

Votre adhésion n° {{ $('Déplier adhésions').item.json.id }} est arrivée à échéance le {{ $('Déplier adhésions').item.json.date_fin }}.

Votre fiche est désormais « inactive » : elle n'est plus visible sur madeleine-riviera.fr. Vos données sont conservées 6 mois.

Pour remettre votre fiche en ligne, renouvelez depuis votre espace commerçant.

L'association Madeleine Riviera
```

→ NoOp `Fin - M-10 envoyée`. Sortie fallback du Switch → NoOp `Fin - rien`.

### 12. Error Trigger `Erreur détectée` → Send Email `Alerte au bureau - erreur workflow`

Copier les 2 nœuds de la n°7 (racine séparée, non reliée au cron) : To `=bureau@madeleine-riviera.fr`, Subject `==⚠️...` non — **un seul `=`** : `=⚠️ Échec du workflow n8n : {{ $json.workflow.name }}`, Text avec `{{ $json.execution.lastNodeExecuted }}`, `{{ $json.execution.error.message }}`, `{{ $json.execution.url }}`. On Error Continue sur l'e-mail.

### 13. Save (Ctrl+S)

## Règles anti-erreurs

Voir `../REGLES-ERREURS-n8n.md` (18 règles). Ici en particulier : règle 3 (un seul `=`), règle 16 (`$json` après IF/HTTP/e-mail = sortie du dernier nœud → référencer `$('Nœud amont').item.json`), règle 17 (timezone Paris, pas de `\n` dans les URLs), règle 15 (pas de `=` devant le body OAuth s'il y en avait un), Never Error + On Error Continue + timeout sur tous les HTTP, références `$('...')` via la liste déroulante.

## Tests de recette

Bouton **▶ Execute workflow** (exécution manuelle). Pas de Pin → **FAUX GET** (Code temporaire, même nom exact que le GET remplacé — supprimer le HTTP d'abord, règle 13).

Faux GET `PLACEHOLDER - Demandes en attente de paiement` (E1) :

```javascript
const d = { data: [
  { id: "D1", email: "a@test.fr", nom_commercial: "Boulangerie Meme", payer_token: "tok1", date_creation: new Date(Date.now() - 5*86400000).toISOString(), m08_dernier_envoi: null },
  { id: "D2", email: "b@test.fr", nom_commercial: "Chez Marcel", payer_token: "tok2", date_creation: new Date(Date.now() - 86400000).toISOString(), m08_dernier_envoi: null }
] };
return { json: d };
```

Faux GET `PLACEHOLDER - Adhésions actives` (E2) :

```javascript
const j = 86400000;
const d = { data: [
  { id: "A1", email: "a@test.fr", nom_commercial: "Boulangerie Meme", date_fin: new Date(Date.now() + 60*j).toISOString(), m09_dernier_rappel: null },
  { id: "A2", email: "b@test.fr", nom_commercial: "Chez Marcel", date_fin: new Date(Date.now() + 20*j).toISOString(), m09_dernier_rappel: "J-90" },
  { id: "A3", email: "c@test.fr", nom_commercial: "Tabac Presse", date_fin: new Date(Date.now() - 1*j).toISOString(), m09_dernier_rappel: null },
  { id: "A4", email: "d@test.fr", nom_commercial: "Coiffeur", date_fin: new Date(Date.now() + 60*j).toISOString(), m09_dernier_rappel: "J-90" }
] };
return { json: d };
```

Attendus dans **Executions** :

- **E1** : M-08 envoyé **1 fois** (D1 uniquement) + PATCH `m08_dernier_envoi` 1 fois (URL contient `/demande_adhesion/D1`)
- **E2** : M-09 envoyé 2 fois (A1 avec J-90/60 jours, A2 avec J-30/20 jours) + PATCH `m09_dernier_rappel` 2 fois ; A3 → PATCH `statut: "echue"` (URL `/adhesion/A3`) + M-10 vers `c@test.fr` ; A4 → rien
- **E3** : listes `{ data: [] }` → aucun e-mail, aucune PATCH

Puis remets les vrais nœuds HTTP et Save. Aucun FAUX GET dans l'export final.

## Livraison

Quand E1–E3 sont OK : ⋯ → **Export** → envoie-moi le JSON (vérifie : zéro `==`, `pinData` vide, pas de FAUX GET) → push dans `08-rappels-echeances/workflow-rappels-echeances.json` sur cette branche, merge vers `main` (PR #7).
