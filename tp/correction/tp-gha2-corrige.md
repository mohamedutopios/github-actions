# TP GitHub Actions — Actions, Variables & Secrets · CORRIGÉ

**Formation GitHub Actions · Utopios × Mohamed**

---

## Solution complète — `.github/workflows/pipeline-pro.yml`

Le fichier solution est fourni séparément (`pipeline-pro.yml`).
Ce document explique chaque choix technique étape par étape.

---

## Étape 1 — Les actions

### Pourquoi `actions/checkout@v4` en premier ?

Le runner GitHub Actions démarre sur une machine vierge. Le dépôt n'y est pas disponible par défaut. Sans `checkout`, les fichiers `index.js` et `index.test.js` n'existent pas — les steps suivants échouent immédiatement.

```yaml
- name: "📥 Cloner le dépôt"
  uses: actions/checkout@v4
```

La différence entre `uses:` et `run:` est fondamentale :

| | `uses:` | `run:` |
|---|---|---|
| Exécute | Une action préfabriquée | Une commande shell |
| Vient de | GitHub Marketplace | Vous |
| Exemple | `actions/checkout@v4` | `npm install` |

### Pourquoi `cache: "npm"` dans setup-node ?

Sans cache, npm retélécharge tous les packages à chaque run. Avec `cache: "npm"`, GitHub stocke le dossier `~/.npm` entre les runs et le restaure si le `package.json` n'a pas changé. Sur un vrai projet, cela économise 1 à 3 minutes par run.

```yaml
- uses: actions/setup-node@v4
  with:
    node-version: "20"
    cache: "npm"
```

### Comment fonctionne `upload-artifact` ?

L'artifact est un dossier zippé sauvegardé par GitHub après le run. Il reste téléchargeable pendant `retention-days` jours. C'est le seul moyen de **faire sortir des fichiers** d'un job — sinon tout est perdu quand la machine s'arrête.

```yaml
- uses: actions/upload-artifact@v4
  with:
    name: rapport-tests        # nom visible dans l'interface
    path: test-results/        # dossier à zipper
    retention-days: 7
```

---

## Étape 2 — Les variables

### Les trois niveaux de portée

```
Workflow env:          → APP_NAME, NODE_ENV
  └── Job env:         → CI, RAPPORT_DIR
        └── Step env:  → variables locales au step
```

Une variable définie au niveau workflow est visible partout. Une variable définie au niveau step disparaît dès que le step se termine.

```yaml
# Niveau workflow — visible dans TOUS les jobs
env:
  APP_NAME: "utopios-app"
  NODE_ENV: "test"

jobs:
  ci:
    # Niveau job — visible dans tous les steps du job ci
    env:
      CI: "true"
      RAPPORT_DIR: "test-results"
```

### Comment passer une valeur d'un step à un autre ?

Deux steps ne partagent pas de mémoire. Le seul canal officiel est `$GITHUB_OUTPUT`. Le step émetteur écrit dans ce fichier, et les steps suivants lisent via `steps.<id>.outputs.<cle>`.

```yaml
# Step émetteur — doit avoir un "id"
- name: "📋 Afficher les variables"
  id: infos
  run: |
    VERSION="1.0.${{ github.run_number }}"
    echo "version=$VERSION" >> $GITHUB_OUTPUT

# Step récepteur — lit l'output du step "infos"
- name: "📄 Créer le rapport"
  run: |
    echo "Version : ${{ steps.infos.outputs.version }}" >> rapport.txt
```

**Erreur courante :** oublier l'`id:` sur le step émetteur. Sans `id`, il est impossible de référencer ses outputs.

### Pourquoi utiliser `${{ env.RAPPORT_DIR }}` dans `upload-artifact` ?

Dans la section `with:` d'une action, le contexte shell (`$RAPPORT_DIR`) n'est pas disponible. Il faut utiliser la syntaxe d'expression GitHub : `${{ env.RAPPORT_DIR }}`.

```yaml
# ✅ Correct dans with:
path: ${{ env.RAPPORT_DIR }}/

# ❌ Ne fonctionne pas dans with:
path: $RAPPORT_DIR/
```

---

## Étape 3 — Les secrets

### Pourquoi passer les secrets par `env:` et non directement dans `run:` ?

