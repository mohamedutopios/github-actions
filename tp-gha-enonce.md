# TP GitHub Actions — Pipeline CI/CD complet

**Formation GitHub Actions · Utopios × Mohamed**
 · Un seul fichier `.yml` à construire

---

## Contexte

Vous intégrez l'équipe DevOps d'une startup. Votre mission : mettre en place le pipeline CI/CD de l'application `utopios-app` de zéro, en un seul fichier GitHub Actions.

Ce pipeline doit :

- se déclencher automatiquement sur les bons événements
- exécuter des contrôles qualité en parallèle
- enchaîner les étapes dans le bon ordre
- déployer uniquement quand tout est vert

---

## Mise en place

Créez un dépôt GitHub public nommé **`utopios-app`**, clonez-le, puis créez le dossier des workflows :

```bash
git clone https://github.com/VOTRE_USERNAME/utopios-app.git
cd utopios-app
mkdir -p .github/workflows
```

Tout votre travail se fera dans un seul fichier :

```
.github/workflows/pipeline.yml
```

---

## Travail demandé

### Étape 1 — Les déclencheurs

Configurez le workflow pour qu'il se déclenche dans **trois situations** :

1. À chaque `push` sur les branches `main` et `develop` uniquement
2. À chaque `pull_request` ciblant la branche `main`
3. Manuellement, avec un paramètre `environment` au choix entre `staging` et `production`

> **Vérification :** poussez le fichier sur `main`. Le workflow doit apparaître dans l'onglet Actions. Ensuite, créez une branche `feature/test`, poussez dessus : aucun run ne doit se déclencher.

---

### Étape 2 — Les jobs et leurs steps

Le pipeline contient **quatre jobs**. Pour l'instant, créez-les tous sans `needs` (ils tourneront en parallèle, vous les enchaînerez à l'étape suivante).

Chaque job tourne sur `ubuntu-latest`.

**Job `lint`** — Vérification du style

Contient 2 steps :
- Step 1 : affiche `Analyse du style en cours...` puis attend 2 secondes (`sleep 2`)
- Step 2 : affiche `✅ Aucune erreur de style détectée`

**Job `tests`** — Tests automatisés

Contient 3 steps :
- Step 1 : affiche `Lancement des tests...`
- Step 2 : attend 4 secondes, affiche `✅ 21 tests passés, 0 échoué`
- Step 3 : doit **toujours s'exécuter**, même si les steps précédents échouent — il affiche `Rapport de tests généré`

**Job `build`** — Compilation

Contient 2 steps :
- Step 1 : affiche `Compilation de utopios-app...`, attend 3 secondes
- Step 2 : affiche `✅ Build réussi — artefact prêt`

**Job `deploy`** — Déploiement

Contient 2 steps :
- Step 1 : affiche `Déploiement sur [valeur de l'input environment]...`
- Step 2 : affiche `✅ utopios-app est en ligne !`

> **Vérification :** déclenchez manuellement. Les 4 jobs doivent démarrer en même temps. Observez le graphe dans l'onglet Summary.

---

### Étape 3 — L'orchestration avec `needs`

Les 4 jobs doivent maintenant s'exécuter dans cet ordre précis :

```
lint ──┐
       ├──→ build ──→ deploy
tests ─┘
```

- `lint` et `tests` tournent **en parallèle**
- `build` attend que **les deux** soient terminés avec succès
- `deploy` attend que `build` soit terminé
- `deploy` ne s'exécute **que sur la branche `main`**

> **Vérification 1 :** déclenchez manuellement. Le graphe doit refléter exactement le schéma ci-dessus.

> **Vérification 2 :** faites intentionnellement échouer le job `tests` (ajoutez `exit 1` dans un de ses steps). Relancez. Les jobs `build` et `deploy` doivent être annulés. Le job `lint` doit lui terminer normalement.

> **Vérification 3 :** retirez le `exit 1`, poussez sur `develop`. Le job `deploy` doit être ignoré (skipped), les 3 autres doivent passer au vert.

---

## Critères de réussite

| Critère | Vérifié |
|---|---|
| Le workflow ne se déclenche pas sur `feature/*` | ☐ |
| Le workflow se déclenche sur push `main`, push `develop`, PR vers `main`, et manuellement | ☐ |
| Le déclenchement manuel affiche bien la valeur de `environment` dans les logs | ☐ |
| Le step de rapport dans `tests` s'exécute même en cas d'échec | ☐ |
| `lint` et `tests` démarrent simultanément | ☐ |
| `build` n'attend bien que `lint` ET `tests` | ☐ |
| `deploy` s'exécute uniquement sur `main` | ☐ |
| Un échec dans `tests` annule `build` et `deploy` mais pas `lint` | ☐ |

---

*© 2025 Mohamed × Utopios — Formation GitHub Actions*
