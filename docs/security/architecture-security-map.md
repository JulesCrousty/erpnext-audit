# Cartographie sécurité — architecture

Référence : commit `08550102c1d1830c843abd1b36b6b647be3a16b8` (branche `develop`).

## 1. Vue d'ensemble

ERPNext est une application ERP construite sur le framework **Frappe** (Python,
serveur HTTP intégré, ORM maison, MariaDB/PostgreSQL, Redis pour cache/files).
Le dépôt contient :

- `erpnext/` : code serveur Python (~3 046 fichiers `.py`) organisé en modules
  métier : `accounts`, `stock`, `buying`, `selling`, `manufacturing`, `crm`,
  `support`, `projects`, `hr` (déplacé vers l'app HRMS), `regional`,
  `shopping_cart`, `portal`, `www`, `edi`, `telephony`, `utilities`.
- `banking/` : interface bancaire React 19 + Vite + TypeScript (424 packages
  npm), construite vers `erpnext/public/banking`, servie sur `/banking`
  (`erpnext/www/banking.py`).
- `erpnext/patches/` : migrations de base de données exécutées par le bench.
- `.github/workflows/` : 24 workflows CI/CD.
- Dépendances Python directes : 12 (pyproject.toml, Python >= 3.14, frappe >= 17).

L'authentification, les sessions, le CSRF, les permissions RBAC/ABAC, la
validation d'entrée de bas niveau et le rate limiting sont fournis par le
framework Frappe — hors périmètre de ce dépôt, mais les usages qui en sont faits
sont dans le périmètre.

## 2. Points d'entrée

| Type | Emplacement | Contrôle d'accès |
|---|---|---|
| API HTTP whitelistée | 841 fonctions `@frappe.whitelist()` dans `erpnext/` | Session utilisateur ; `allow_guest=True` pour 7 d'entre elles |
| Endpoints invités | `erpnext/www/banking.py`, `erpnext/www/book_appointment/index.py`, `erpnext/templates/utils.py`, `erpnext/templates/pages/search_help.py` | Revue individuelle — voir §4 |
| Pages web (Jinja) | `erpnext/www/`, `erpnext/templates/`, portail `erpnext/portal/` | Session + permissions de lecture sur les doctypes liés |
| Uploads | `upload_bank_statement` (`bank_transaction_upload.py`), Data Import Frappe | `frappe.has_permission("Bank Transaction", "create")` |
| Jobs planifiés | `erpnext/hooks.py` (`scheduler_events`) : exécution de méthodes serveur | Serveur uniquement |
| Webhooks entrants | Aucun endpoint webhook dédié dans ce dépôt (les intégrations Stripe/GoCardless vivent dans `frappe/payments`) | N/A |
| CLI | `erpnext/commands/` (`bench update`, migrations, etc.) | Administrateur bench |
| WebSockets | `frappe-realtime` (framework), utilisé par `banking` via `frappe-react-sdk` / socket.io-client | Framework |

## 3. Frontières de confiance

1. **Client (navigateur) / serveur** : les protections frontend (JS du portail,
   validation de formulaire React de `banking`) ne sont jamais des contrôles de
   sécurité ; le serveur applique `frappe.has_permission` / `doc.check_permission`
   sur les opérations sensibles (vérifié par échantillonnage — §4).
2. **Utilisateur / administrateur** : RBAC par rôles Frappe + User Permissions
   par société/entité. Les fonctions whitelistées destructives vérifiées
   (`delete_dimension`, `submit_quality_inspection`, `download_statements`,
   `upload_bank_statement`, `cancel_stock_reservation_entries`, etc.) appliquent
   les permissions serveur.
3. **Application / base de données** : requêtes via ORM Frappe ou `frappe.db.sql`
   paramétré (placeholders `%s` / `%(name)s`). 74 constats Bandit B608 vérifiés
   individuellement : interpolation structurelle (noms de tables/colonnes issus
   de constantes internes), valeurs utilisateur paramétrées ou `frappe.db.escape()`.
   Aucune injection exploitable confirmée.
4. **Application / API externes** : taux de change (`Currency Exchange
   Settings`), APIs de recherche configurées (`Support Settings.search_apis`),
   eDI/code lists (`erpnext/edi`). URLs administrées par un System Manager.
5. **CI/CD / production** : workflows GitHub Actions ; secrets GitHub ;
   `build-and-commit-assets.yml` pousse les assets construits.

## 4. Endpoints accessibles aux invités (revue exhaustive)

| Endpoint | Fichier | Analyse |
|---|---|---|
| `get_context_for_dev` | `erpnext/www/banking.py:32` | POST, protégé par `frappe.conf.developer_mode` ; ne s'exécute qu'en mode dev. RAS. |
| `send_message` | `erpnext/templates/utils.py:12` | Rate limit 10/3 min, e-mail validé, HTML échappé. RAS. |
| `get_help_results_sections` | `erpnext/templates/pages/search_help.py:25` | Interroge des APIs configurées par l'admin ; **requête sortante sans timeout** (corrigé — ERPSEC-001) ; pas de rate limit propre (recommandation). |
| `get_appointment_settings` | `erpnext/www/book_appointment/index.py:33` | Lecture de configuration seule, page désactivable. RAS. |
| `get_timezones` | `index.py:44` | Liste statique. RAS. |
| `get_appointment_slots` | `index.py:51` | Lecture de créneaux libres uniquement. RAS. |
| `create_appointment` | `index.py:110` | POST, rate limit 5/5 min, timezone validée, insertion soumise à la validation du doctype Appointment. RAS. |

## 5. Données sensibles

- Données financières (grand livre, factures, comptes bancaires — module
  `accounts`, `banking`), données personnelles (contacts, employés via HRMS),
  taux de change configurés.
- Mots de passe / tokens : gérés par le framework Frappe (non présents dans ce
  dépôt).
- Aucun secret codé en dur détecté (greps de motifs + revue CI).

## 6. Comportements critiques (revue ciblée)

- Connexion/déconnexion/sessions : framework Frappe (hors dépôt).
- Paiements : `erpnext/payments/` délègue à l'app `payments` ; pas de logique
  de paiement sensible dans ce dépôt.
- Imports/exports : `chart_of_accounts_importer` (permissions vérifiées, noms de
  fichiers temporaires générés par hash), Data Import framework.
- Suppression en masse : `transaction_deletion_record` (contrôles framework).
- Administration : doctypes Settings (System Manager par défaut).
