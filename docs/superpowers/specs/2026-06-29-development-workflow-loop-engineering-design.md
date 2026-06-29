# Design — Skill `development-workflow` « loop engineering »

**Date** : 2026-06-29
**Statut** : design validé (brainstorming), en attente de revue du spec avant plan d'implémentation
**Périmètre** : monorepo `balados/` + sous-modules `balados.app` et `balados.sync`

---

## 1. Contexte & problème

Le workflow de développement existe aujourd'hui sous forme d'un **faux agent** (`.claude/agents/development-workflow.md`) dupliqué dans les deux sous-modules :

- `balados.app/.claude/agents/development-workflow.md` — 317 lignes (build TS/Vite, label `needs-claude-review`)
- `balados.sync/.claude/agents/development-workflow.md` — 654 lignes (CQRS/ES, migrations Ecto, TODOS.md)

Problèmes identifiés :

1. **Divergence non maîtrisée** — deux copies à maintenir en parallèle, déjà désynchronisées (la version app connaît le label `needs-claude-review`, pas la sync).
2. **« Faux agent »** — les deux fichiers déclarent eux-mêmes « Claude n'utilise PAS d'agent, il exécute directement ». Sémantiquement c'est un **guide/skill**, rangé au mauvais endroit.
3. **Verification gate distant et lent** — la review passe par un label GitHub (`needs-claude-review`) + un workflow CI (`claude-review`), avec aller-retour push → label → attente → correction → re-label.

## 2. Objectifs

