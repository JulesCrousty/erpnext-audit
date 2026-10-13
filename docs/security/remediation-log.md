# Journal de remédiation

Chaque correctif : fichiers, comportement avant/après, justification, risque de
régression, tests exécutés.

## Correctif 1 — Timeouts sur requêtes HTTP sortantes (ERPSEC-001/002/003)

**Commit** : `security(http): add timeouts to outbound requests`

| Fichier | Modification |
|---|---|
| `erpnext/templates/pages/search_help.py` | constante `SEARCH_API_TIMEOUT = 15` ; `requests.get(..., timeout=SEARCH_API_TIMEOUT)` |
| `erpnext/setup/utils.py` | constante `CURRENCY_EXCHANGE_API_TIMEOUT = 15` ; `requests.get(..., timeout=...)` |
| `erpnext/accounts/doctype/currency_exchange_settings/currency_exchange_settings.py` | constante `API_REQUEST_TIMEOUT = 15` ; `requests.get(..., timeout=...)` |

- **Ancien comportement** : appels HTTP sortants sans limite de temps ; un serveur
  cible silencieux suspend le worker jusqu'à la fermeture TCP (potentiellement
  illimitée).
- **Nouveau comportement** : échec au bout de 15 s (connect + read), exception
  `requests.Timeout` traitée par les gestionnaires d'erreurs existants :
  - `search_help.py` : l'exception remonte en erreur serveur (identique à un
    `raise_for_status()` en échec — les timeouts sont déjà partiellement échoués
    côté HTTP aujourd'hui) ;
  - `setup/utils.py` : attrapée par `except Exception` existant → log +
    `frappe.msgprint` + retour `0.0` (strictement identique aux erreurs réseau
    actuelles) ;
  - `currency_exchange_settings.py` : attrapée par
    `except requests.exceptions.RequestException` existant → `frappe.throw`.
- **Choix de la valeur** : 15 s, aligné sur la convention interne existante
  (`GENERICODE_FETCH_TIMEOUT = 15` dans `code_list_import.py`). Valeur large
  par rapport aux latences réelles des API de taux de change et de recherche.
- **Risque de régression** : faible. Seul cas comportemental : un service mettant
  plus de 15 s à répondre passait avant (en bloquant un worker) et échouera
  désormais proprement. Signature des fonctions, formats de réponse, codes
  d'erreur applicatifs inchangés.
- **Tests exécutés** : `python3 -m py_compile` sur les 3 fichiers : OK.
- **Tests non exécutables ici** : tests d'intégration Frappe (bench requis) —
  recommandé de lancer les tests des modules `setup` et `accounts` en CI :
  `bench --site <site> run-tests --app erpnext --module erpnext.setup.utils` et
  les tests `currency_exchange_settings` / `search_help` existants s'il y en a.

## Correctif 2 — Mise à jour des dépendances npm vulnérables (ERPSEC-004/005/006/007)

**Commit** : `security(deps): bump axios, brace-expansion, socket.io-parser, source-map-js`

| Package | Avant | Après | Range déclaré | CVE corrigées |
|---|---|---|---|---|
| axios | 1.18.1 | 1.20.0 | `^1.18.1` | 12 |
| brace-expansion | 5.0.7 / 5.0.9 | 5.0.12 | `^5.0.5` / `^5.0.8` | 5 |
| socket.io-parser | 4.2.6 | 4.2.7 | `~4.2.4` | 1 |
| source-map-js | 1.2.1 | 1.2.2 | `^1.2.1` | 1 |

- **Fichier** : `banking/yarn.lock` uniquement (`package.json` inchangé — toutes
  les nouvelles versions sont dans les ranges déjà déclarés, aucune mise à jour
  majeure, aucune nouvelle dépendance).
- **Ancien/nouveau comportement** : aucun changement d'API ; axios 1.18.1 →
  1.20.0 reste une montée mineure compatible (changelog : correctifs de
  sécurité uniquement pour les chemins utilisés par `frappe-react-sdk`).
- **Tests exécutés** :
  1. `yarn install --frozen-lockfile` (yarn 1.22.22, Node 24) : succès —
     intégrité des tarballs vérifiée par les hashes du lockfile ;
  2. `yarn build` (build de production Vite, config du dépôt) : succès ;
  3. versions installées vérifiées dans `node_modules` :
     axios 1.20.0, brace-expansion 5.0.12, source-map-js 1.2.2,
     socket.io-parser 4.2.7 ;
  4. comparaison baseline : build avec lockfile d'origine = succès (373 ms),
     build avec lockfile corrigé = succès (353 ms), même structure de bundle.
- **Artefacts de build** : les fichiers générés dans `erpnext/public/banking/`
  et `erpnext/www/banking.html` ont été rétablis à l'état du dépôt ; le
  pipeline existant `build-and-commit-assets.yml` reconstruit les assets en CI.
  Les intégrateurs qui déploient depuis le repo doivent reconstruire les assets
  après fusion (voir `deployment-considerations.md`).
- **Risque de régression** : faible — montées de versions mineures/patch dans
  les ranges semver existants, build de production validé avant/après.
- **Non corrigé** : `ws` 8.17.1 (ERPSEC-008) — version corrigée hors range
  déclarée par `engine.io-client` ; usage dev uniquement. Voir
  `dependency-audit.md`.

## Correctifs volontairement NON appliqués (revue humaine requise)

| ID | Motif |
|---|---|
| ERPSEC-008 | Bump `ws` hors range semver du dépendant — risque d'incompatibilité serveur de dev non vérifiable ici. |
| ERPSEC-009 | Épinglage SHA de toutes les actions GitHub — gros diff CI, cassable si SHA mal choisis ; décision upstream. |
| ERPSEC-010 | Épinglage de `frappe/backport` — dépôt externe, décision upstream. |
| ERPSEC-011 | `@rate_limit` sur `get_help_results_sections` — pourrait throttlé le rendu de la page `/search_help` (appel direct depuis `get_context`) ; nécessite une vérification du comportement `rate_limit` en contexte page. |
