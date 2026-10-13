# Audit des dépendances et supply chain

Date : 2026-10-13. Source : OSV.dev (API querybatch sur les versions exactes).

## Dépendances Python (pyproject.toml)

12 dépendances directes vérifiées contre OSV : **0 vulnérabilité connue**.

| Paquet | Version déclarée | Vulnérabilités |
|---|---|---|
| Unidecode | ~=1.4.0 | 0 |
| barcodenumber | ~=0.5.0 | 0 |
| rapidfuzz | ~=3.14.3 | 0 |
| holidays | ~=0.87 | 0 |
| googlemaps | ~=4.10.0 | 0 |
| plaid-python | ~=7.2.1 | 0 |
| python-youtube | ~=0.9.9 | 0 |
| pypng | ~=0.20220715.0 | 0 |
| mt-940 | >=4.26.0 | 0 |
| pdfplumber | >=0.11.0 | 0 |

Le framework (`frappe >= 17.0.0-dev`) et ses propres dépendances sont hors de ce
dépôt — à auditer côté `frappe/frappe`.

## Dépendances npm (`banking/yarn.lock`, 424 packages)

21 vulnérabilités connues affectaient les versions verrouillées ; **20 corrigées**
par bump de 4 packages, **1 risque résiduel documenté** (`ws`).

### Corrigées

| Package | Avant | Après | CVE | Contexte d'usage |
|---|---|---|---|---|
| axios | 1.18.1 | 1.20.0 | GHSA-3pq3-5fj3-cg6v (HAUTE), GHSA-542g-h47m-68v8 (HAUTE), GHSA-c29m-xwm3-cm6r (HAUTE), GHSA-m8m8-qj5v-23w3 (HAUTE), GHSA-mghh-pgcx-3jjj (HAUTE), GHSA-r4gj-5m52-g5wh (HAUTE), GHSA-9fr6-4gfg-395g (MOYENNE), GHSA-4hqw-qxg8-jxx2 (MOYENNE), GHSA-j8rh-479h-cp32 (MOYENNE), GHSA-vh66-26gq-q6x8 (MOYENNE), GHSA-44g4-m2mj-wpvx (MOYENNE), GHSA-x97p-jq2g-jp4f (MOYENNE) | runtime (navigateur, via `frappe-react-sdk`) |
| brace-expansion | 5.0.7 / 5.0.9 | 5.0.12 | GHSA-6j4f-fj2g-mc7p (HAUTE), GHSA-rgw5-rvv9-x895 (HAUTE), GHSA-qhr7-859c-m2p7 (HAUTE), GHSA-mh99-v99m-4gvg (HAUTE), GHSA-q2hr-2g5m-vwhr (MOYENNE) | build/dev (outils de build, non embarqué) |
| socket.io-parser | 4.2.6 | 4.2.7 | GHSA-2m8v-j782-fhvr (HAUTE) | runtime (navigateur, via `socket.io-client` 4.7.1) |
| source-map-js | 1.2.1 | 1.2.2 | GHSA-68fv-2mgg-jv7q (HAUTE) | build/dev |

### Non corrigée (risque résiduel)

| Package | Version | CVE | Raison |
|---|---|---|---|
| ws | 8.17.1 | GHSA-58qx-3vcg-4xpx (MOYENNE), GHSA-96hv-2xvq-fx4p (HAUTE) | Correctif 8.21.0 hors range `~8.17.1` d'`engine.io-client` ; usage limité au serveur de développement (non embarqué dans le bundle de production). À corriger via une montée de version de l'app dépendante. |

## Supply chain

- **Scripts d'installation** : `package.json` racine exécute `postinstall:
  "cd banking && yarn install"` — standard pour un dépôt Frappe, exécuté à
  l'installation des assets, pas par des utilisateurs finaux.
- **Intégrité des packages** : tous les tarballs vérifiés par les hashes
  `integrity` du lockfile lors de l'installation gelée (`--frozen-lockfile`).
- **Épinglage des actions CI** : partiel — voir ERPSEC-009/010
  (`security-recommendations.md`).
- **Dépendances non utilisées** : aucune constatée ; `package.json` racine
  ne déclare qu'`onscan.js` (utilisé par le scan de codes-barres POS).
