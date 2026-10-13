# Baseline fonctionnelle — état avant correctifs

Référence : commit `08550102c1d1830c843abd1b36b6b647be3a16b8` (branche `develop`),
enregistré avant toute modification. Environnement d'audit : macOS, Python 3.9.6
système, Node v24.14.1, yarn 1.22.22 (via npx), bandit 1.8.6.

## Limites environnementales (à lire en premier)

1. **Python local 3.9.6 vs `requires-python >= 3.14`** : la suite de tests
   serveur d'ERPNext nécessite un bench Frappe v17 (MariaDB/PostgreSQL, Redis,
   Python >= 3.14) non installable sur la machine d'audit dans le temps imparti.
   Les tests Python n'ont donc **pas** été exécutés.
2. Le dépôt cloné avec `--depth 1`, l'historique Git complet n'est pas disponible
   pour la chasse aux secrets historiques.
3. Les 2 fichiers utilisant la syntaxe `match` (Python 3.10+) ne peuvent pas être
   compilés par l'interpréteur 3.9 local (limite d'outillage, pas un défaut) :
   - `erpnext/controllers/subcontracting_inward_controller.py`
   - `erpnext/stock/doctype/serial_and_batch_bundle/serial_and_batch_bundle.py`

## Vérifications exécutées (avant)

| Vérification | Commande | Résultat |
|---|---|---|
| Compilation syntaxe | `python3 -m compileall erpnext/` | 3 044/3 046 fichiers OK ; 2 échecs documentés ci-dessus (syntaxe 3.10+) |
| SAST Python | `bandit 1.8.6 -r erpnext/ banking/ -ll` | 0 critique/haute ; 77 moyenne (74 B608 SQL — triés : interpolations structurelles, aucun flux d'entrée utilisateur injectable ; 3 B113 requêtes sans timeout — vulnérabilités réelles, corrigées) |
| Lint JS/TS | `yarn build` (vite, config dépôt) | Succès, build en 373 ms, bundle généré |
| Installation deps JS | `yarn install --frozen-lockfile` | Succès (424 packages, lockfile cohérent) |
| Dépendances Python | OSV.dev (API querybatch), 12 deps directes | 0 vulnérabilité connue |
| Dépendances npm | OSV.dev (API querybatch), 424 packages du lockfile | 21 vulnérabilités — voir `dependency-audit.md` |
| Secrets | greps de motifs (api_key/secret/password/token = littéraux) sur `.py/.js/.yml/.json` | 0 constat |
| Commandes sensibles | greps subprocess / os.system / os.popen / eval / exec / pickle / yaml.load | 0 usage en code de production |
| Crypto | greps hashlib.md5/sha1, verify=False | 0 usage |
| CI | lecture des 24 workflows | permissions explicites restrictives présentes ; 4 `pull_request_target` vérifiés (aucun checkout de code de PR externe avec secrets) |

## Comportements enregistrés (caractérisation)

- `search_help.get_response` : requête HTTP sortante sans timeout vers une URL
  configurée dans Support Settings (`search_apis`). Un appel peut suspendre un
  worker indéfiniment si le serveur cible ne répond jamais — c'est le
  comportement pré-correction, considéré comme vulnérable et modifié
  (ERPSEC-001).
- `setup.utils.get_exchange_rate` : requête sans timeout, enveloppée dans un
  `try/except Exception` qui journalise l'erreur et retourne 0.0 — les
  exceptions `requests.Timeout` y sont déjà gérées sans crash.
- `CurrencyExchangeSettings.validate_parameters` : requête sans timeout,
  enveloppée dans `except requests.exceptions.RequestException` — les
  exceptions `Timeout` (sous-classe de `RequestException`) y sont déjà gérées.

Aucun test existant modifié, supprimé ou contourné.
