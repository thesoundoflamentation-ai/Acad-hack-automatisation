# Guide pas à pas — Workflow n8n « Contrôle SIRET » (Madeleine Riviera)

Projet : site Madeleine Riviera · Cahier tech v0.5 §5.7 (B-14) · Règle : le contrôle informe le bureau, **jamais bloquant**, le commerçant n'est pas averti.

Chaque bloc de code ci-dessous est à copier tel quel. Les délimiteurs `===== DEBUT ... =====` / `===== FIN ... =====` te permettent de sélectionner facilement un seul bloc.

---

## ÉTAPE 0 — Créer le workflow

1. Menu **Projects** → **Add workflow** (ou bouton **+**).
2. Double-clique sur le titre « Untitled » en haut → renomme : `MR - Contrôle SIRET à la demande d'adhésion`
3. Laisse le toggle **OFF** (inactif) pendant les tests.

---

## ÉTAPE 1 — Nœud Webhook (déclencheur)

1. Clique sur le **+** au centre du canvas (ou touche **Tab**), tape `webhook`, choisis **Webhook**.
2. Double-clique sur le nœud → renomme en haut : `Réception demande d'adhésion`
3. Paramètres :
   - **HTTP Method** : `POST`
   - **Path** : `demande-adhesion`
   - **Respond** : `Using 'Respond to Webhook' Node`

---

## ÉTAPE 2 — Nœud Code « Préparer le contrôle »

