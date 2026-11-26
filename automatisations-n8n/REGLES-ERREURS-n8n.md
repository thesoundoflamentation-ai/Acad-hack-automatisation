# Règles anti-erreurs n8n — rétrospective de la n°1 (contrôle SIRET)

Erreurs commises dans le code/JSON de la n°1 (à ne plus reproduire dans les prochains workflows) :

| # | Erreur | Symptôme dans n8n | Règle pour la suite |
|---|---|---|---|
| 1 | `Object.keys($json.body)` sans vérifier le type — le webhook peut envoyer le body en **string** (Content-Type non reconnu) | `A 'json' property isn't an object [item 0]` | Toujours parser défensivement : `let body = $json.body; if (typeof body === "string") { try { body = JSON.parse(body) } catch(e) { body = null } } if (!body \|\| typeof body !== "object" \|\| Array.isArray(body)) body = $json;` |
| 2 | `.first()` dans un nœud **Code** (`$input.first()`, `$('X').first()`) | `Can't use .first() here [item 0]` | Dans un nœud Code : utiliser `$json` (item courant), `$('Nom').all()[0].json` ou `$('Nom').item.json`. Jamais `.first()`. |
| 3 | Double `=` (`==`) dans une expression — on a tapé `=` dans un champ déjà en mode expression (fx bleu) | l'expression est invalide, le nœud plante | Un seul `=` max. Mode fixe (gris) → tape `=expr` ; mode expression (fx bleu) → tape `expr` sans `=`. **Vérif** : ⋯ → Export → chercher `"=="` → aucun. |
| 4 | `rightValue: "==true"` dans le nœud IF (même bug n°3) | condition douteuse | En boolean, préférer l'opération `is true` (singleValue, pas de Value 2). |
| 5 | Lire `$json.champ` au nœud **suivant un nœud en « On Error: Continue »** (ex: PATCH placeholder qui échoue) → l'item transmis est `{error:...}`, pas les données | champ vide/inattendu dans l'e-mail et la réponse webhook | Après un nœud en continue-on-fail, référencer explicitement le nœud amont : `{{ $('Nom du nœud').item.json.champ }}`. |
| 6 | `jsonBody` mal formé (`=={{ ... }}`) sur un HTTP Request en body JSON | erreur d'expression au nœud | Body JSON en expression : `={ "champ": $json.x }` (un seul `=`, pas de guillemets autour de l'objet). |
| 7 | Faute de frappe dans une adresse (`madelaine-riviera.fr`) | e-mail rejeté plus tard | Relire les valeurs en dur avant Save. |
| 8 | Retour du nœud Code : `json` doit être un **objet** | `In the returned data, every key named 'json' must point to an object` | `return [{ json: { ... } }]` ou `return { json: { ... } }` — les deux formes marchent, mais `json` = objet obligatoire. |

Règles générales :
- Toujours **Save** avant de tester (point orange sur l'onglet = non sauvegardé).
- Nœuds externes (HTTP, SMTP) en phase placeholder : **On Error = Continue (regular output)** + **Never Error** + timeout 10 s.
- En mode test, le webhook répond **200 dès la réception** même si l'exécution échoue après → le vrai statut est dans **Executions** (vert/rouge), pas dans le terminal.
- Windows PowerShell : `curl` est un alias cassant → `curl.exe` ou `Invoke-RestMethod`, **une commande à la fois** (coller plusieurs commandes d'un coup ouvre un bloc `>>` et déforme le JSON → 422 de n8n).
- Aucun secret en dur dans le repo GitHub : placeholders uniquement (`Bearer PLACEHOLDER`, `https://api.madeleine-riviera.fr`).

| 9 | Faute de frappe `lenght` au lieu de `length` dans une expression IF (`$json.data.lenght > 0`) | l'expression vaut toujours false → mauvaise branche | Relire les noms de propriétés dans les expressions ; en cas de doute, tester l'expression dans un nœud Set/Debug. |
