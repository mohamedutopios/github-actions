# TP GitHub Actions — Pipeline CI/CD complet · CORRIGÉ

**Formation GitHub Actions · Utopios × Mohamed**

---

## Solution complète — `.github/workflows/pipeline.yml`

Le fichier ci-dessous est la solution finale intégrant les trois étapes du TP.
Les annotations expliquent chaque choix technique.

---

## Fichier `pipeline.yml`

```yaml
# =============================================================
# TP CORRIGÉ — Pipeline CI/CD complet · utopios-app
# Concepts : déclencheurs · jobs & steps · needs
# =============================================================

name: "Pipeline CI/CD · utopios-app"

# ─────────────────────────────────────────────────────────────
# ÉTAPE 1 — DÉCLENCHEURS
#
# Trois déclencheurs configurés :
#   1. push sur main ou develop uniquement (pas sur feature/*)
#   2. pull_request ciblant main
#   3. workflow_dispatch avec un input "environment"
# ─────────────────────────────────────────────────────────────
on:
  push:
    branches:
      - main
      - develop

  pull_request:
    branches:
      - main

  workflow_dispatch:
    inputs:
      environment:
        description: "Environnement cible"
        required: true
        type: choice
        options:
          - staging
          - production

# ─────────────────────────────────────────────────────────────
# ÉTAPES 2 & 3 — JOBS, STEPS ET ORCHESTRATION AVEC NEEDS
#
# Graphe d'exécution final :
#
#   lint ──┐
#          ├──→ build ──→ deploy (sur main uniquement)
#   tests ─┘
# ─────────────────────────────────────────────────────────────
jobs:

  # ───────────────────────────────────────────────────────────
  # JOB 1 : lint
  # Tourne en parallèle avec "tests".
  # Pas de "needs" : démarre immédiatement.
  # ───────────────────────────────────────────────────────────
  lint:
    name: "🎨 Lint — Vérification du style"
    runs-on: ubuntu-latest

    steps:
      - name: "Analyser le style du code"
        run: |
          echo "Analyse du style en cours..."
          sleep 2

      - name: "Confirmer le résultat"
        run: echo "✅ Aucune erreur de style détectée"

  # ───────────────────────────────────────────────────────────
  # JOB 2 : tests
  # Tourne en parallèle avec "lint".
  # Le step de rapport utilise "if: always()" pour s'exécuter
  # même si les tests échouent — essentiel pour les rapports.
  # ───────────────────────────────────────────────────────────
  tests:
    name: "🧪 Tests — Tests automatisés"
    runs-on: ubuntu-latest

    steps:
      - name: "Préparer les tests"
        run: echo "Lancement des tests..."

      - name: "Exécuter les tests"
        run: |
          sleep 4
          echo "✅ 21 tests passés, 0 échoué"

      - name: "Générer le rapport de tests"
        if: always()   # ← s'exécute même si un step précédent échoue
        run: echo "Rapport de tests généré"

  # ───────────────────────────────────────────────────────────
  # JOB 3 : build
  # Attend que "lint" ET "tests" soient terminés avec succès.
  # Si l'un des deux échoue, ce job est annulé.
  # ───────────────────────────────────────────────────────────
  build:
    name: "🔨 Build — Compilation"
    runs-on: ubuntu-latest
    needs: [lint, tests]   # ← attend les deux jobs en parallèle

    steps:
      - name: "Compiler l'application"
        run: |
          echo "Compilation de utopios-app..."
          sleep 3

      - name: "Confirmer le build"
        run: echo "✅ Build réussi — artefact prêt"

  # ───────────────────────────────────────────────────────────
  # JOB 4 : deploy
  # Attend "build". Ne s'exécute QUE sur la branche main.
  # Sur "develop" ou PR, ce job est ignoré (skipped).
  #
  # L'input "environment" est accessible via ${{ inputs.environment }}.
  # En dehors d'un dispatch manuel, inputs.environment est vide :
  # on utilise || 'staging' comme valeur par défaut.
  # ───────────────────────────────────────────────────────────
  deploy:
    name: "🚀 Deploy — Déploiement"
    runs-on: ubuntu-latest
    needs: build
    if: github.ref_name == 'main'   # ← ignoré si pas sur main

    steps:
      - name: "Déployer l'application"
        run: |
          ENV="${{ inputs.environment }}"
          echo "Déploiement sur ${ENV:-staging}..."

      - name: "Confirmer le déploiement"
        run: echo "✅ utopios-app est en ligne !"
```

