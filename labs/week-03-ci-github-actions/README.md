# TP 3 — Intégration Continue avec GitHub Actions

Ce document consigne les étapes de réalisation, les preuves d'exécution et les exercices du TP 3 du cours "DevOps et IA" (Master IS2IA - ESISA).

---

## 1. Architecture du Pipeline CI

Le pipeline est structuré en deux fichiers afin d'exploiter la modularité et la réutilisabilité des GitHub Actions :
1. `.github/workflows/lint-test.yml` : Workflow réutilisable (`workflow_call`) prenant en paramètre `python-version`. Il réalise le checkout, la configuration de Python avec mise en cache pip, l'installation des dépendances, le linting (`ruff check .`), l'exécution des tests avec génération de rapport JUnit (`pytest -q --junitxml=report.xml`) et la publication de l'artefact (`test-report-${{ inputs.python-version }}`).
2. `.github/workflows/ci.yml` : Workflow principal déclenché sur `pull_request` et `push` vers `main`. Il orchestre une matrice de versions Python (`3.11` et `3.12`) en appelant le workflow réutilisable.

---

## 2. Validation Locale avec `actionlint`

Exécution locale d'`actionlint` sur l'ensemble des fichiers de workflow :
```bash
actionlint -verbose .github/workflows/ci.yml .github/workflows/lint-test.yml
```

**Sortie réelle obtenue :**
```text
verbose: Linting 2 files
verbose: Linting .github\workflows\lint-test.yml
verbose: Using project at C:\Users\Oussa\OneDrive\Desktop\capstone-starter
verbose: Linting .github\workflows\ci.yml
verbose: Using project at C:\Users\Oussa\OneDrive\Desktop\capstone-starter
verbose: Found 0 parse errors in 0 ms for .github\workflows\lint-test.yml
verbose: Found 0 parse errors in 0 ms for .github\workflows\ci.yml
verbose: Found total 0 errors in 37 ms for .github\workflows\ci.yml
verbose: Found total 0 errors in 39 ms for .github\workflows\lint-test.yml
verbose: Found 0 errors in 2 files
```

---

## 3. Preuves d'Exécution : Pull Request, Run Vert et Run Rouge

