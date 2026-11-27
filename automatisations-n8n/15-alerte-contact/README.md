# Alerte e-mail au bureau — nouveau message de contact (faite seul par le groupe)

Branche : `vibe/alerte-contact-182b13` (PR #11) — ✅ **VALIDÉ**, export final nettoyé versionné ici : **`workflow-alerte-contact.json`**.

## Objectif (CdC fonctionnel A-05, B-07)

Quand un visiteur envoie le formulaire de contact (page Contact), une **Flow Directus** (création sur `message_contact`) appelle le webhook n8n. n8n envoie un **e-mail au bureau** avec le contenu du message, et répond 200. Le message reste enregistré et visible du bureau dans Directus même si l'e-mail n'arrive pas.

## Contrat du webhook

`POST /webhook/message-contact` — **Header Auth** (règle 21, credential placeholder `MR - secret webhook contact`) — body (envoyé par la Flow Directus) :

```json
{ "id": "M12", "nom": "Jeanne Dupont", "email": "jeanne@test.fr", "objet": "adhésion", "connu_par": "internet", "message": "Bonjour, ..." }
```

Réponses : 200 `{"recu":true,"id":"M12"}` ou 400 `{"erreur":"message requis"}`.

> Champs `objet` / `connu_par` = ceux de la collection `message_contact` (maquettes 25/11) — à aligner avec le modèle réel côté devs.

## Schéma (4 nœuds + Error Trigger optionnel)

```
Webhook message-contact → Normaliser le message → IF "Message présent ?" ─ false → 400
  true → Alerte bureau (POST /api/n8n/email) → 200 - reçu
Error Trigger (optionnel, racine séparée) → Alerte au bureau - erreur workflow
```

## Étapes

Nouveau workflow **`MR - Alerte contact`**, OFF.

### 1. Webhook `Nouveau message de contact`

POST, Path `message-contact`, Respond = `Using 'Respond to Webhook' Node`, **Authentication = Header Auth** (credential placeholder `MR - secret webhook contact`).

### 2. Code `Normaliser le message`

```javascript
let body = $json.body;
if (typeof body === "string") { try { body = JSON.parse(body); } catch (e) { body = null; } }
if (!body || typeof body !== "object" || Array.isArray(body)) body = $json;
return { json: {
  id: String(body.id || ""),
  nom: String(body.nom || "").trim().slice(0, 200),
  email: String(body.email || "").trim().slice(0, 200),
  objet: String(body.objet || "").trim().slice(0, 200),
  connu_par: String(body.connu_par || "").trim().slice(0, 200),
  message: String(body.message || "").trim().slice(0, 5000),
  present: !!String(body.message || "").trim()
} };
```

### 3. IF `Message présent ?`

`={{ $json.present === true }}` is true → étape 4. false → Respond `400 - message requis` (`={{ JSON.stringify({ erreur: "message requis" }) }}`, code 400, règle 14).

### 4. HTTP `Alerte bureau - nouveau message`

POST `https://site.g3.madeleine-riviera.fr/api/n8n/email` (API livrée — e-mail envoyé par l'API, plus de nœud SMTP), credential **Header Auth** placeholder `MR - API n8n (header)`, header `Content-Type: application/json`, body (règles 3 + 16) :

```
={{ JSON.stringify({ to: "bureau@madeleine-riviera.fr", subject: "Nouveau message de contact — " + $('Normaliser le message').item.json.objet + " (" + $('Normaliser le message').item.json.nom + ")", text: "Nouveau message via le formulaire de contact (id " + $('Normaliser le message').item.json.id + ").\n\nNom : " + $('Normaliser le message').item.json.nom + "\nE-mail : " + $('Normaliser le message').item.json.email + "\nObjet : " + $('Normaliser le message').item.json.objet + "\nConnu par : " + $('Normaliser le message').item.json.connu_par + "\n\nMessage :\n" + $('Normaliser le message').item.json.message + "\n\nÀ traiter dans le back-office (messages de contact)." }) }}
```

Options : **Never Error ON** + timeout 20000, On Error = Continue (une panne d'envoi ne bloque pas le site ; le message est de toute façon dans Directus).

### 5. Respond `200 - reçu`

```
={{ JSON.stringify({ recu: true, id: $('Normaliser le message').item.json.id }) }}
```

Code 200 (règles 14 + 16).

### 6. (Optionnel) Error Trigger `Erreur détectée` → `Alerte au bureau - erreur workflow`

Copier depuis la n°11 (racine séparée, e-mail via `POST /api/n8n/email`). Règle 18.

### 7. Save (Ctrl+S)

## Tests de recette

Webhook → « Listen for test event » (1 appel par clic, recliquer entre chaque test), PowerShell une commande à la fois :

- **M1 — message normal** :
  `Invoke-RestMethod -Uri "https://n8n.acadenice.com/webhook-test/message-contact" -Method POST -ContentType "application/json" -Body '{"id":"M12","nom":"Jeanne Dupont","email":"jeanne@test.fr","objet":"adhésion","connu_par":"internet","message":"Bonjour, je souhaite adhérer."}'`
  → attendu : 200 `{"recu":true,"id":"M12"}` ; dans Executions, le POST `/api/n8n/email` contient le texte du message.
- **M2 — body envoyé en string** (copier le JSON entre quotes dans le `-Body`) → même résultat (parse défensif, règle 1).
- **M3 — message vide** → 400 `{"erreur":"message requis"}`.

## Fichier final ✅

`workflow-alerte-contact.json` = export « MR - Alerte contact » nettoyé (8 nœuds, `pinData` vide, zéro `==`, `active: false`, ids neufs) :

- webhook `message-contact` : POST + **Header Auth**, credential placeholder `MR - secret webhook contact` ;
- 2 nœuds HTTP `POST https://site.g3.madeleine-riviera.fr/api/n8n/email` (API réelle en clair) avec `authentication: genericCredentialType` / `httpHeaderAuth`, credential placeholder `MR - API n8n (header)` — **aucun secret en clair** (vérifié) ;
- Never Error ON sur `Alerte bureau - nouveau message` (On Error Continue, timeout 20 s) ; l'alerte d'erreur du Error Trigger laisse remonter ;
- branches terminées par Respond : `200 - reçu` et `400 - message requis`.

## Livraison

M1–M3 OK → merge vers `main` (PR #11 à fermer dans l'UI).

## Checklist après import

1. Credential **Header Auth** `MR - secret webhook contact` sur le webhook (envoyé par la Flow Directus « nouveau message_contact », jamais dans le repo) + credential **Header Auth** `MR - API n8n (header)` sur le nœud HTTP (secret de l'API livrée).
2. (Optionnel) ⋯ → Settings → Error workflow → ce workflow lui-même.
3. Côté devs : Flow Directus qui appelle `POST /webhook/message-contact` (avec le header auth) à chaque création sur `message_contact`, champs envoyés = `id, nom, email, objet, connu_par, message`.

ℹ️ L'API réelle (`https://site.g3.madeleine-riviera.fr/api/n8n`) est en clair dans le JSON — pas un secret ; seuls les credentials sont en placeholder.
