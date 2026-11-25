# Automatisations n8n — Madeleine Riviera

Projet : site de l'association des commerçants du boulevard de la Madeleine (Nice).
Sources : cahier des charges fonctionnel v0.4, cahier des charges technique v0.5, diagramme Excalidraw des processus.

Stack : Astro (public) + Directus (back-office/API) + **n8n** (automatisations) + PostgreSQL + HelloAsso (paiement) + Scaleway TEM (e-mails) + Cloudflare Workers AI (assistants).

## Règle de livraison

Une automatisation à la fois, validée avant de passer à la suivante. Chaque dossier contient :
- `workflow-<nom>.json` : export n8n, à importer (⋯ → Import from File, ou Ctrl+V du JSON sur le canvas)
- `guide-etape-par-etape.md` : construction manuelle nœud par nœud + tests de recette

**Placeholder** : tant que l'API du site (Directus) n'est pas livrée par les devs, les appels vers elle passent par des champs marqués `PLACEHOLDER` (URL `https://api.madeleine-riviera.fr`, header `Bearer PLACEHOLDER`) — à remplacer en un seul endroit par workflow quand l'API arrive.

## Pièges n8n à retenir

- Jamais de `==` dans une expression : mode fixe → tape `=expr` ; mode expression (fx bleu) → tape `expr` sans `=`.
- Dans un nœud **Code** : jamais `.first()` (« Can't use .first() here ») → utiliser `$json` (item courant) et `$('Nom').all()[0].json` / `$input.all()[0].json`.
- Le webhook peut recevoir le body en string : toujours parser défensivement (`typeof body === "string" → JSON.parse`).
- Windows PowerShell : `curl` est un alias cassant → utiliser `curl.exe` ou `Invoke-RestMethod`, **une commande à la fois**.

## Index des automatisations (ordre prévu)

| # | Dossier | Automatisation | Statut |
|---|---|---|---|
| 1 | `01-controle-siret/` | Contrôle SIRET à la demande d'adhésion (API Recherche d'entreprises, B-14 §5.7) — jamais bloquant, bureau seul averti | 🟡 logique validée en local (Node, 7 cas), en test dans n8n (T1→T5) |
| 2 | `02-notifications-base/` | E-mails M-01/M-02/M-03 (confirmation demande, alerte bureau, refus avec motif) | à faire |
| 3 | `03-validation-paiement/` | Validation bureau → adhésion « En attente de paiement » + lien `/payer/{jeton}` + e-mail M-04 | à faire |
| 4 | `04-checkout-helloasso/` | Création checkout HelloAsso API v5 (100 €, métadonnée id adhésion) | à faire |
| 5 | `05-webhook-paiement/` | Webhook HelloAsso → relire commande → règlement + adhésion active (12 mois) + compte + e-mail M-05 (idempotent) | à faire |
| 6 | `06-moderation-fiches/` | Proposition fiche → alerte bureau ; validation → M-06 ; refus → M-07 | à faire |
| 7 | `07-reconciliation/` | Réconciliation quotidienne HelloAsso (webhooks perdus) | à faire |
| 8 | `08-rappels-echeances/` | Cron quotidien : relance M-08 (>3 j), rappels M-09 (J-90, J-30), échéance → fiche inactive + M-10 | à faire |
| 9 | `09-photos/` | Toutes les 5 min : contrôle type réel, ré-encodage (EXIF/GPS supprimés), publication, refus notifié | à faire |
| 10 | `10-purges-sauvegarde/` | Cron : purges RGPD (messages 1 an, demandes 1 an, comptes échus 6 mois, conversations chatbot 30 j) + dump PG chiffré → S3 + export workflows | à faire |
| 11 | `11-copilote-ia/` | Correction obligatoire du texte : Workers AI + skill → 2 versions ≤ 300 car. → filtre mots interdits → Directus → alerte rouge si illégal (CP-01…CP-14) | à faire |
| 12 | `12-chatbot-ia/` | Index pgvector des contenus publics + webhook question → Llama 3.3 70B → réponse + liens ; repli recherche simple si quota (IA-01…IA-09) | à faire |
| 13 | `13-bilan-assistant/` | Bilan mensuel B-12 au bureau (questions, sans réponse, jours quota épuisé) | à faire |
| 14 | `14-newsletter/` | Newsletter (hors MVP, processus Excalidraw) | à faire |

## Celle que le groupe fait seul

L'alerte e-mail au bureau sur nouveau message de contact (webhook → e-mail) : la plus simple (3 nœuds), pour prendre en main n8n.
