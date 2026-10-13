# Rapport de non-régression

## Résumé avant / après

| Vérification | Avant | Après | Verdict |
|---|---|---|---|
| Compilation syntaxe Python (compileall, 3 046 fichiers) | 3 044 OK, 2 échecs (syntaxe `match` ≥ 3.10, limite Python 3.9 local) | identique | aucune régression |
| Syntaxe des 3 fichiers Python modifiés (`py_compile`) | OK | OK | aucune régression |
| `yarn install --frozen-lockfile` (banking, 424 packages) | OK | OK | aucune régression |
| Build de production Vite (`yarn build`) | succès, 373 ms | succès, 353 ms | aucune régression |
| Structure du bundle généré | référence | identique (mêmes chunks, mêmes tailles à hachage près) | aucune régression |
| Tests serveur Frappe | non exécutables (bench requis — voir `baseline.md` §Limites) | idem | **non vérifiés** |

## Changements de comportement volontaires

1. **Timeouts HTTP (ERPSEC-001/002/003)** : un service externe qui mettait plus
   de 15 s à répondre, en bloquant un worker Python, échoue désormais proprement
   après 15 s. Chemins d'erreur inchangés (les exceptions `Timeout` sont
   déjà prises en charge par les gestionnaires `except` existants des trois
   sites appelants — vérifié fichier par fichier).
2. **Versions npm (ERPSEC-004/005/006/007)** : montées patch/mineure dans les
   ranges semver déclarés ; build de production identique.

Aucune route supprimée, aucun endpoint modifié, aucune signature de fonction
modifiée, aucun schéma de données modifié, aucun test modifié ou supprimé.

## Zones non testées (transparence)

- **Tests unitaires/intégration serveur** : non exécutés sur cette machine
  (Python 3.14 + bench Frappe requis). Commandes de validation recommandées
  en CI après fusion :
  - `bench --site <site> run-tests --app erpnext`
  - ciblés : `erpnext.setup.utils`, `currency_exchange_settings`,
    `bank_transaction`, `templates/pages/search_help`.
- **Comportement runtime des APIs de taux de change configurées** : les
  timeout n'ont pas été observés en conditions réelles (aucune instance).
- **Serveur de développement vite/socket.io** : non démarré ; le bump de
  `socket.io-parser` est validé par build seul.
- **Interface bancaire en conditions réelles** : le bundle construit n'a pas
  été exercé dans un navigateur contre un site Frappe.

## Niveau de confiance

Moyen-élevé : les modifications sont minimales, typées, validées statiquement
et par build de production avant/après ; les chemins d'erreur impactés ont été
vérifiés manuellement. La validation dynamique complète reste à la CI upstream.
