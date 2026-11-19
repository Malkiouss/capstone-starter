# TP 2 — Team Git Workflow & Bonnes Pratiques GitHub

Ce document consigne les étapes de réalisation, les preuves d'exécution et les exercices du TP 2 du cours "DevOps et IA" (Master IS2IA - ESISA).

---

## 1. Protection de la branche `main`

### 1.1 État initial
Interrogation de la protection de branche via GitHub CLI :
```bash
gh api repos/Malkiouss/capstone-starter/branches/main/protection
```
**Résultat initial** : HTTP 404 (`Branch not protected`). La branche `main` ne possédait aucune règle de protection.

### 1.2 Configuration requise appliquée
Règles appliquées :
- Exiger une Pull Request avant de fusionner (`required_pull_request_reviews`).
- 1 approbation requise (`required_approving_review_count: 1`).
- Résolution obligatoire des conversations (`required_conversation_resolution: true`).

```bash
gh api -X PUT repos/Malkiouss/capstone-starter/branches/main/protection \
  -f "required_pull_request_reviews[required_approving_review_count]=1" \
  -F "required_pull_request_reviews[dismiss_stale_reviews]=false" \
  -F "required_pull_request_reviews[require_code_owner_reviews]=false" \
  -F "required_conversation_resolution=true" \
  -F "enforce_admins=false" \
  -f "required_status_checks=null" \
  -f "restrictions=null"
```

**Preuve (retour gh api)** :
```json
{
  "required_pull_request_reviews": {
    "required_approving_review_count": 1,
    "dismiss_stale_reviews": false,
    "require_code_owner_reviews": false
  },
  "required_conversation_resolution": {
    "enabled": true
  },
  "enforce_admins": {
    "enabled": false
  }
}
```

[CAPTURE À AJOUTER : Interface GitHub Settings > Branches montrant les règles appliquées sur main]

---

## 2. Templates GitHub et CODEOWNERS

