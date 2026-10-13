# Recommandations de sécurité (non implémentées)

Ces mesures ne sont PAS appliquées sur la branche : elles nécessitent une
décision upstream, un accès administrateur GitHub, ou présentent un risque de
régression non évaluable depuis l'environnement d'audit.

## CI/CD (ERPSEC-009, ERPSEC-010)

1. **Épingler toutes les actions par SHA** (`actions/checkout`, `setup-node`,
   `cache`, `upload-artifact`, `labeler`, `github-script`, `pre-commit/action`,
   `octokit/request-action`, `dessant/lock-threads`, `codecov/codecov-action`).
   Actuellement par tag (`@v6`, `@v4`...) — le déplacement d'un tag compromis
   exécuterait du code malveillant avec les secrets du dépôt. Dependabot
   (catégorie `github-actions`) sait maintenir l'épinglage.
2. **`backport.yml`** : épingler `frappe/backport` sur un SHA de release au lieu
   de la branche `develop` ; l'action tourne aujourd'hui avec `secrets.RELEASE_TOKEN`.
3. **Branch protection** (paramètres dépôt, hors code) : exiger `semantic-commits`
   et `linters` sur `develop`, interdire le force-push, activer Dependabot npm.

## Endpoints (ERPSEC-011)

4. **Rate limit sur `get_help_results_sections`** (`search_help.py`) : le
   décorateur frappe `rate_limit` doit d'abord être vérifié pour ne pas
   throttlé l'appel interne depuis `get_context` (rendu de la page
   `/search_help`). Alternatives : rate limit applicatif dans `get_response`
   uniquement, ou protection via le rate limiter global du site.

## Dépendances (ERPSEC-008)

5. **`ws` 8.17.1** : suivre la montée de version d'`engine.io-client`/`socket.io`
   (usage dev uniquement aujourd'hui). Si la CI upstream valide le serveur de
   développement, un `resolutions` yarn temporaire vers `^8.21.0` est possible.

## Processus

6. **Semgrep/CodeQL en CI** : le dépôt embarque des règles Semgrep de test
   (`semgrep/test-correctness.yml`) ; ajouter une passe des règles de sécurité
   Frappe (`frappe-semgrep-rules`) en CI empêcherait les régressions sur les
   patterns les plus sensibles (SQL, guest methods).
7. **Secrets scanning** : activer gitleaks/trufflehog en CI sur l'historique
   complet (l'audit local n'a couvert que le snapshot courant, clone depth 1).
8. **Rotation des tokens** : vérifier sur GitHub que `RELEASE_TOKEN` et les
   secrets Crowdin suivent un cycle de rotation annuel.
9. **CVE monitoring continu** : activer Dependabot + GitHub Security Advisories
   sur les deux écosystèmes ; l'audit ponctuel du 2026-10-13 est déjà obsolète
   le jour de sa remise par définition.

## Vulnérabilités externes constatées mais hors dépôt

Aucune. Les modules critiques (auth, sessions, ORM, fichiers) sont fournis par
le framework `frappe/frappe` — recommandé de répliquer cet audit sur ce dépôt.
