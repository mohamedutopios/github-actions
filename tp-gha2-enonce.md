# TP GitHub Actions — Actions, Variables & Secrets

**Formation GitHub Actions · Utopios × Mohamed**
. Un seul fichier `.yml` à construire

---

## Contexte

Vous rejoignez l'équipe DevOps de `utopios-app`, une application Node.js.
Le pipeline de base existe déjà (TP précédent). Votre mission aujourd'hui :
le rendre **professionnel** en y intégrant de vraies actions du Marketplace,
des variables d'environnement structurées, et des secrets sécurisés.

À la fin, votre pipeline devra :

- utiliser des **actions** pour configurer l'environnement et sauvegarder les résultats
- injecter des **variables** à différentes portées (workflow, job, step)
- utiliser des **secrets** pour les informations sensibles sans jamais les exposer

---

## Mise en place

Repartez du dépôt `utopios-app` créé au TP précédent, ou créez-en un nouveau.

Créez les fichiers suivants à la racine du dépôt :

**`package.json`**
```json
{
  "name": "utopios-app",
  "version": "1.0.0",
  "scripts": {
    "test": "node --experimental-vm-modules index.test.js"
  }
}
```

**`index.js`**
```js
function saluer(prenom) {
  return `Bonjour, ${prenom} !`;
}
module.exports = { saluer };
```

**`index.test.js`**
```js
const { saluer } = require('./index');
const assert = require('assert');

assert.strictEqual(saluer('Mohamed'), 'Bonjour, Mohamed !');
assert.strictEqual(saluer('Alice'), 'Bonjour, Alice !');
console.log('✅ Tous les tests passent');
```

Votre workflow se trouvera dans :

```
.github/workflows/pipeline-pro.yml
```

---

## Travail demandé

### Étape 1 — Les actions

Le workflow contient un seul job `ci` qui tourne sur `ubuntu-latest` et se déclenche sur `push` et `workflow_dispatch`.

Construisez la séquence de steps suivante dans ce job :

**Step 1 — Récupérer le code**
Utilisez l'action `actions/checkout@v4` pour cloner le dépôt sur le runner.
Vérifiez qu'elle fonctionne en affichant le contenu du dossier courant avec `ls -la`.

**Step 2 — Configurer Node.js**
Utilisez l'action `actions/setup-node@v4` avec la version `20` et le cache `npm`.
Affichez ensuite les versions de `node` et `npm` dans les logs.

**Step 3 — Installer les dépendances**
Exécutez `npm install` avec la commande shell `run:`.

**Step 4 — Lancer les tests**
Exécutez `node index.test.js` avec `run:`.

**Step 5 — Sauvegarder les logs de test**
Créez un fichier `test-results/rapport.txt` contenant le texte `Tests exécutés le` suivi de la date (`$(date)`).
Puis utilisez l'action `actions/upload-artifact@v4` pour sauvegarder le dossier `test-results/` sous le nom `rapport-tests`.

> **Vérification :** poussez et allez dans Actions. En bas de la page du run, une section **Artifacts** doit apparaître avec `rapport-tests` téléchargeable.

---

### Étape 2 — Les variables

Sans changer la structure du job, ajoutez des variables d'environnement à trois niveaux.

**Au niveau du workflow** (sous la clé `env:` racine, avant `jobs:`) :
- `APP_NAME` avec la valeur `utopios-app`
- `NODE_ENV` avec la valeur `test`

**Au niveau du job `ci`** (sous la clé `env:` du job) :
- `CI` avec la valeur `true`
- `RAPPORT_DIR` avec la valeur `test-results`

**Dans un step dédié**, ajoutez un step entre "Installer les dépendances" et "Lancer les tests" qui :
- affiche le nom de l'application (`APP_NAME`)
- affiche l'environnement (`NODE_ENV`)
- affiche si on est en mode CI (`CI`)
- affiche le dossier de rapports (`RAPPORT_DIR`)
- calcule et affiche un numéro de version dynamique sous la forme `1.0.` suivi du numéro de run GitHub (`github.run_number`), et écrit cette valeur dans `$GITHUB_OUTPUT` sous la clé `version`

Dans le step "Sauvegarder les logs de test", modifiez le contenu du fichier `rapport.txt` pour qu'il inclue aussi la version calculée (récupérée depuis l'output du step précédent).

> **Vérification :** dans les logs, le step d'affichage doit montrer toutes les variables. Le fichier `rapport.txt` téléchargé doit contenir le numéro de version.

---

### Étape 3 — Les secrets

Avant de modifier le workflow, créez deux secrets dans votre dépôt GitHub :

`Settings → Secrets and variables → Actions → New repository secret`

| Nom | Valeur (pour les tests) |
|---|---|
| `DEPLOY_TOKEN` | `mon-token-de-deploiement-secret` |
| `NOTIFY_WEBHOOK` | `https://hooks.example.com/test` |

Maintenant ajoutez deux steps à la fin du job `ci` :

**Step "Vérifier les secrets"**
Injectez `DEPLOY_TOKEN` et `NOTIFY_WEBHOOK` en variables d'environnement du step.
Affichez dans les logs :
- la longueur du token (nombre de caractères) sans jamais afficher sa valeur
- les 5 premiers caractères du webhook suivi de `***`
- un message `✅ Secrets correctement configurés` si les deux sont non vides, ou `❌ Secrets manquants` sinon

**Step "Simuler une notification"**
Injectez `NOTIFY_WEBHOOK` en variable d'environnement du step.
Affichez : `Notification envoyée vers [les 8 premiers caractères du webhook]***`

> **Vérification :** dans les logs GitHub, si vous affichez directement la valeur d'un secret, elle apparaît masquée (`***`). Observez ce comportement en ajoutant temporairement `echo $DEPLOY_TOKEN` dans le step.

---

## Critères de réussite

| Critère | Vérifié |
|---|---|
| Le code est bien cloné (`ls -la` montre `index.js`) | ☐ |
| La version de Node.js 20.x apparaît dans les logs | ☐ |
| Les tests passent sans erreur | ☐ |
| L'artifact `rapport-tests` est téléchargeable après le run | ☐ |
| Les variables `APP_NAME`, `NODE_ENV`, `CI`, `RAPPORT_DIR` sont visibles dans les logs | ☐ |
| Le numéro de version dynamique est dans `rapport.txt` | ☐ |
| La valeur de `DEPLOY_TOKEN` n'apparaît jamais en clair dans les logs | ☐ |
| GitHub masque automatiquement la valeur du secret si elle est affichée | ☐ |

---

*© 2025 Mohamed × Utopios — Formation GitHub Actions*
