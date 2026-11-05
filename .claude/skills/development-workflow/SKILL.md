---
name: development-workflow
description: Use when Pof says "continue le workflow", or asks to process / advance the dev backlog (open PRs, prioritized issues, or the next task of the master plan) autonomously.
---

# Development Workflow (Loop Engineering)

## Overview

Une boucle autonome — discover → review → implement → verify → commit → loop — qui tourne jusqu'à ce que la file de travail soit vide. Des sous-agents font la review adversariale, des git worktrees isolent chaque unité de travail, et GitHub (PRs, commentaires d'issues) sert de mémoire externe partagée entre les itérations. **Cette boucle s'exécute EN orchestrant des skills superpowers à chaque étape (voir « Use superpowers » plus bas) — jamais en réimplémentant l'équivalent à la main.**

## Detect & load the stack

Avant toute implémentation ou review, déterminer le sous-module ciblé par l'issue/PR
(`balados.app` ou `balados.sync`) puis **charger ses conventions** :

    Read <sous-module>/.claude/workflow-stack.md

Tout ce qui suit (commandes de test/lint/build, format de commit, pièges connus) est
défini par ce fichier stack — ne jamais supposer la stack, toujours la charger.

> Tous les `gh` passent par `~/.config/podclaude/gh.sh` (jamais `gh` nu).

## Sources de travail (par priorité)

1. **PRs ouvertes** sur les deux dépôts (triage ci-dessous) — on finit avant de commencer.
2. **Plan directeur** : `docs/superpowers/plans/<date>-plan-directeur.md` (dépôt racine).
   La prochaine tâche non cochée de la phase en cours est l'item prioritaire. Une tâche
   du plan devient une issue GitHub (si elle n'en a pas déjà une) avant toute implémentation.
3. **Issues ouvertes** : `follow-up` et `bug` d'abord, puis `enhancement` par ancienneté.
   Une issue `enhancement` sans spec (capture Discord, idée) passe par
   `superpowers:brainstorming` avec Pof avant toute branche — on n'implémente pas une idée.

## The loop

| Étape | Action |
|---|---|
| **DISCOVER** | `~/.config/podclaude/gh.sh pr list` + `issue list` sur chaque dépôt, puis le plan directeur. Choisir l'item via les sources ci-dessus. **Dès l'item choisi : poser le claim lock (`references/review-gate.md`) avant toute lecture de code / branche / commit — le verrou s'applique aux issues comme aux PR.** |
| **REVIEW** | (a) PR à reviewer → worktree détaché + sous-agents reviewers adversariaux (voir `references/review-gate.md`). (b) Retours à appliquer → fixer, puis re-review. |
| **IMPLEMENT** | Worktree isolé (`.worktrees/` du sous-module), TDD, conventions de la stack chargée. Pour une tâche du plan directeur, suivre ses étapes et tenir le ledger `.superpowers/sdd/<plan>/progress.md` (racine, gitignoré). |
| **VERIFY** | Lancer test/lint/build de la stack, PUIS review adversariale locale de son propre travail (`references/review-gate.md`). Gate vérifiable avant merge : tests verts ET reviewers OK ET chaque remarque fixée ou tracée. |
| **COMMIT** | Commit (author Claude), push, PR **sans label** : `~/.config/podclaude/gh.sh pr create ...`. Si prête à merger : `~/.config/podclaude/gh.sh pr merge <n> --merge --delete-branch`, **puis** synchroniser et mettre à jour la doc racine et le plan (voir `references/merge-rules.md#post-merge-doc-sync`) avant de reboucler. |
| **LOOP** | Revenir à DISCOVER tant que la file n'est pas vide (voir `references/stop-conditions.md`). |

## Triage (état réel, sans label)

Pour chaque PR ouverte, déduire l'état de git/gh (PAS d'un label) :
- commits poussés APRÈS la dernière review locale, ou jamais reviewée → « à reviewer » → REVIEW (a)
- review locale présente SANS commit de correction depuis, avec must-fix/should-fix ouverts → « retours à appliquer » → REVIEW (b)
- review clean + rien en suspens + mergeable → « prête à merger » → COMMIT (merge)

## Use superpowers (REQUIRED — non négociable)

Ce workflow EST une boucle superpowers. À chaque étape, invoquer la skill
superpowers correspondante AVANT d'agir — ne jamais improviser l'équivalent à la main.

| Étape du loop | Skill superpowers à invoquer |
|---|---|
| Issue sans spec (feature, idée) | `superpowers:brainstorming` puis `superpowers:writing-plans` |
| Exécution d'une tâche du plan directeur | `superpowers:executing-plans` (natif) ou `superpowers:subagent-driven-development` |
| Isolation du travail (review ET implement) | `superpowers:using-git-worktrees` |
| Dispatch des sous-agents reviewers | `superpowers:dispatching-parallel-agents` (N reviewers en parallèle) |
| Implémentation d'un fix/feature | `superpowers:test-driven-development` |
| Review adversariale locale | `superpowers:requesting-code-review` + `superpowers:receiving-code-review` |
| Gate « c'est fait » avant push/merge | `superpowers:verification-before-completion` |
| Debug d'un échec | `superpowers:systematic-debugging` |
| Clôture de branche | `superpowers:finishing-a-development-branch` |

**Violer la lettre de cette règle = violer son esprit.** Red flag STOP :
« je vais juste lancer un agent review vite fait sans passer par la skill ».
Si une skill superpowers couvre l'étape, on l'invoque — pas d'exception.

## References

- **REQUIRED:** Adversarial local review: references/review-gate.md
- **REQUIRED:** Merge rules & anti-rationalization: references/merge-rules.md
- **REQUIRED:** Stop conditions & autonomy: references/stop-conditions.md
