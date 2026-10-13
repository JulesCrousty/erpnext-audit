# Audit de sécurité ERPNext — Documentation

Ce dossier contient la documentation complète de l'audit de sécurité réalisé sur
`frappe/erpnext` (branche `develop`), conformément à une mission d'ingénierie
sécurité (AppSec / DevSecOps).

## Périmètre

- Dépôt : `frappe/erpnext`, clone local `/Users/julfav/Desktop/ZAZA/erpnext`
- Branche de référence : `develop`
- Commit de référence : `08550102c1d1830c843abd1b36b6b647be3a16b8`
- Branche de sécurité : `security/comprehensive-hardening-20261013`
- Date de l'audit : 2026-10-13

## Documents

| Fichier | Contenu |
|---|---|
| `architecture-security-map.md` | Cartographie technique : points d'entrée, surfaces d'attaque, frontières de confiance |
| `baseline.md` | Comportement de référence avant modification |
| `audit-report.md` | Rapport d'audit : méthodologie, outils, constats, limites |
| `vulnerability-register.md` | Registre des vulnérabilités identifiées et classées |
| `remediation-log.md` | Journal des correctifs : fichiers, comportements, tests, justification |
| `regression-report.md` | Comparaison avant/après, régressions, zones non testées |
| `dependency-audit.md` | Audit des dépendances Python et JavaScript, CVE, versions |
| `security-recommendations.md` | Recommandations non implémentées nécessitant décision humaine |
| `deployment-considerations.md` | Impacts déploiement, rollback, actions manuelles |

## Résumé

- **841 endpoints whitelistés** passés au crible par motifs ; 7 accessibles aux
  invités revus individuellement.
- **Aucune injection SQL exploitable** identifiée (pattern Frappe paramétré ;
  interpolation structurelle uniquement).
- **Aucun secret** exposé dans le code, les configurations ou les workflows.
- **3 requêtes HTTP sortantes sans timeout** corrigées (dont 1 endpoint accessible
  aux invités — vecteur de déni de service).
- **21 CVE npm** dans `banking/yarn.lock` : 20 corrigées (4 packages), 1 risque
  résiduel documenté (`ws`, dépendance de développement uniquement).
- **0 CVE Python** sur les 12 dépendances directes.
- Validation non-régression : build de production du module `banking` réussi
  avant et après correctifs ; suite de tests serveur non exécutable sur cette
  machine (bench Frappe requis) — voir `baseline.md`.

Aucun changement de comportement fonctionnel pour les usages légitimes, hormi
les comportements explicitement documentés dans `remediation-log.md`.
