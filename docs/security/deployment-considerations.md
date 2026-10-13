# Considérations de déploiement

## Impact des correctifs

Les deux correctifs (timeouts HTTP, bumps npm) sont **sans migration de base de
données, sans variable d'environnement nouvelle, sans changement de contrat
API** et **compatibles ascendantes**.

### 1. Timeouts HTTP (15 s)

- Aucune configuration nouvelle ; la valeur est une constante nommée dans le
  code. Si un intégrateur exploite un service de taux de change ou une API de
  recherche particulièrement lent (> 15 s), il verra désormais des erreurs
  temporaires au lieu de workers suspendus — c'est le changement documenté.
- Les chemins d'erreur applicatifs sont inchangés (les gestionnaires `except`
  existants prennent en charge `requests.Timeout`).

### 2. Dépendances banking

- `banking/yarn.lock` uniquement ; `package.json` inchangé.
- **Reconstruction des assets requise** : les bundles de production de
  `erpnext/public/banking` ne sont pas reconstruits dans cette branche ; le
  pipeline `build-and-commit-assets.yml` doit être exécuté après fusion, ou
  localement : `yarn build` dans `banking/` (le présent audit a validé ce
  build).
- Aucun asset à régénérer côté Python.

## Validation avant mise en production

1. `yarn install --frozen-lockfile && yarn build` dans `banking/` — succès.
2. `bench --site <site> run-tests --app erpnext` (tests serveur non exécutés
   pendant l'audit — voir `baseline.md`).
3. Test manuel : page `/banking` (connexion, affichage des comptes), page
   `/search_help` avec une Support Settings search API configurée, saisie
   d'un taux de change via API externe dans un environnement de recette.
4. Vérifier la présence des CVE résiduelles acceptées (`ws` — dev only) dans
   le rapport de dépendance du CI.

## Plan de rollback

- Les deux correctifs sont portés par des commits séparés, réversibles
  indépendamment (`git revert` sur la branche de sécurité) :
  - commit timeouts : 3 fichiers Python ;
  - commit dépendances : 1 fichier `yarn.lock`.
- Aucune donnée n'est modifiée par les correctifs ; le rollback est purement
  applicatif.
- En cas de régression du module banking : `git revert` du commit de dépendances
  puis reconstruction des assets.

## Actions manuelles requises (aucune n'est bloquante pour le déploiement)

- Reconstruire les assets banking après fusion (CI dédiée).
- Décisions upstream sur `security-recommendations.md` (épinglage SHA CI,
  rate limit search_help, `ws`).
- Aucune rotation de secrets nécessaire au vu de l'audit (aucun secret exposé).