---

## Explication des choix techniques

### Étape 1 — Déclencheurs

**Pourquoi filtrer les branches dans `push` ?**
Sans filtre, le workflow se déclenche sur toutes les branches, y compris `feature/*`. Cela consomme des minutes CI inutilement. On surveille uniquement `main` (production) et `develop` (intégration).

**Pourquoi `pull_request: branches: [main]` séparément ?**
Le filtre `push.branches` ne couvre pas les PR. Une PR ouverte depuis `feature/x` vers `main` est un événement `pull_request`, pas un `push`. Les deux doivent être déclarés explicitement.

**Pourquoi `type: choice` pour l'input ?**
Il force l'utilisateur à choisir parmi des valeurs prédéfinies, évitant les erreurs de frappe (`Staging`, `PRODUCTION`, `prod`…).

---

### Étape 2 — Jobs et steps

**Pourquoi `if: always()` sur le step rapport ?**
Sans cette condition, si un test échoue (`exit 1`), GitHub annule les steps suivants du même job. Le rapport ne serait jamais généré — alors qu'il est précisément utile en cas d'échec pour diagnostiquer l'erreur.

**Pourquoi `sleep` dans les steps ?**
Pour simuler un temps de traitement réel (compilation, tests…) et rendre l'effet du parallélisme visible dans l'interface : on voit clairement que `lint` et `tests` avancent en même temps.

---

### Étape 3 — Orchestration avec `needs`

**Pourquoi `needs: [lint, tests]` sur `build` et pas `needs: lint` + `needs: tests` ?**
`needs` n'accepte qu'une seule déclaration par job. Pour attendre plusieurs jobs, on passe une liste : `[lint, tests]`. GitHub attend que **tous** soient réussis.

**Pourquoi `if: github.ref_name == 'main'` sur `deploy` et pas sur un step ?**
Si la condition est sur le **job**, le job entier est marqué `skipped` (ignoré). Les jobs suivants dans la chaîne `needs` ne sont pas bloqués. Si la condition était sur un step seulement, le job démarrerait quand même — ce qui serait trompeur dans les logs.

**Que se passe-t-il quand `tests` échoue ?**

```
lint   → ✅ succès
tests  → ❌ échec
         └─ step rapport → ✅ (grâce à if: always())
build  → ⊘ annulé (needs: tests non satisfait)
deploy → ⊘ annulé (needs: build non satisfait)
```

**Que se passe-t-il sur `develop` ?**

```
lint   → ✅
tests  → ✅
build  → ✅
deploy → ⏭ skipped (if: github.ref_name == 'main' → false)
```

---

## Vérifications attendues

| Critère | Explication |
|---|---|
| Pas de run sur `feature/*` | `push.branches` liste explicite sans wildcard `feature/*` |
| Run sur `main`, `develop`, PR→main, dispatch | Les 3 blocs `on:` couvrent exactement ces cas |
| `environment` visible dans les logs | `${{ inputs.environment }}` injecté dans le step deploy |
| Rapport généré même en cas d'échec | `if: always()` sur le step rapport de `tests` |
| `lint` et `tests` démarrent en même temps | Absence de `needs` sur ces deux jobs |
| `build` attend les deux | `needs: [lint, tests]` |
| `deploy` ignoré sur `develop` | `if: github.ref_name == 'main'` au niveau du job |
| Échec de `tests` → `build` et `deploy` annulés | Propagation automatique des `needs` non satisfaits |

---

*© 2025 Mohamed × Utopios — Formation GitHub Actions*