1. **Dédupliquer** : un tronc commun unique partagé, des spécificités par stack isolées (priorité #1).
2. **Convertir en vraie skill** Claude Code, partagée à la racine (priorité #3).
3. **Aligner sur le « loop engineering »** (Addy Osmani / Claire Vo « How I AI ») : boucle autonome discover → implement → verify → commit, sous-agents, worktrees, mémoire externe, conditions d'arrêt vérifiables.
4. **Remplacer le verification gate distant** (label + CI) par un **adversarial review 100 % local** via sous-agent.

### Alignement loop engineering (6 piliers)

| Pilier | Implémentation |
|---|---|
| Automatisations (déclencheur) | `continue le workflow` (manuel) + goal loop autonome interne. Routine cron = add-on optionnel futur. |
| Worktrees (isolation) | un worktree par item de travail (implémentation + review) |
| Skills (savoir centralisé) | la skill `development-workflow` + superpowers |
| Connecteurs externes | GitHub (PR/issues/commentaires) = mémoire externe partagée |
| Sous-agents (création ≠ révision) | reviewer adversarial en contexte séparé |
| Mémoire externe sur disque | CLAUDE.md, TODOS.md (sync), commentaires de PR |

## 3. Architecture des fichiers (décision : « core racine + refs par stack »)

```
balados/.claude/skills/development-workflow/
  SKILL.md                  # Orchestrateur : le loop + règles communes (~70 %)
  references/
    review-gate.md          # Reviewer adversarial local (prompt, critères, vote)
    merge-rules.md          # Merge, follow-ups, anti-rationalisation (bulletproofing)
    stop-conditions.md      # Goal, garde-fou anti-blocage, autonomie
balados.app/.claude/workflow-stack.md     # build TS/Vite, Biome, Vitest, i18n, offline-first
balados.sync/.claude/workflow-stack.md    # CQRS/ES, migrations Ecto, mix test/format, TODOS.md
```

- Claude est lancé **depuis la racine `balados/`** → la skill partagée est visible.
- Au début de la phase IMPLEMENT (et de la phase REVIEW), la skill **détecte le sous-module ciblé** par l'issue/PR et fait un `Read` du `workflow-stack.md` correspondant.
- **Zéro duplication** du tronc commun. Chaque sous-module **possède ses conventions**, versionnées avec son code et reviewées dans ses propres PR.
- Les anciens `.claude/agents/development-workflow.md` (app + sync) sont **supprimés**.

## 4. Le loop (discover → review → implement → verify → commit → loop)

| Étape | Contenu | Origine |
|---|---|---|
| **DISCOVER** | Pre-flight (working dir clean, sur main, auth) + sync TODOS.md (sync) → **Triage** des PR et issues | Phases 0,1,3,3.5 |
| **REVIEW** *(local, sans label/CI)* | (a) PR à reviewer → reviewer adversarial → review postée en commentaire PR. (b) PR avec retours → corrige → re-review | nouveau + Phase 2 |
| **IMPLEMENT** | issue choisie → worktree → code selon `workflow-stack.md` → tests/lint/build/migrations | Phase 4 |
| **VERIFY** | review adversarial locale de SON propre travail (N reviewers, vote). Gate vérifiable : tests verts ET reviewers OK | nouveau (remplace bot CI) |
| **COMMIT** | merge `--no-ff` quand review locale clean → push/PR (sans label) → follow-ups post-merge | Phases 1,5 |
| **LOOP** | conditions d'arrêt + self-healing de la skill | Phases 6,7 |

### 4.1 Triage basé sur l'état réel (plus de label)

Pour chaque PR ouverte, l'état est **déduit de git/gh** (dates de commits vs dates de reviews), pas d'un label :

- **commits poussés APRÈS la dernière review locale, ou jamais reviewée** → « à reviewer » → REVIEW (a)
- **review locale présente SANS commit de correction depuis** avec des `must-fix`/`should-fix` ouverts → « retours à appliquer » → REVIEW (b)
- **review clean + rien en suspens + mergeable** → « prête à merger » → COMMIT (merge)

### 4.2 Review gate local (cœur du changement)

- **Sous-agent dédié, contexte séparé**, prompté pour **réfuter/casser** le code (pas pour approuver).
- Tourne sur un **worktree** de la branche (isolation).
- **Vote** : N reviewers, **minimum 2**, N croissant selon la taille du diff (échelle exacte fixée pendant `writing-skills`) ; verdict catégorisé `must-fix` / `should-fix` / `nice-to-have`.
- **Trace** : la review est postée en **commentaire de PR** (mémoire externe partagée entre runs/agents).
- **Plus de label `needs-claude-review`, plus de workflow CI `claude-review`.**

### 4.3 Verrou de claim (anti-collision multi-agent) — RÈGLE BULLETPROOFÉE

**Avant de démarrer tout travail sur une PR** (review *ou* fix), l'agent **commente la PR** pour poser un verrou visible :

- « 🤖 Review locale en cours »
- « 🔧 Correction des retours en cours »

**Raison** : sans ce commentaire, on accumule des PR avec des commits mais aucun commentaire, et aucun agent ne sait qu'un autre y travaille → travail en double / boucle infinie. **Le commentaire = le verrou.**

Bulletproofing : pas d'exception « je commenterai après ». Commenter d'abord, travailler ensuite. Red flag STOP : « je commence le fix et je commenterai à la fin ».

### 4.4 Règles de merge (inchangées, juste la source des remarques change) — BULLETPROOFÉ

- **Aucun `must-fix` ni `should-fix` en suspens** avant merge.
- `nice-to-have` → **follow-up issue** (sauf si la PR est elle-même un follow-up → tout corriger, pas de follow-up de follow-up).
- Red flag STOP : « je merge et je corrige après », « cette remarque est hors scope ».
- Merge : `gh pr merge <n> --merge --delete-branch` (jamais `--squash`), `--no-ff`.

## 5. Autonomie & conditions d'arrêt

### 5.1 Autonomie totale (plus de human gate)

- L'agent **merge, push, et supprime les branches déjà mergées librement**, sans validation humaine.
- **Seul `git push --force` reste interdit** sans autorisation explicite de l'utilisateur.
- Le loop tourne **jusqu'au goal**, sans interruption pour confirmation.

### 5.2 Goal = file de travail vide (condition d'arrêt vérifiable)

Le loop s'arrête (et rend un résumé) quand **tout** est vide :

- aucune PR à reviewer,
- aucune PR avec retours à corriger,
- aucune PR prête à merger,
- aucune issue prioritaire restante.

### 5.3 Garde-fou anti-blocage (par item)

Si une **même** PR échoue sa review en boucle (fix → re-review → toujours des `must-fix`) :

- **après 3 cycles fix→re-review infructueux**, l'agent **résout les responsables** via une cascade (premier non-vide gagne) :
  1. **Team(s) ayant accès explicite au repo** (source settings-driven, prioritaire) :
     `gh api "repos/{owner}/{repo}/teams" --jq '[.[] | "@{org}/" + .slug]'` → ping de la team (`@podCloud/balados`).
  2. Sinon, **team de l'org dont le slug/nom matche le projet** :
     `gh api "orgs/{org}/teams" --jq '...'` → match sur `balados`.
  3. Sinon, **collaborateurs admin/maintain** :
     `gh api "repos/{owner}/{repo}/collaborators" --jq '[.[] | select(.role_name=="admin" or .role_name=="maintain") | .login]'`
  4. Sinon, **@PofMagicfingers**.
- puis **assigne** les personnes résolues sur la PR (`gh pr edit <n> --add-assignee …` ; pour une team : assigner ses membres) **et les ping** dans un commentaire de blocage expliquant l'échec après 3 tentatives ;
- puis **passe à l'item suivant** de la file (le loop ne s'arrête pas, il continue le reste).

> Décision : la team `@podCloud/balados` est **attachée aux deux repos** (rôle maintain/admin) → l'étape 1 résout directement. Tout futur membre de la team est ainsi ping/assigné automatiquement, sans hardcoder.

> Note org : le repo appartient à l'org `podCloud` (on ne peut pas assigner l'org). Les rôles `admin`/`maintain` viennent des **settings du repo** (Collaborators and teams), pas du fait d'avoir une PR mergée : un contributeur externe open source n'y figure jamais tant qu'il n'est pas ajouté explicitement — c'est le comportement voulu (on ping les responsables, pas un contributeur de passage).
>
> La permission **Organization → Members: Read** a été accordée à la GitHub App du wrapper `gh.sh` → le bot peut désormais lister les teams. La team `@podCloud/balados` est attachée aux deux repos (voir cascade §5.3), ce qui en fait la source de ping pluriel et dynamique privilégiée. La détection des collaborateurs admin/maintain reste en fallback.

