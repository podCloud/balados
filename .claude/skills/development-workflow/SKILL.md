---
name: development-workflow
description: Use when Pof says "continue le workflow" or asks to process the dev backlog — triaging PRs, reviewing PRs locally, fixing review feedback, implementing prioritized issues, and merging, autonomously until the work queue is empty.
---

# Development Workflow (Loop Engineering)

## Overview

An autonomous loop — discover → review → implement → verify → commit → loop — that runs until the work queue is empty. Subagents perform adversarial review, git worktrees isolate each unit of work, and GitHub (PRs, issue comments) is the shared external memory across iterations. **This loop is executed BY orchestrating superpowers skills at every step (see "Use superpowers" below) — it is never a manual re-implementation.**

## Detect & load the stack

Avant toute implémentation ou review, déterminer le sous-module ciblé par l'issue/PR
(`balados.app` ou `balados.sync`) puis **charger ses conventions** :

    Read <sous-module>/.claude/workflow-stack.md

Tout ce qui suit (commandes de test/lint/build, format de commit, patterns) est
défini par ce fichier stack — ne jamais supposer la stack, toujours la charger.

> Tous les `gh` passent par `~/.config/podclaude/gh.sh` (jamais `gh` nu).

## The loop

| Étape | Action |
|---|---|
| **DISCOVER** | `~/.config/podclaude/gh.sh pr list` + `issue list`. Trier les PRs ouvertes via Triage ci-dessous ; sinon prendre l'issue priorisée suivante. |
| **REVIEW** | (a) PR à reviewer → worktree + sous-agents reviewers adversariaux (voir `references/review-gate.md`). (b) Retours à appliquer → fixer, puis re-review. |
| **IMPLEMENT** | Worktree isolé, TDD, conventions de la stack chargée. |
| **VERIFY** | Lancer test/lint/build de la stack. Gate « c'est fait » avant tout push. |
| **COMMIT** | Commit (author Claude), push, PR **sans label** : `gh pr create ...`. Si prête à merger : `gh pr merge <n> --merge --delete-branch`. |
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
| Isolation du travail (review ET implement) | `superpowers:using-git-worktrees` |
| Dispatch des sous-agents (reviewer, fix) | `superpowers:subagent-driven-development` (ou `dispatching-parallel-agents` pour les N reviewers) |
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
