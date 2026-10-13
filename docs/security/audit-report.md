# Rapport d'audit de sécurité

- **Dépôt** : `frappe/erpnext` (clone local `/Users/julfav/Desktop/ZAZA/erpnext`)
- **Branche de référence** : `develop` — commit `08550102c1d1830c843abd1b36b6b647be3a16b8`
- **Branche de sécurité** : `security/comprehensive-hardening-20261013`
- **Date** : 2026-10-13

## 1. Périmètre

- Code serveur Python : 3 046 fichiers (`erpnext/`), frontend TypeScript/React
  (`banking/`, 424 packages npm), 24 workflows GitHub Actions, dépendances
  Python (12 directes), configurations de build et lint.
- Le framework Frappe lui-même (authentification, sessions, CSRF, ORM) est
  hors dépôt ; ses usages dans ERPNext sont dans le périmètre.

## 2. Méthodologie

Phases exécutées : reconnaissance, baseline (compileall, bandit, build JS,
installation des deps), audit SAST par motifs + revue manuelle ciblée, audit
SCA (OSV.dev, API querybatch sur chaque version verrouillée), chasse aux
secrets, audit CI/CD, correction minimale, validation avant/après,
documentation.

## 3. Outils et versions

| Outil | Version | Usage |
|---|---|---|
| bandit | 1.8.6 (venv isolé) | SAST Python, seuil sévérité moyenne+ |
| python3 compileall | 3.9.6 (limite : dépôt exige >= 3.14) | contrôle de syntaxe de masse |
| OSV.dev API | querybatch + /v1/vulns | SCA Python et npm, versions exactes du lockfile |
| yarn / vite | 1.22.22 / config du dépôt (Node 24.14.1) | installation gelée + build de production |
| grep / scripts Python | - | motifs : SQLi, injection de commandes, désérialisation, secrets, crypto faible, verify=False, allow_guest, opérations destructives whitelistées |

## 4. Constats principaux

1. **Hygiène de code élevée.** Aucun subprocess/eval/exec/pickle en production,
   aucun hash faible (MD5/SHA1), aucun `verify=False`, aucun secret codé en dur,
   SQL systématiquement paramétré, endpoints invités pourvus de rate limits et
   de validation. 7 endpoints invités revus individuellement : protégés.
2. **Vulnérabilités réelles corrigées (7) :**
   - 3 requêtes HTTP sortantes sans timeout (CWE-400), dont 1 sur endpoint
     accessible sans authentification ;
   - 4 packages npm vulnérables mis à jour dans leurs ranges semver
     (20 CVE corrigées).
3. **Risques résiduels documentés (5)** : `ws` (dev-only, hors range), épinglage
   SHA des actions GitHub, `backport.yml` + token de release, rate limit absent
   sur un endpoint invité, dépendance aux releases Frappe.
4. **Faux positifs triés et vérifiés** : 74 alertes SQL B608 (pattern Frappe),
   3 endpoints destructifs protégés par le framework ou par vérifications
   explicites.

## 5. Limites de l'audit

- **Tests applicatifs non exécutés** : la suite de tests serveur requiert un
  bench Frappe v17 (MariaDB/PostgreSQL, Redis, Python >= 3.14), non disponible
  sur la machine d'audit. Aucune exécution de test Frappe n'est revendiquée.
- **Historique Git non audité** (clone `--depth 1`) : les secrets éventuels dans
  l'historique complet n'ont pas été recherchés.
- **Couverture des 841 endpoints whitelistés** : tri automatisé par motifs
  (méthodes, verbes sensibles, permissions) + revue manuelle de tous les
  endpoints invités et des fonctions sensibles identifiées. Une revue
  exhaustive un-à-un des 841 endpoints n'a pas été faite.
- **Analyse dynamique non réalisée** : aucun site n'était en fonctionnement ;
  aucun scan dynamique intrusif n'a été tenté (et ne l'aurait pas été sans
  autorisation explicite).
- **Modules externes hors dépôt** (frappe, payments, HRMS, HRMS utils) non
  audités.

## 6. Niveau de confiance

Élevé sur les constats corrigés (chaque vulnérabilité confirmée par lecture du
code et correctif validé par build/installation réels). Moyen sur l'absence
d'autres vulnérabilités dans le périmètre non couvert par les limites ci-dessus
(aucun audit ponctuel ne peut garantir l'absence totale de vulnérabilité).

## 7. Résultat

Voir `vulnerability-register.md` (registre complet), `remediation-log.md`
(correctifs) et `regression-report.md` (validation).