## 6. Skills superpowers mobilisées

- `subagent-driven-development` / `dispatching-parallel-agents` — orchestration des sous-agents reviewer/implémenteur
- `using-git-worktrees` — isolation par item
- `requesting-code-review` + adversarial — le review gate local
- `verification-before-completion` — gate vérifiable avant de déclarer « fait »
- `writing-skills` — pour rédiger la SKILL.md elle-même (bulletproofing, SDO, efficacité tokens)

## 7. Contenu des fichiers stack

### `balados.app/.claude/workflow-stack.md`
- Commandes : `npm run lint` (Biome), `npm test` (Vitest), `npm run build` (tsc+Vite)
- Conventions : offline-first, IndexedDB/Dexie, i18n obligatoire, TypeScript strict, Tailwind utilities
- Branches : `feature/issue-<n>-<slug>` / `fix/issue-<n>-<slug>`
- PR : assigner `pofmagicfingers`, **sans** label `needs-claude-review`
- Commit type set : `feat|fix|refactor|docs|test|chore`

### `balados.sync/.claude/workflow-stack.md`
- Commandes : `mix test`, `mix format`, migrations `mix db.migrate` + `MIX_ENV=test mix db.migrate`
- Conventions : CQRS/ES (Command→Aggregate→Event→Projector→Projection), events immuables, 5 bounded contexts
- Repos Ecto : SystemRepo / ProjectionsRepo / EventStore
- TODOS.md : sync queue ↔ GitHub
- Wrapper `~/.config/podclaude/gh.sh` pour toutes les commandes `gh`
- PR : **sans** label `needs-claude-review`

## 8. Nettoyage / migrations associées (même lot)

- **Supprimer** `balados.app/.claude/agents/development-workflow.md` et `balados.sync/.claude/agents/development-workflow.md`.
- **Supprimer le workflow CI** `claude-review` (`.github/workflows/…`) dans les deux repos **et** retirer le label `needs-claude-review` de tout le process.
- **Mettre à jour** `balados.app/CLAUDE.md` et `balados.sync/CLAUDE.md` : retirer les blocs « label / attendre la review CI / re-trigger via label » → renvoyer vers la review locale de la skill.
- **Mettre à jour** `balados/CLAUDE.md` (racine) : pointer vers la nouvelle skill partagée.
- **Mettre à jour la mémoire** (`~/.claude/projects/.../memory/`) : réécrire les entrées « Bot Review Workflow » (re-trigger via label, reviews postées par le CI) pour le review local. **La règle ABSOLUE de merge reste** (seule la source des remarques change).

## 9. Hors périmètre (pour plus tard)

- Routine/cron qui lance le loop sans déclenchement manuel (add-on optionnel).
- Parallélisme multi-PR simultané au-delà de l'isolation par worktree.

## 10. Critères de validation (definition of done du design)

- [ ] La skill partagée existe à la racine et est découverte depuis `balados/`.
- [ ] Les deux `workflow-stack.md` existent et sont chargés selon le repo ciblé.
- [ ] Le triage classe les PR sans aucun label.
- [ ] La review tourne en local via sous-agent adversarial, postée en commentaire PR.
- [ ] Le verrou de claim (commentaire avant travail) est bulletproofé.
- [ ] Plus aucune référence au label `needs-claude-review` ni au workflow CI dans les repos.
- [ ] Autonomie totale ; seul `git push --force` reste interdit.
- [ ] Garde-fou 3 tentatives → assign+ping dynamique → skip item.
- [ ] Les deux anciens fichiers `agents/development-workflow.md` sont supprimés.
- [ ] CLAUDE.md (×3) et mémoire mis à jour.