```yaml
# ✅ BONNE PRATIQUE
- name: Utiliser le token
  env:
    MON_TOKEN: ${{ secrets.DEPLOY_TOKEN }}
  run: |
    curl -H "Authorization: Bearer $MON_TOKEN" https://api.example.com

# ⚠️ À ÉVITER — le secret apparaît dans le YAML parsé
- name: Utiliser le token
  run: |
    curl -H "Authorization: Bearer ${{ secrets.DEPLOY_TOKEN }}" https://api.example.com
```

La deuxième forme fonctionne, mais elle intègre la valeur du secret directement dans la commande shell avant l'exécution. Cela peut exposer la valeur dans certains contextes de débogage. La première forme garde le secret dans une variable d'environnement système, plus sûre.

### Comment vérifier qu'un secret est configuré sans l'afficher ?

```bash
# Longueur → confirme que le secret n'est pas vide
echo "Longueur : ${#DEPLOY_TOKEN}"

# Premiers caractères → permet d'identifier le secret sans l'exposer
echo "Début : ${DEPLOY_TOKEN:0:3}***"

# Test de présence
if [ -z "$DEPLOY_TOKEN" ]; then
  echo "❌ Secret manquant"
fi
```

### Que se passe-t-il si on affiche directement un secret ?

GitHub détecte automatiquement la valeur du secret dans les logs et la remplace par `***`. C'est une protection de dernier recours — elle ne doit pas être votre seule défense.

```bash
echo $DEPLOY_TOKEN
# → Affiché dans les logs : ***
```

---

## Schéma complet du workflow

```
push / workflow_dispatch
         │
         ▼
    ┌──────────────────────────────────────────────────┐
    │  Job ci                                          │
    │                                                  │
    │  1. checkout           (action)                  │
    │  2. ls -la             (vérification)            │
    │  3. setup-node         (action + cache)          │
    │  4. afficher versions  (vérification)            │
    │  5. npm install        (commande shell)          │
    │  6. afficher variables (env + GITHUB_OUTPUT)     │
    │  7. node index.test.js (commande shell)          │
    │  8. créer rapport.txt  (utilise output step 6)   │
    │  9. upload-artifact    (action)                  │
    │  10. vérifier secrets  (env: secrets)            │
    │  11. notifier          (env: secrets)            │
    └──────────────────────────────────────────────────┘
```

---

## Erreurs fréquentes et solutions

**`Error: Cannot find module './index'`**
→ Le step `checkout` est absent ou mal placé. Il doit être en premier.

**`cache: npm — Cache not found`**
→ Normal au premier run. Le cache se constitue après le premier `npm install`. Les runs suivants seront plus rapides.

**`steps.infos.outputs.version` est vide**
→ L'`id: infos` est absent du step qui écrit dans `$GITHUB_OUTPUT`, ou la syntaxe `echo "cle=valeur" >> $GITHUB_OUTPUT` est incorrecte (vérifiez l'absence d'espaces autour du `=`).

**Le secret apparaît comme vide dans le step**
→ Le secret n'est pas configuré dans `Settings → Secrets`. Un secret non défini produit une chaîne vide, pas une erreur.

**`path: $RAPPORT_DIR/` ne fonctionne pas dans `with:`**
→ Utilisez `${{ env.RAPPORT_DIR }}/` — la syntaxe shell n'est pas interprétée dans les champs `with:`.

---

## Critères de réussite

| Critère | Explication |
|---|---|
| `ls -la` montre `index.js` | `checkout` est bien en premier step |
| Node 20.x dans les logs | `setup-node` avec `node-version: "20"` |
| Tests passent | `index.test.js` ne lance pas d'erreur |
| Artifact téléchargeable | `upload-artifact` avec le bon `path:` |
| Variables visibles dans les logs | `env:` aux trois niveaux correctement définis |
| Version dans `rapport.txt` | `$GITHUB_OUTPUT` et `steps.infos.outputs.version` |
| Token jamais en clair | Secrets injectés uniquement via `env:` du step |
| GitHub masque le secret affiché | Comportement natif GitHub — observable avec `echo $DEPLOY_TOKEN` |

---

*© 2025 Mohamed × Utopios — Formation GitHub Actions*