1. Survole la boule de sortie (à droite) du webhook → **+** → tape `code` → choisis **Code**.
2. Renomme le nœud (double-clic sur le titre) en **exactement** : `Préparer le contrôle` (l'accent compte, un autre nœud y fait référence).
3. En haut du panneau : **Language** = `JavaScript`, **Mode** = `Run Once for Each Item`.
4. Dans l'éditeur de code, colle ce bloc :

===== DEBUT BLOC 1 — CODE DU NŒUD « Préparer le contrôle » =====
```javascript
// Le webhook peut fournir le body en objet OU en chaîne (selon Content-Type) : on parse si besoin.
let body = $json.body;
if (typeof body === "string") { try { body = JSON.parse(body); } catch (e) { body = null; } }
if (!body || typeof body !== "object" || Array.isArray(body)) body = $json;
const siretBrut = String(body.siret || body.numero_siret || body.SIRET || "");
const siret = siretBrut.replace(/[\s.\-]/g, "");
function luhnOk(s){let sum=0;for(let i=0;i<s.length;i++){let d=+s[s.length-1-i];if(i%2===1){d*=2;if(d>9)d-=9;}sum+=d;}return sum%10===0;}
const siretFormatOk = /^\d{14}$/.test(siret) && luhnOk(siret);
return [{
  json: {
    body: {
      demande_id: body.demande_id || body.id || null,
      siret: siretBrut,
      adresse: body.adresse || "",
      representant: body.representant || [body.representant_prenom, body.representant_nom].filter(Boolean).join(" "),
      nom_commercial: body.nom_commercial || "",
      email: body.email || ""
    },
    siret: siret,
    siretFormatOk: siretFormatOk
  }
}];
```
===== FIN BLOC 1 =====

---

## ÉTAPE 3 — Nœud IF « SIRET au bon format ? »

1. Depuis la sortie du nœud Code, ajoute un nœud **IF**, renomme : `SIRET au bon format ?`
2. Dans **Conditions** :
   - **Value 1** : bouton **fx** à droite du champ → `{{ $json.siretFormatOk }}`
   - **Type de comparaison** (sélecteur à côté de l'opération) : choisis **boolean**
   - **Operation** : `is true` (pas besoin de Value 2)
   - Alternative si « is true » introuvable : opération `is equal to` + **Value 2** = `true` (toggle boolean)
   - ⚠️ Ne choisis PAS « string » : la comparaison échouerait (vrai booléen ≠ texte "true").
3. Le nœud IF a deux sorties à droite : **true** (haut) et **false** (bas).

---

## ÉTAPE 4 — Branche « true » : HTTP Request vers l'API de l'État

1. Depuis la sortie **true** de l'IF → **+** → `http request` → **HTTP Request**, renomme : `API Recherche d'entreprises (État)`
2. Paramètres :
   - **Method** : `GET`
   - **URL** : passe en expression (fx ou tape le `=` devant) et colle :

===== DEBUT BLOC 2 — URL DU NŒUD HTTP API =====
```
=https://recherche-entreprises.api.gouv.fr/search?q={{ $json.siret }}&per_page=1&minimal=true&include=siege,dirigeants,matching_etablissements,complements
```
===== FIN BLOC 2 =====

   - **Send Headers** : ON → **Add header** ×2 :
     - Name `accept` → Value `application/json`
     - Name `User-Agent` → Value `madeleine-riviera-site/1.0 (contact: bureau@madeleine-riviera.fr)`
   - **Options** (déplier) :
     - **Timeout** = `10000`
     - **Response → Never Error** : ON
   - Onglet **Settings** (en haut du panneau du nœud) : **On Error** = `Continue (regular output)`

---

## ÉTAPE 5 — Nœud Code « Calculer le résultat du contrôle » (reçoit les 2 branches)

1. Depuis la sortie du HTTP Request → **+** → **Code**, renomme : `Calculer le résultat du contrôle`
2. Language `JavaScript`, Mode `Run Once for Each Item`.
3. Colle ce bloc dans l'éditeur :

===== DEBUT BLOC 3 — CODE DU NŒUD « Calculer le résultat du contrôle » =====
```javascript
function normaliser(s) {
  if (!s) return "";
  return s.toString().normalize("NFD").replace(/[̀-ͯ]/g, "")
    .toUpperCase().replace(/[^A-Z0-9]/g, " ").replace(/\s+/g, " ").trim();
}
function luhnOk(siret) {
  let sum = 0;
  for (let i = 0; i < siret.length; i++) {
    let d = parseInt(siret[siret.length - 1 - i], 10);
    if (i % 2 === 1) { d *= 2; if (d > 9) d -= 9; }
    sum += d;
  }
  return sum % 10 === 0;
}
function validerSiret(brut) {
  const s = (brut || "").toString().replace(/[\s.-]/g, "");
  if (!/^\d{14}$/.test(s)) return { ok: false, siret: s, raison: "14 chiffres requis" };
  if (!luhnOk(s)) return { ok: false, siret: s, raison: "Clé de contrôle invalide" };
  return { ok: true, siret: s };
}
function trouverEtab(rep, siret) {
  const r = rep.results && rep.results[0];
  if (!r) return null;
  const m = (r.matching_etablissements || []).find(e => e.siret === siret);
  if (m) return { unite: r, etab: m };
  if (r.siege && r.siege.siret === siret) return { unite: r, etab: r.siege };
  return { unite: r, etab: r.siege };
}
function adresseCorrespond(adresseSaisie, etab) {
  const a = normaliser(adresseSaisie || "");
  if (!a) return true;
  const cp = (etab.code_postal || "").toString();
  const commune = normaliser(etab.libelle_commune);
  const voie = normaliser((etab.libelle_voie || "") + " " + (etab.numero_voie || ""));
  const cpOk = !cp || a.includes(cp);
  const communeOk = !commune || a.includes(commune);
  const voieOk = !voie.trim() || a.includes(normaliser(etab.libelle_voie || "")) || a.includes(voie);
  return cpOk && communeOk && voieOk;
}
function representantTrouve(repSaisi, unite) {
  const r = normaliser(repSaisi || "");
  if (!r) return null;
  const tokens = r.split(" ").filter(Boolean);
  const dirigeants = (unite.dirigeants || []).filter(d => d.type_dirigeant === "personne physique");
  const trouve = dirigeants.some(d => {
    const dn = normaliser(d.nom || "");
    const dp = normaliser(d.prenoms || "");
    if (!dn) return false;
    if (tokens.length === 1) return dn.includes(tokens[0]) || tokens[0].includes(dn);
    const dansNom = tokens.some(t => dn.includes(t) || t.includes(dn));
    const dansPrenoms = !dp || tokens.some(t => dp.includes(t));
    return dansNom && dansPrenoms;
  });
  if (!trouve && unite.complements && unite.complements.est_entrepreneur_individuel) {
    return tokens.some(t => normaliser(unite.nom_complet || "").includes(t));
  }
  return trouve;
}
function buildResultat(input) {
  const body = input.body || {};
  const v = validerSiret(body.siret);
  const base = {
    effectue_le: new Date().toISOString(),
    source: "recherche-entreprises.api.gouv.fr",
    siret_controle: v.siret || null,
    statut: null, controles: [], donnees_api: null
  };
  const push = (id, ok, label, attendu, saisi) => {
    base.controles.push({ id, ok, label, attendu: attendu || null, saisi: saisi || null });
  };
  if (!v.ok) {
    push("format_siret", false, "Format du SIRET invalide", "14 chiffres avec clé de contrôle valide", body.siret);
    base.statut = "a_verifier";
    return base;
  }
  if (input.apiError || !input.api) {
    base.statut = "non_effectue";
    base.controles.push({ id: "api", ok: false, label: "API indisponible : contrôle non effectué, vérification manuelle requise", attendu: null, saisi: null });
    return base;
  }
  const rep = input.api;
  if (!rep.results || rep.results.length === 0) {
    push("siret_existe", false, "SIRET introuvable ou établissement fermé", "Un établissement actif correspondant au SIRET saisi", v.siret);
    base.statut = "a_verifier";
    return base;
  }
  const { unite, etab } = trouverEtab(rep, v.siret);
  const actif = etab.etat_administratif === "A" && !etab.date_fermeture;
  push("etablissement_actif", actif,
    actif ? "Établissement actif" : "Établissement fermé",
    "Établissement actif (état A)",
    etab.etat_administratif === "A" ? "Actif" : "Fermé" + (etab.date_fermeture ? " le " + etab.date_fermeture : ""));
  const estSiege = etab.est_siege === true || (unite.siege && unite.siege.siret === v.siret);
  const adresseSaisie = body.adresse || "";
  const siegeAilleurs = estSiege && unite.siege && adresseSaisie && !adresseCorrespond(adresseSaisie, unite.siege);
  push("siret_siege", !siegeAilleurs,
    siegeAilleurs ? "SIRET du siège : demander le SIRET du commerce du boulevard" : "SIRET d'établissement (hors siège) ou siège à l'adresse saisie",
    unite.siege ? unite.siege.adresse : null, adresseSaisie || null);
  const adrOk = adresseCorrespond(adresseSaisie, etab);
  push("adresse", adrOk,
    adrOk ? "Adresse conforme" : "Adresse différente de celle du registre",
    etab.adresse || null, adresseSaisie || null);
  const repOk = representantTrouve(body.representant, unite);
  if (repOk !== null) {
    push("representant", repOk,
      repOk ? "Représentant retrouvé parmi les dirigeants connus" : "Représentant non retrouvé",
      "Un dirigeant connu de l'entreprise (registre RNE)", body.representant || null);
  }
  if (unite.statut_diffusion === "P" || etab.statut_diffusion_etablissement === "P") {
    push("diffusion_partielle", false, "Données en diffusion partielle : à vérifier manuellement", null, null);
  }
  base.statut = base.controles.some(c => !c.ok) ? "a_verifier" : "conforme";
  base.donnees_api = {
    siren: unite.siren,
    nom_raison_sociale: unite.nom_raison_sociale || unite.nom_complet,
    enseigne: (etab.liste_enseignes && etab.liste_enseignes[0]) || etab.nom_commercial || unite.nom_raison_sociale,
    adresse: etab.adresse, code_postal: etab.code_postal, commune: etab.libelle_commune,
    activite_principale: etab.activite_principale,
    est_entrepreneur_individuel: !!(unite.complements && unite.complements.est_entrepreneur_individuel)
  };
  return base;
}

// Fusion : branche "true" = réponse de l'API, branche "false" du IF = item de préparation
// ⚠️ Dans un nœud Code, ne JAMAIS utiliser .first() (erreur "Can't use .first() here") : utiliser $json et .all()[0]
const prep = $('Préparer le contrôle').all()[0].json;
const cur = $json;
let api = null, apiError = false;
if (cur && cur.results) api = cur;
else if (cur && cur.body && cur.body.results) api = cur.body;
else if (cur && cur.error) apiError = true;
else if (cur && cur.statusCode && cur.statusCode >= 400) apiError = true;
const resultat = buildResultat({ body: prep.body, api: api, apiError: apiError });
return [{
  json: {
    demande_id: prep.body.demande_id,
    siret: prep.siret,
    statut: resultat.statut,
    controle_siret: resultat,
    resume: resultat.controles.map(c => (c.ok ? "✅ " : "⚠️ ") + c.label +
      (c.ok ? "" : " | attendu : " + (c.attendu || "—") + " | saisi : " + (c.saisi || "—"))).join("\n")
  }
}];
```
===== FIN BLOC 3 =====

4. **Branche la sortie « false » de l'IF sur ce nœud aussi** : tire un fil depuis la boule `false` (bas du nœud IF) jusqu'à `Calculer le résultat du contrôle`. Le nœud a donc 2 entrées (HTTP Request + IF/false), c'est normal.

---

## ÉTAPE 6 — PLACEHOLDER : écrire le résultat via l'API du site

1. Depuis la sortie de `Calculer le résultat du contrôle` → **+** → **HTTP Request**, renomme : `PLACEHOLDER - Écrire le résultat via l'API du site`
2. Paramètres :
   - **Method** : `PATCH`
   - **URL** (expression) :

===== DEBUT BLOC 4 — URL DU PATCH PLACEHOLDER =====
```
={{ $env.API_SITE_URL }}/items/demande_adhesion/{{ $json.demande_id }}
```
===== FIN BLOC 4 =====

   - **Send Headers** ON → Add header ×2 :
     - `Authorization` → `=Bearer {{ $env.DIRECTUS_TOKEN_N8N }}` (expression, avec le `=`)
     - `Content-Type` → `application/json`
   - **Send Body** ON → **Body Content Type** = `JSON` → **Specify Body** = `JSON`
   - **JSON Body** (expression, `=` devant, PAS de guillemets autour de l'objet) :

===== DEBUT BLOC 5 — BODY DU PATCH =====
```
={ "controle_siret": $json.controle_siret }
```
===== FIN BLOC 5 =====

   - Options : Timeout `10000`, Never Error ON
   - Settings → **On Error** = `Continue (regular output)` (tant que l'API des devs n'existe pas, ce nœud échoue mais le flux continue — voulu)

---

## ÉTAPE 7 — E-mail au bureau (M-02, placeholder SMTP)

1. Depuis la sortie du PATCH → **+** → **Send Email**, renomme : `Alerte au bureau (M-02) - PLACEHOLDER SMTP`
2. Paramètres :
   - **From Email** (expression, avec le `=` devant) : `=site@madeleine-riviera.fr` (adresse d'envoi du site ; devra être déclarée/validée chez Scaleway TEM)
   - **To** (expression) :

===== DEBUT BLOC 6 — TO =====
```
={{ $env.BUREAU_EMAIL }}
```
===== FIN BLOC 6 =====

   - **Subject** (expression) :

===== DEBUT BLOC 7 — SUBJECT =====
```
=Nouvelle demande d'adhésion — contrôle SIRET : {{ $json.statut }}
```
===== FIN BLOC 7 =====

   - **Email Format** : `Text`
   - **Text** (expression) :

===== DEBUT BLOC 8 — TEXT DU MAIL =====
```
=Nouvelle demande d'adhésion reçue — contrôle SIRET : {{ $json.statut }}\n\nCommerce : {{ $('Préparer le contrôle').first().json.body.nom_commercial }}\nSIRET saisi : {{ $json.siret }}\nDemande n° : {{ $json.demande_id }}\n\nRésultat (informatif, jamais bloquant — décision humaine) :\n{{ $json.resume }}
```
===== FIN BLOC 8 =====

   - **Credentials** : laisse vide (Scaleway TEM à créer plus tard). Settings → On Error = `Continue`.

---

## ÉTAPE 8 — Réponse au webhook

1. Depuis la sortie de l'e-mail → **+** → **Respond to Webhook**, renomme : `Accusé de réception 200`
2. Paramètres :
   - **Respond With** : `JSON`
   - **Response Body** (expression) :

===== DEBUT BLOC 9 — RESPONSE BODY =====
```
={ "recu": true, "statut_controle": $json.statut, "message": "Demande enregistrée. Le contrôle SIRET est informatif : la décision du bureau reste humaine." }
```
===== FIN BLOC 9 =====

   - Options → **Response Code** : `200`

---

## ÉTAPE 9 — Variables d'environnement (le placeholder)

**Méthode A — UI Variables** (si elle existe ; sur n8n.cloud : **plan Pro uniquement**, sinon invisible) : menu en bas à gauche (avatar/engrenage) → **Settings → Variables → Add variable** :

| Nom | Valeur (placeholder) |
|---|---|
| `API_SITE_URL` | `https://api.madeleine-riviera.fr` |
| `DIRECTUS_TOKEN_N8N` | `PLACEHOLDER` |
| `BUREAU_EMAIL` | `bureau@madeleine-riviera.fr` |

**Méthode B — fichier `.env` du serveur** (si l'UI Variables n'existe pas, comme chez toi) :

1. Sur le VPS, dans le dossier du projet (là où se trouve `docker-compose.yml`) :

===== DEBUT BLOC 9b — COMMANDES .ENV =====
```bash
cd /chemin/du/projet          # dossier contenant docker-compose.yml
nano .env                     # ou vim .env
```
===== FIN BLOC 9b =====

2. Ajoute ces 3 lignes à la fin du fichier (pas d'espace autour du `=`) :

===== DEBUT BLOC 9c — CONTENU .ENV =====
```
API_SITE_URL=https://api.madeleine-riviera.fr
DIRECTUS_TOKEN_N8N=PLACEHOLDER
BUREAU_EMAIL=bureau@madeleine-riviera.fr
```
===== FIN BLOC 9c =====

3. Redémarre le conteneur n8n pour qu'il relise le `.env` (les variables d'env ne sont prises en compte qu'au démarrage) :

===== DEBUT BLOC 9d — REDEMARRAGE =====
```bash
docker compose restart n8n
```
===== FIN BLOC 9d =====

4. Dans le workflow, `{{ $env.API_SITE_URL }}` est alors disponible. Vérifie dans n8n : exécute un nœud qui affiche `{{ $env.BUREAU_EMAIL }}` (ex. un nœud Set de test) pour confirmer.

⚠️ `.env` doit rester dans `.gitignore` (seul `.env.example` sans valeurs est versionné) — ne commit jamais de vraies clés.

**Méthode C — valeurs en dur dans les nœuds (obligatoire sur n8n.cloud gratuit, sans accès au `.env`)** : remplace les 3 expressions `$env` directement dans les nœuds. Pour garder ça propre, ajoute une **note** sur chaque nœud concerné (champ « Notes » en bas du panneau) : « PLACEHOLDER — remplacer ici quand l'API des devs est livrée ».

- Nœud **PLACEHOLDER PATCH** :
  - URL : `https://api.madeleine-riviera.fr/items/demande_adhesion/{{ $json.demande_id }}` (sans `$env`, en expression avec le `=` devant)
  - Header `Authorization` : `Bearer PLACEHOLDER` (en dur)
- Nœud **Send Email** :
  - To Email : `bureau@madeleine-riviera.fr` (en dur)

⚠️ Quand l'API réelle des devs sera livrée, reviens sur ces 3 champs pour y mettre les vraies valeurs (ou passe en variables si l'UI Variables est apparue entre-temps : avatar en bas à gauche → Settings → Variables).

---

## ÉTAPE 10 — Tester (workflow toujours OFF)

**Pas de nouveau nœud** : on réutilise le webhook existant `Réception demande d'adhésion`.

1. Dans n8n, double-clique sur le nœud webhook → clique sur le bouton **« Listen for test event »** (en haut du panneau). n8n affiche « Waiting for test event » + une **URL de test temporaire** du type `https://<ton-compte>.app.n8n.cloud/webhook-test/demande-adhesion`. **Copie cette URL** (bouton copier à côté).
2. Ouvre un **terminal sur ton ordinateur** (séparé de n8n) : Windows → `Win+R` puis `powershell` (ou l'app « Terminal ») ; Mac → Terminal. Ne ferme pas n8n.
3. Dans ce terminal, colle les commandes curl ci-dessous en remplaçant `URL_DE_TEST` par l'URL copiée à l'étape 1. Chaque `Entrée` déclenche une exécution dans n8n.
   - ⚠️ **Windows PowerShell** : `curl` y est un alias de `Invoke-WebRequest` (qui rejette `-H`/`-d`). Utilise **`curl.exe`** (vrai binaire, dans `C:\Windows\System32`) : `curl.exe -X POST "URL" -H "Content-Type: application/json" -d '{...}'`
   - Ou directement sans curl : `Invoke-RestMethod -Uri "URL_DE_TEST" -Method POST -ContentType "application/json" -Body '{"demande_id":"T1",...}'`
4. Retourne dans n8n → menu **Executions** (icône horloge, colonne de gauche) :

===== DEBUT BLOC 10 — COMMANDES CURL DES TESTS =====
```bash
# T1 — attendu : conforme (Boulangerie Meme, 128 bd de la Madeleine, actif)
curl -X POST "URL_DE_TEST" -H "Content-Type: application/json" \
  -d '{"demande_id":"T1","siret":"93088442400019","adresse":"128 boulevard de la Madeleine, 06000 Nice","representant":"Youssef Abdelli","nom_commercial":"Boulangerie Meme","email":"t1@test.fr"}'

# T2 — attendu : a_verifier, "Établissement fermé le 2004-05-31"
curl -X POST "URL_DE_TEST" -H "Content-Type: application/json" \
  -d '{"demande_id":"T2","siret":"44102144100031","adresse":"22 boulevard de la Madeleine 06000 Nice","representant":"Jean Francois Panier"}'

# T3 — attendu : a_verifier, "SIRET du siège" + "Adresse différente" (attendu : 9 rue du Colonel Pierre Avia, 75015 Paris)
curl -X POST "URL_DE_TEST" -H "Content-Type: application/json" \
  -d '{"demande_id":"T3","siret":"35600000000048","adresse":"12 boulevard de la Madeleine 06000 Nice","representant":"Irene Baudry"}'

# T4 — attendu : a_verifier, "SIRET introuvable" (clé valide mais inexistant)
curl -X POST "URL_DE_TEST" -H "Content-Type: application/json" \
  -d '{"demande_id":"T4","siret":"12345678900007","adresse":"1 rue test 06000 Nice","representant":"Test"}'

# T5 — attendu : a_verifier, format invalide (le nœud HTTP API ne doit PAS s'exécuter)
curl -X POST "URL_DE_TEST" -H "Content-Type: application/json" \
  -d '{"demande_id":"T5","siret":"123","adresse":"","representant":""}'
```

===== DEBUT BLOC 10b — MÉTHODE WINDOWS POWERSHELL FIABLE (UNE COMMANDE À LA FOIS) =====
```powershell
# ⚠️ Ne colle JAMAIS plusieurs commandes d'un coup dans PowerShell :
# les lignes vides ouvrent un bloc ">>" et déforment le JSON (n8n renvoie alors 422 "Failed to parse request body").
# Colle UNE commande, Entrée, attends la réponse JSON, puis la suivante.

Invoke-RestMethod -Uri "URL_DE_TEST" -Method POST -ContentType "application/json" -Body '{"demande_id":"T1","siret":"93088442400019","adresse":"128 boulevard de la Madeleine, 06000 Nice","representant":"Youssef Abdelli","nom_commercial":"Boulangerie Meme","email":"t1@test.fr"}'

Invoke-RestMethod -Uri "URL_DE_TEST" -Method POST -ContentType "application/json" -Body '{"demande_id":"T2","siret":"44102144100031","adresse":"22 boulevard de la Madeleine 06000 Nice","representant":"Jean Francois Panier"}'

Invoke-RestMethod -Uri "URL_DE_TEST" -Method POST -ContentType "application/json" -Body '{"demande_id":"T3","siret":"35600000000048","adresse":"12 boulevard de la Madeleine 06000 Nice","representant":"Irene Baudry"}'

Invoke-RestMethod -Uri "URL_DE_TEST" -Method POST -ContentType "application/json" -Body '{"demande_id":"T4","siret":"12345678900007","adresse":"1 rue test 06000 Nice","representant":"Test"}'

Invoke-RestMethod -Uri "URL_DE_TEST" -Method POST -ContentType "application/json" -Body '{"demande_id":"T5","siret":"123","adresse":"","representant":""}'
```
===== FIN BLOC 10b =====

Vérifie aussi que **« Listen for test event »** est toujours actif sur le nœud webhook dans n8n (le mode test se coupe après un moment d'inactivité : reclique dessus si besoin).

3. Lecture des résultats : menu **Executions** → ouvre l'exécution → clique sur le nœud `Calculer le résultat du contrôle` → onglet **Output** : vérifie `statut` et `controle_siret.controles[]` (`ok`, `label`, `attendu`, `saisi`).

| Test | `statut` attendu | Détail |
|---|---|---|
| T1 | `conforme` | 4 contrôles ✅ |
| T2 | `a_verifier` | ⚠️ établissement fermé le 2004-05-31 |
| T3 | `a_verifier` | ⚠️ SIRET du siège + ⚠️ adresse différente |
| T4 | `a_verifier` | ⚠️ SIRET introuvable |
| T5 | `a_verifier` | ⚠️ format invalide, pas d'appel API |

Les nœuds PLACEHOLDER (PATCH + e-mail) seront en erreur dans les exécutions : normal, « On Error: Continue » laisse passer le flux.

---

## ÉTAPE 11 — Activer (plus tard)

Quand le site enverra vraiment les demandes : toggle **ON** en haut à droite. L'URL de prod sera `.../webhook/demande-adhesion` (sans `-test`). À faire uniquement quand le Flow Directus côté site appelle ce webhook ; prévoir un header secret partagé en prod.

---

## Schéma final du canvas

```
[Réception demande d'adhésion] → [Préparer le contrôle] → [SIRET au bon format ?] ──true──→ [API Recherche d'entreprises (État)] ─┐
                                                           │                                                              │
                                                           └────false──────────────────────────────→ [Calculer le résultat du contrôle] → [PLACEHOLDER PATCH] → [Alerte bureau M-02] → [Accusé de réception 200]
```

## Checklist de validation

- [ ] 8 nœuds présents, noms exacts (surtout `Préparer le contrôle` avec l'accent)
- [ ] IF en boolean + `is true`, sorties true/false toutes deux branchées
- [ ] HTTP API : URL en expression (avec `=`), 2 headers, timeout 10000, Never Error ON, On Error = Continue
- [ ] PATCH : URL `{{ $env.API_SITE_URL }}`, header Bearer `{{ $env.DIRECTUS_TOKEN_N8N }}`, body `={ "controle_siret": $json.controle_siret }`, On Error = Continue
- [ ] 3 variables d'env créées (ou valeurs en dur — méthode C)
- [ ] T1→T5 donnent les statuts du tableau ci-dessus

---

## ⚠️ PIÈGE CLASSIQUE — le double `=` (`==`) dans les expressions

Un champ n8n a deux modes :
- mode **fixe** (pastille grise) : tape `=ton expression` — le `=` active le mode expression ;
- mode **expression** (pastille bleue, bouton fx activé) : tape `ton expression` **sans** le `=`, l'UI ajoute le `=` tout seul.

Si tu tapes `=` dans un champ déjà en mode expression, l'export contient `==...` et l'expression est invalide (le nœud plante en erreur).

**Vérification rapide** : menu ⋯ → **Export** (JSON du workflow) → cherche `"=="` : il ne doit jamais y en avoir. Toute expression doit commencer par un seul `=` (`"=..."`).

---

## 🛠 Dépannage — erreur « a json property isn't an object [item 0] » sur un nœud Code

**Cause** : `Object.keys($json.body)` (ou tout accès en profondeur) alors que `$json.body` n'est pas un objet — typiquement le webhook a reçu le body en **texte brut** (Content-Type non reconnu). Corrigé dans le bloc 1 (parse défensif).

**Autre erreur fréquente dans un nœud Code** : `Can't use .first() here [item 0]` → dans un nœud Code, `.first()` est interdit. Remplacer par `$json` (item courant) ou `$('Nom du nœud').all()[0].json` / `$input.all()[0].json`.

**Si l'erreur persiste :**
1. Vérifie que le bloc 1 corrigé est bien collé dans le nœud `Préparer le contrôle` et que le workflow est **sauvegardé** (bouton Save ; un point orange sur l'onglet = non sauvegardé).
2. Executions → exécution en rouge → nœud **webhook** → onglet **Output** → note la structure exacte de `$json` (body en objet ou en string ?).
3. Code de debug temporaire à mettre dans le nœud 2 :

===== DEBUT BLOC DEBUG =====
```javascript
// ⚠️ pas de .first() dans un nœud Code : $json = item courant
const it = { json: $json };
return [{ json: {
  type: typeof it.json,
  isArray: Array.isArray(it.json),
  keys: (it.json && typeof it.json === "object") ? Object.keys(it.json) : null,
  bodyType: typeof (it.json ? it.json.body : undefined),
  bodyPreview: JSON.stringify(it.json ? it.json.body : null).slice(0, 300)
} }];
```
===== FIN BLOC DEBUG =====