- **Fichiers créés/améliorés** : `.github/pull_request_template.md` et `.github/CODEOWNERS`.
- **Branche** : `chore/github-templates`.
- **Pull Request associée** : [#2 (chore(github): add PR template and CODEOWNERS)](https://github.com/Malkiouss/capstone-starter/pull/2).
- **Statut** : Fusionnée (`MERGED`).

---

## 3. Démonstration de Conflit et Résolution par Rebase

### 3.1 Création des branches divergentes
Deux branches ont été créées à partir du même commit de `main` et ont modifié la même ligne dans `app/main.py` :
- `feat/a` : renvoie `{"status": "ok", "service": "capstone-api"}` -> PR [#3](https://github.com/Malkiouss/capstone-starter/pull/3) fusionnée en premier.
- `feat/b` : renvoie `{"status": "ok", "version": "1.0.0"}` -> PR [#4](https://github.com/Malkiouss/capstone-starter/pull/4).

### 3.2 Constat du conflit sur GitHub
GitHub reporte l'état suivant sur la PR #4 :
```json
{"mergeStateStatus":"DIRTY","mergeable":"CONFLICTING"}
```

### 3.3 Rapport de résolution de conflit en 5 lignes
1. Récupération des nouveautés et amorçage du rebase : `git fetch origin && git rebase origin/main`.
2. Détection du marqueur de conflit sur `app/main.py` entre `HEAD` (`service`) et le commit de `feat/b` (`version`).
3. Fusion manuelle des deux enrichissements : `{"status": "ok", "service": "capstone-api", "version": "1.0.0"}`.
4. Validation locale des tests (`pytest -q`) et du linter (`ruff check .`), puis `git add app/main.py && git rebase --continue`.
5. Publication de l'historique réaligné de manière sécurisée avec `git push --force-with-lease origin feat/b`.

### 3.4 Historique Git avant et après rebase

**Avant rebase (`git log --oneline -5`) :**
```text
f60c30e feat(api): add version field to health endpoint
69db37c Merge pull request #2 from Malkiouss/chore/github-templates
6498f7a chore(github): update PR template and CODEOWNERS
b9874c3 Merge pull request #1 from Malkiouss/feature/new-feature
aaef91e fix(lint): resolve ruff formatting and linting errors
```

**Après rebase (`git log --oneline -5`) :**
```text
5eee611 feat(api): add version field to health endpoint
cfca154 Merge pull request #3 from Malkiouss/feat/a
746f90d feat(api): add service field to health endpoint
69db37c Merge pull request #2 from Malkiouss/chore/github-templates
6498f7a chore(github): update PR template and CODEOWNERS
```

Après rebase, la PR #4 devient `{"mergeStateStatus":"BLOCKED","mergeable":"MERGEABLE"}` et est fusionnée avec succès.

---

## 4. Grille de Revue Croisée (Cross-Review Checklist)

### 4.1 Grille d'évaluation
| Critère | Questions clés |
|---|---|
| **Tests** | Les tests existants passent-ils (`pytest`) ? De nouveaux tests couvrent-ils le comportement ajouté ? |
| **Lisibilité** | Le code respecte-t-il PEP 8 / Ruff ? Les noms de variables/fonctions sont-ils explicites ? |
| **Sécurité** | Aucun secret / clé API / mot de passe en dur ? Gestion appropriée des erreurs sans fuite d'informations ? |
| **Architecture** | Le changement est-il atomique ? Respecte-t-il les responsabilités du module ? |

### 4.2 Application à la PR #4 (`feat/b`)
- **Tests** : Validés (`2 passed`).
- **Lisibilité** : Conforme aux standards Ruff, commit Conventional Commits clair.
- **Sécurité** : Aucun secret introduit, endpoint non vulnérable.
- **Nécessité d'un 2ème relecteur** : Non requise pour un patch mineur de documentation/payload d'API, mais obligatoire si modification de `k8s/` ou de `infra/` selon `CODEOWNERS`.

---

## 5. Exercices Pratiques

### 5.1 Exercice `git revert` vs `git reset`
- **Exécution** :
  ```bash
  git checkout -b scratch/revert-exercise main
  git revert 5eee611 --no-edit
  ```
  *Résultat* : commit `c84750d` créé annulant les changements de `5eee611`.
- **Pourquoi `revert` plutôt que `reset` sur une branche partagée ?**  
  `git reset` réécrit l'historique public en supprimant des commits. Cela désynchronise les copies locales des collaborateurs et force des résolutions de divergence complexes. À l'inverse, `git revert` avance l'historique en créant un nouveau commit qui inverse exactement l'effet du commit ciblé, préservant l'intégrité et la traçabilité de la branche partagée.

### 5.2 Exercice `git bisect`
1. Création d'une branche `scratch/bisect-exercise` et introduction intentionnelle d'une anomalie dans `app/main.py` au commit `bdada74`.
2. Lancement automatisé :
   ```bash
   git bisect start
   git bisect bad HEAD
   git bisect good main
   git bisect run .venv/Scripts/pytest.exe -q
   ```
3. **Sortie réelle obtenue** :
   ```text
   Bisecting: 0 revisions left to test after this (roughly 0 steps)
   [9b6d9757e3e9dab9c036ffef4686c73c9cb6e301] feat: add step 1 comment
   running '.venv\Scripts\pytest.exe' '-q'
   .. [100%]
   2 passed, 1 warning in 0.75s
   bdada743ecd5ec36da856e01e463a610e4b034ab is the first bad commit
   commit bdada743ecd5ec36da856e01e463a610e4b034ab
   Author: Malki Oussama <oussama.malkiou@gmail.com>
   Date: Thu Nov 19 17:00:55 2026 +0000

       feat: change prediction threshold
   ```
4. Clôture avec `git bisect reset` et suppression de la branche scratch.

### 5.3 Exercice IA : Rédaction de Commit et Description de PR

**Brouillon initial proposé par une IA naive :**
```text
Commit : update code
PR Body : I fixed the health check endpoint and added new stuff.
```

**Version corrigée et professionnelle :**
```text
Commit : feat(api): include version and service metadata in health check response

PR Body :
## Quoi / Pourquoi
Enrichissement du point de terminaison `/health` afin de renvoyer le nom du service (`capstone-api`)
ainsi que la version courante (`1.0.0`). Cela facilite l'observabilité et le diagnostic lors des déploiements.

## Comment tester
1. Démarrer l'API ou lancer les tests unitaires : `pytest -q`
2. Vérifier la réponse de `curl http://localhost:8000/health` qui doit contenir `status`, `service` et `version`.

## Checklist
- [x] Tests unitaires mis à jour et validés
- [x] Linter ruff conforme
- [x] Aucun secret ou credential inclus
```

**Justification des corrections :**
1. **Format Conventional Commits** : `feat(api): ...` permet le versionnement sémantique automatique et identifie immédiatement le périmètre technique (`api`).
2. **Précision contextuelle** : Remplacement de termes vagues ("update code", "fixed stuff") par l'intention réelle du changement.
3. **Reproductibilité** : Ajout d'une procédure claire de test pour le relecteur.
4. **Garantie qualité/sécurité** : Présence de la checklist explicite pour prévenir toute régression ou fuite de secrets.

---

## 6. Pièges fréquents / Erreurs et Solutions

| Erreur / Symptôme | Cause racine | Solution |
|---|---|---|
| `git push --force` rejeté sur `main` | Protection de branche active interdisant la réécriture d'historique. | Ne jamais forcer sur `main`. Utiliser des PRs et `git push --force-with-lease` uniquement sur sa branche de travail. |
| `mergeStateStatus: DIRTY` sur GitHub | Modifications concurrentes sur les mêmes lignes de code. | Rebaser localement sur `origin/main` (`git fetch origin && git rebase origin/main`), résoudre les conflits, puis repousser. |
| Fichier `.env` présent dans `git status` | Fichier de secrets non ignoré ou préalablement indexé. | Vérifier `.gitignore` et exécuter `git rm --cached .env` si déjà suivi. Ne jamais commiter de secret. |
| Rebase bloqué dans un éditeur interactif sous Windows | `core.editor` ou variable `GIT_EDITOR` non configurée pour un terminal non-interactif. | Utiliser `git commit --no-edit` ou configurer `git config core.editor true`. |