- **Pull Request d'intégration CI** : [#6 (ci: matrix tests on python 3.11/3.12 with reusable workflow)](https://github.com/Malkiouss/capstone-starter/pull/6).
- **Run Vert Initial (PR #6)** : [Run #44573334408](https://github.com/Malkiouss/capstone-starter/actions/runs/44573334408) — Statut : `success` (3.11 en 13s, 3.12 en 30s). Artefacts générés : `test-report-3.11` et `test-report-3.12`.
- **Run Rouge (PR #7 scratch)** : [Run #44573636273](https://github.com/Malkiouss/capstone-starter/actions/runs/44573636273) — Statut : `failure`.
- **Run Vert après correction (PR #7 scratch)** : [Run #44573831059](https://github.com/Malkiouss/capstone-starter/actions/runs/44573831059) — Statut : `success` (3.12 en 15s, 3.11 en 17s).

### Extrait du log en échec (Run Rouge #44573636273)
```text
=================================== FAILURES ===================================
_________________________________ test_health __________________________________

    def test_health():
        r = client.get("/health")
>       assert r.status_code == 418
E       assert 200 == 418
E        +  where 200 = <Response [200 OK]>.status_code

tests/test_main.py:10: AssertionError
=========================== short test summary info ============================
FAILED tests/test_main.py::test_health - assert 200 == 418
1 failed, 1 passed, 1 warning in 0.45s
Error: Process completed with exit code 1.
```

[CAPTURE À AJOUTER : Graphique d'exécution GitHub Actions montrant le statut vert du workflow CI]

---

## 4. Statuts de Vérification Requis (Branch Protection)

### 4.1 Noms exacts des vérifications (Matrix-expanded)
Les noms exacts des jobs générés par GitHub Actions sont :
- `test (3.11) / lint-test`
- `test (3.12) / lint-test`

### 4.2 Commande `gh api` pour appliquer la protection stricte
```bash
gh api -X PUT repos/Malkiouss/capstone-starter/branches/main/protection \
  --input - << 'EOF'
{
  "required_status_checks": {
    "strict": true,
    "contexts": [
      "test (3.11) / lint-test",
      "test (3.12) / lint-test"
    ]
  },
  "required_pull_request_reviews": {
    "required_approving_review_count": 1,
    "dismiss_stale_reviews": false,
    "require_code_owner_reviews": false
  },
  "required_conversation_resolution": true,
  "enforce_admins": false,
  "restrictions": null
}
EOF
```

---

## 5. Mesure de l'Impact de la Mise en Cache (`cache: pip`)

| Métrique | Premier run (Run #44573334408) | Run ultérieur (Run #44573831059) | Différence |
|---|---|---|---|
| **Étape "Set up Python 3.12"** | 15s (cache miss, téléchargement toolcache) | 2s (cache hit) | **-13s (gain de 87%)** |
| **Durée totale du job Python 3.12** | 30s | 15s | **-15s (gain de 50%)** |
| **Étape "Set up Python 3.11"** | < 1s | 1s | Stable |
| **Durée totale du job Python 3.11** | 13s | 17s | +4s (variabilité runner) |

> **Note méthodologique :** Bien que le gain sur Python 3.12 soit net (-13s sur l'étape de setup grâce à la réutilisation des packages pré-téléchargés), l'échantillon mesuré (2 exécutions) est trop restreint pour constituer une preuve statistique définitive, les variations de latence réseau et de charge des runners hébergés par GitHub (Azure) introduisant une marge de bruit de ±3 secondes.

---

## 6. Exercice IA : Diagnostic d'Anomalie CI

### 6.1 Entrée fournie à l'IA
Le log d'échec du Run #44573636273 :
`FAILED tests/test_main.py::test_health - assert 200 == 418`

### 6.2 Diagnostic produit par l'IA
> **Analyse automatique :**  
> L'assertion sur la ligne 10 de `tests/test_main.py` a échoué. Le client HTTP a reçu le code de statut HTTP `200` (`OK`), alors que le test s'attendait explicitement au code HTTP `418` (`I'm a teapot`).  
> **Cause racine :** Le test a été modifié pour attendre un code non standard, ou l'endpoint a changé de comportement.  
> **Recommandation :** Rétablir l'assertion `assert r.status_code == 200` si l'endpoint est censé être nominal, ou implémenter la route spécifique correspondante dans FastAPI.

### 6.3 Évaluation humaine du diagnostic
- **Exactitude :** 100% correct. L'IA a immédiatement isolé la divergence entre le code retourné par FastAPI (`200`) et la valeur factice injectée (`418`).
- **Utilité en contexte réel :** Très élevée, car elle évite au développeur de chercher une erreur d'infrastructure CI ou de dépendance en pointant directement vers la ligne du code de test incriminée.

---

## 7. Pièges fréquents / Erreurs et Solutions

| Erreur / Symptôme | Cause racine | Solution |
|---|---|---|
| `The filename, directory name, or volume label syntax is incorrect` avec `actionlint` sous Windows | Les shells Windows (PowerShell/CMD) ne résolvent pas automatiquement les wildcards (`*.yml`) passés à des exécutables Go externes. | Passer la liste des fichiers résolue explicitement : `actionlint (Get-Item .github/workflows/*.yml \| ForEach-Object { $_.FullName })`. |
| Échec de téléchargement des artefacts (`upload-artifact`) après un test rouge | L'étape d'upload par défaut ne s'exécute que si les étapes précédentes ont réussi (`success()`). | Ajouter la directive conditionnelle `if: always()` sur l'étape `upload-artifact` pour garantir la publication des rapports de test en toute circonstance. |
| Nom de status check introuvable dans la protection de branche | Les workflows matriciels génèrent des contextes dynamiques incluant les arguments de la matrice et les jobs appelés. | Inspecter le nom réel via `gh pr checks <PR_NUMBER>` avant de le déclarer dans `contexts`. |
| Actionlint signale des erreurs de syntaxe sur `workflow_call` | Syntaxe obsolète ou confusion entre `workflow_dispatch` et `workflow_call`. | Définir clairement les `inputs` sous `workflow_call` avec types explicites (`type: string`). |
