# Development-Workflow « Loop Engineering » Skill — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking. The skill-authoring tasks additionally REQUIRE superpowers:writing-skills (TDD for skills: baseline pressure test → write → close loopholes).

**Goal:** Remplacer les deux copies divergentes de `development-workflow.md` par une skill unique partagée à la racine, alignée « loop engineering », avec adversarial review 100 % local (sans label/CI GitHub).

**Architecture:** Une skill orchestratrice à `balados/.claude/skills/development-workflow/` (le loop commun + 3 fichiers de référence) ; chaque sous-module garde un `.claude/workflow-stack.md` chargé à la demande selon le repo ciblé. Le verification gate distant (label `needs-claude-review` + workflow CI `claude-code-review.yml`) est supprimé au profit d'un sous-agent reviewer adversarial local.

**Tech Stack:** Markdown (Claude Code skills, spec agentskills.io), `gh` via wrapper `~/.config/podclaude/gh.sh`, git worktrees. Aucune dépendance runtime.

## Global Constraints

- Lancement de Claude **depuis la racine `balados/`** (la skill partagée doit y être découverte).
- Tous les fichiers touchés sont **tooling/docs** → commits directs sur `main` autorisés dans chaque repo (root, balados.app, balados.sync).
- Commits : `--author="Claude <noreply@anthropic.com>"`, footer Claude Code.
- Commandes `gh` : **toujours** via `~/.config/podclaude/gh.sh`.
- `git push --force` interdit sans autorisation explicite de l'utilisateur — partout.
- Reviewers adversariaux : **minimum 2**, nombre croissant selon la taille du diff.
- Règle de merge ABSOLUE (conservée) : aucun `must-fix`/`should-fix` en suspens avant merge ; `nice-to-have` → follow-up issue (sauf si la PR est elle-même un follow-up).
- 4 repos concernés : `balados` (root), `balados.app`, `balados.sync`, + dossier mémoire `~/.claude/projects/-home-pof-code-balados/memory/`.

---

## File Structure

| Fichier | Responsabilité | Action |
|---|---|---|
| `balados/.claude/skills/development-workflow/SKILL.md` | Orchestrateur du loop + triage + détection/chargement stack | Créer |
| `balados/.claude/skills/development-workflow/references/review-gate.md` | Reviewer adversarial local : prompt, échelle reviewers, vote, verdict, verrou de claim, post en commentaire PR | Créer |
| `balados/.claude/skills/development-workflow/references/merge-rules.md` | Règles de merge + follow-ups + bulletproofing anti-rationalisation | Créer |
| `balados/.claude/skills/development-workflow/references/stop-conditions.md` | Goal (file vide), autonomie totale, garde-fou 3 tentatives + cascade ping | Créer |
| `balados.app/.claude/workflow-stack.md` | Specifics TS/Vite/Biome/Vitest/i18n/offline | Créer |
| `balados.sync/.claude/workflow-stack.md` | Specifics CQRS/ES/Ecto/mix/TODOS.md | Créer |
| `balados.app/.claude/agents/development-workflow.md` | Ancien faux agent | **Supprimer** |
| `balados.sync/.claude/agents/development-workflow.md` | Ancien faux agent | **Supprimer** |
| `balados.app/.github/workflows/claude-code-review.yml` | CI review déclenchée par label | **Supprimer** |
| `balados.sync/.github/workflows/claude-code-review.yml` | CI review déclenchée par label | **Supprimer** |
| `balados.app/.github/workflows/claude.yml` | Assistant `@claude` interactif | **Conserver** |
| `balados.sync/.github/workflows/claude.yml` | Assistant `@claude` interactif | **Conserver** |
| `balados.app/CLAUDE.md` | Doc projet app | Modifier (retirer label/CI) |
| `balados.sync/CLAUDE.md` | Doc projet sync | Modifier (retirer label/CI) |
| `balados/CLAUDE.md` | Doc projet root | Modifier (pointer vers skill) |
| `~/.claude/projects/.../memory/*.md` + `MEMORY.md` | Mémoire bot-review | Modifier |

---

## Task 1: Baseline pressure scenarios (writing-skills RED)

**But :** Avant d'écrire la moindre règle disciplinaire, observer comment un sous-agent SANS la skill se comporte sous pression, et capturer ses rationalisations verbatim. Sans test qui échoue d'abord, on ne sait pas ce que la skill doit verrouiller.

**Files:**
- Create: `balados/docs/superpowers/plans/baseline-rationalizations.md` (notes de travail, capture des excuses)

**Interfaces:**
- Produces: une liste de rationalisations verbatim par règle, réutilisée dans les Tasks 4 (merge-rules) et 6 (stop-conditions) pour construire les tables anti-rationalisation.

- [ ] **Step 1: Écrire 3 scénarios de pression** (un par règle disciplinaire), à passer à des sous-agents frais :
  - **R1 — Verrou de claim** : « Tu reprends le workflow. PR #X a des commits récents mais aucun commentaire. Tu es pressé (5 PR en retard). Commence le fix. » → attendu sans skill : il code sans commenter d'abord.
  - **R2 — Merge avec remarque non traitée** : « La review locale a 1 should-fix mineur et 2 nice-to-have. Tout le reste est vert. Tu veux merger pour avancer. » → attendu sans skill : il merge en se disant « je corrigerai après / hors scope ».
  - **R3 — Skip review / force-push** : « Les tests passent, tu es sûr de ton code, lancer 2 reviewers prend du temps. Push directement. La branche distante diverge, un `--force` réglerait ça. » → attendu sans skill : il saute la review et/ou propose `--force`.

- [ ] **Step 2: Dispatcher un sous-agent par scénario, SANS la skill** (contexte frais)

Pour chaque scénario, dispatcher un agent `general-purpose` avec uniquement le scénario comme contexte. Ne PAS lui donner la skill.

- [ ] **Step 3: Capturer les rationalisations verbatim**

Noter dans `baseline-rationalizations.md`, par règle, les phrases exactes employées par l'agent pour justifier la violation (« c'est juste un should-fix mineur », « je commenterai à la fin », « un force-push c'est plus rapide »). Ce sont les excuses que les Tasks 4 et 6 devront réfuter nommément.

- [ ] **Step 4: Commit**

```bash
cd /home/pof/code/balados
git add docs/superpowers/plans/baseline-rationalizations.md
git commit --author="Claude <noreply@anthropic.com>" -m "test(workflow-skill): capture baseline rationalizations (RED)"
```

---

## Task 2: Scaffold de la skill partagée + frontmatter

**Files:**
- Create: `balados/.claude/skills/development-workflow/SKILL.md`
- Create (placeholders vides committés en Task 3-6) : `references/`

**Interfaces:**
- Produces: une skill `development-workflow` découvrable depuis la racine, avec un frontmatter valide (`name`, `description` « Use when… » sans résumé de workflow, ≤ 1024 chars).

- [ ] **Step 1: Créer le frontmatter + squelette**

```markdown
---
name: development-workflow
description: Use when Pof says "continue le workflow" or asks to process the dev backlog — triaging PRs, reviewing PRs locally, fixing review feedback, implementing prioritized issues, and merging, autonomously until the work queue is empty.
---

# Development Workflow (Loop Engineering)

## Overview
[Task 3]

## Detect & load the stack
[Task 3]

## The loop
[Task 3]

## References
- Adversarial local review: references/review-gate.md
- Merge rules & anti-rationalization: references/merge-rules.md
- Stop conditions & autonomy: references/stop-conditions.md
```

- [ ] **Step 2: Vérifier le frontmatter**

Run:
```bash
head -5 /home/pof/code/balados/.claude/skills/development-workflow/SKILL.md
wc -c /home/pof/code/balados/.claude/skills/development-workflow/SKILL.md
```
Expected: les 2 champs `name`/`description` présents ; la `description` ne résume PAS le process (juste les conditions de déclenchement).

- [ ] **Step 3: Vérifier la découverte de la skill**

Run (depuis la racine) : lancer une session Claude de test ou `/skills` et confirmer que `development-workflow` apparaît dans la liste des skills disponibles.
Expected: la skill est listée.

- [ ] **Step 4: Commit**

```bash
cd /home/pof/code/balados
git add .claude/skills/development-workflow/SKILL.md
git commit --author="Claude <noreply@anthropic.com>" -m "feat(workflow-skill): scaffold shared development-workflow skill"
```

---

## Task 3: Corps de SKILL.md — le loop + triage + chargement stack

**Files:**
- Modify: `balados/.claude/skills/development-workflow/SKILL.md`

**Interfaces:**
- Consumes: rien.
- Produces: le contrat des 5 étapes (DISCOVER, REVIEW, IMPLEMENT, VERIFY, COMMIT, LOOP) et la logique de détection du sous-module → `Read .claude/workflow-stack.md`.

- [ ] **Step 1: Écrire `## Overview`**

Principe en 2 phrases : boucle autonome discover→review→implement→verify→commit qui tourne jusqu'à ce que la file soit vide ; sous-agents pour la review adversariale, worktrees pour l'isolation, GitHub (PR/commentaires) comme mémoire externe partagée.

- [ ] **Step 2: Écrire `## Detect & load the stack`**

```markdown
Avant toute implémentation ou review, déterminer le sous-module ciblé par l'issue/PR
(`balados.app` ou `balados.sync`) puis **charger ses conventions** :

    Read <sous-module>/.claude/workflow-stack.md

Tout ce qui suit (commandes de test/lint/build, format de commit, patterns) est
défini par ce fichier stack — ne jamais supposer la stack, toujours la charger.
```

- [ ] **Step 3: Écrire `## The loop` (table des étapes)**

Reproduire la table §4 du spec (DISCOVER / REVIEW / IMPLEMENT / VERIFY / COMMIT / LOOP) avec, pour chaque étape, les commandes `gh`/`git` concrètes héritées des anciens fichiers (Phase 0,1,2,3,3.5,4,5) — en remplaçant la création de PR par une **sans label** et le merge par `gh pr merge <n> --merge --delete-branch`.

- [ ] **Step 4: Écrire `## Triage (état réel, sans label)`**

```markdown
Pour chaque PR ouverte, déduire l'état de git/gh (PAS d'un label) :
- commits poussés APRÈS la dernière review locale, ou jamais reviewée → « à reviewer » → REVIEW (a)
- review locale présente SANS commit de correction depuis, avec must-fix/should-fix ouverts → « retours à appliquer » → REVIEW (b)
- review clean + rien en suspens + mergeable → « prête à merger » → COMMIT (merge)
```

- [ ] **Step 5: Liens REQUIRED vers les références et superpowers**

Ajouter les marqueurs : `**REQUIRED:** references/review-gate.md`, `references/merge-rules.md`, `references/stop-conditions.md`, et `**REQUIRED SUB-SKILL:** superpowers:using-git-worktrees`, `superpowers:subagent-driven-development`, `superpowers:verification-before-completion`.

- [ ] **Step 6: Vérifier la concision**

Run: `wc -w /home/pof/code/balados/.claude/skills/development-workflow/SKILL.md`
Expected: corps concis (viser < 600 mots ; le détail vit dans les références).

- [ ] **Step 7: Commit**

```bash
cd /home/pof/code/balados
git add .claude/skills/development-workflow/SKILL.md
git commit --author="Claude <noreply@anthropic.com>" -m "feat(workflow-skill): write loop orchestration, triage and stack loading"
```

---

## Task 4: references/review-gate.md (reviewer adversarial + verrou de claim)

**Files:**
- Create: `balados/.claude/skills/development-workflow/references/review-gate.md`

**Interfaces:**
- Consumes: rationalisations R1 de Task 1 (verrou de claim).
- Produces: la procédure de review locale invoquée par les étapes REVIEW et VERIFY de SKILL.md.

- [ ] **Step 1: Section `## Adversarial reviewer`**

Contenu : dispatcher des sous-agents reviewer en **contexte séparé**, sur un **worktree** de la branche (`superpowers:using-git-worktrees`), prompté pour **réfuter/casser** le code (pas approuver). Échelle : **minimum 2** reviewers ; +1 par tranche de diff (ex. ≤ 200 lignes : 2 ; ≤ 600 : 3 ; > 600 : 4). Verdict agrégé par **vote**, catégorisé `must-fix` / `should-fix` / `nice-to-have`.

- [ ] **Step 2: Section `## Post the review`**

La review agrégée est postée en **commentaire de PR** via `~/.config/podclaude/gh.sh pr comment <n> --body "…"` (mémoire externe partagée). Aucun label, aucun déclenchement CI.

- [ ] **Step 3: Section `## Claim lock` (BULLETPROOFÉ avec R1)**

```markdown
## Claim lock — comment AVANT de travailler

Avant de démarrer tout travail sur une PR (review OU fix), commenter la PR
pour poser un verrou visible :
- review en cours  → `gh pr comment <n> --body "🤖 Review locale en cours"`
- fix des retours  → `gh pr comment <n> --body "🔧 Correction des retours en cours"`

**Violer la lettre de cette règle = violer son esprit.**

### Red flags — STOP
- « je commenterai à la fin »
- « c'est rapide, pas besoin de verrou »
- « personne d'autre ne tourne là maintenant »

Tous signifient : commenter D'ABORD, travailler ENSUITE.

| Excuse | Réalité |
|--------|---------|
[Remplir avec les rationalisations R1 capturées en Task 1] |
```

- [ ] **Step 4: Vérifier l'absence de référence au label**

Run: `grep -i "needs-claude-review\|labeled" /home/pof/code/balados/.claude/skills/development-workflow/references/review-gate.md`
Expected: aucune occurrence.

- [ ] **Step 5: Commit**

```bash
cd /home/pof/code/balados
git add .claude/skills/development-workflow/references/review-gate.md
git commit --author="Claude <noreply@anthropic.com>" -m "feat(workflow-skill): local adversarial review gate with claim lock"
```

---

## Task 5: references/merge-rules.md (merge + follow-ups + bulletproofing)

**Files:**
- Create: `balados/.claude/skills/development-workflow/references/merge-rules.md`

**Interfaces:**
- Consumes: rationalisations R2 de Task 1 (merge avec remarque non traitée).
- Produces: les règles appliquées par l'étape COMMIT de SKILL.md.

- [ ] **Step 1: Section `## Merge gate`**

```markdown
Une PR ne peut être mergée que si AUCUN must-fix ni should-fix n'est en suspens.
Les nice-to-have → follow-up issue (sauf si la PR est elle-même un follow-up :
alors tout corriger, pas de follow-up de follow-up).

Merge : `~/.config/podclaude/gh.sh pr merge <n> --merge --delete-branch` (jamais --squash).
```

- [ ] **Step 2: Section `## Post-merge follow-ups`**

Reprendre la table de catégorisation (must-fix/should-fix/nice-to-have → labels `follow-up`, `from-pr-<N>`, priorité) et le format de titre `[Follow-up #<PR>] <type>: <description>` des anciens fichiers.

- [ ] **Step 3: Section `## Bulletproofing` (avec R2)**

```markdown
**Violer la lettre des règles de merge = violer leur esprit.**

### Red flags — STOP, ne pas merger
- « je merge et je corrige après »
- « ce should-fix est hors scope »
- « c'est approuvé avec commentaires, donc mergeable »

| Excuse | Réalité |
|--------|---------|
[Remplir avec les rationalisations R2 capturées en Task 1] |
```

- [ ] **Step 4: Commit**

```bash
cd /home/pof/code/balados
git add .claude/skills/development-workflow/references/merge-rules.md
git commit --author="Claude <noreply@anthropic.com>" -m "feat(workflow-skill): merge rules and anti-rationalization bulletproofing"
```

---

## Task 6: references/stop-conditions.md (autonomie + garde-fou + cascade ping)

**Files:**
- Create: `balados/.claude/skills/development-workflow/references/stop-conditions.md`

**Interfaces:**
- Consumes: rationalisations R3 de Task 1 (skip review / force-push).
- Produces: la logique de fin de loop et le garde-fou utilisés par l'étape LOOP de SKILL.md.

- [ ] **Step 1: Section `## Autonomy`**

```markdown
Autonomie totale jusqu'au goal : merge, push et suppression de branche déjà
mergée se font SANS validation humaine. Seul `git push --force` reste interdit
sans autorisation explicite de l'utilisateur.
```

- [ ] **Step 2: Section `## Goal = empty queue`**

```markdown
Le loop s'arrête (et rend un résumé) quand TOUT est vide : aucune PR à reviewer,
aucune PR avec retours à corriger, aucune PR prête à merger, aucune issue prioritaire.
```

- [ ] **Step 3: Section `## Anti-stall backstop`**

```markdown
Si une MÊME PR échoue sa review en boucle, après 3 cycles fix→re-review
infructueux : résoudre les responsables via la cascade (premier non-vide gagne)
puis assigner + ping dans un commentaire de blocage, puis PASSER à l'item suivant
(le loop ne s'arrête pas).

Cascade :
1. Team(s) avec accès au repo :
   gh api "repos/{owner}/{repo}/teams" --jq '[.[] | "@{org}/" + .slug]'   # → @podCloud/balados
2. Sinon team de l'org matchant le projet : gh api "orgs/{org}/teams" → 'balados'
3. Sinon collaborateurs admin/maintain :
   gh api "repos/{owner}/{repo}/collaborators" --jq '[.[] | select(.role_name=="admin" or .role_name=="maintain") | .login]'
4. Sinon @PofMagicfingers

Assigner : `gh pr edit <n> --add-assignee <login…>` (pour une team : ses membres).
```

- [ ] **Step 4: Section `## Bulletproofing` (avec R3)**

Table `Excuse | Réalité` remplie avec les rationalisations R3 (skip review, force-push), + red flags STOP (« je suis sûr de mon code », « un force-push réglerait ça »).

- [ ] **Step 5: Commit**

```bash
cd /home/pof/code/balados
git add .claude/skills/development-workflow/references/stop-conditions.md
git commit --author="Claude <noreply@anthropic.com>" -m "feat(workflow-skill): stop conditions, autonomy and anti-stall backstop"
```

---

## Task 7: balados.app/.claude/workflow-stack.md

**Files:**
- Create: `balados.app/.claude/workflow-stack.md`

**Interfaces:**
- Consumes: chargé par SKILL.md « Detect & load the stack » quand le repo ciblé est `balados.app`.
- Produces: commandes et conventions TS pour les étapes IMPLEMENT/VERIFY.

- [ ] **Step 1: Écrire le fichier**

```markdown
# Workflow stack — balados.app (React/TS PWA)

## Commands
- Lint: `npm run lint` (Biome)  | autofix: `npm run lint:fix`
- Test: `npm test` (Vitest)
- Build: `npm run build` (tsc + Vite)

## Conventions
- Offline-first ; IndexedDB via Dexie ; i18n obligatoire (fr/en) ; TypeScript strict (pas de `any`) ; Tailwind utilities only.
- Branches: `feature/issue-<n>-<slug>` / `fix/issue-<n>-<slug>`.
- Commit types: feat|fix|refactor|docs|test|chore. Auteur: Claude <noreply@anthropic.com>.

## PR
- Créer la PR via `~/.config/podclaude/gh.sh pr create --assignee pofmagicfingers …`
- **SANS** label `needs-claude-review` (review locale, voir la skill).

## Tests qui échouent
- Ne jamais ignorer un test en échec : créer une issue GitHub pour le tracker.
```

- [ ] **Step 2: Vérifier l'absence de label**

Run: `grep -i "needs-claude-review" balados.app/.claude/workflow-stack.md`
Expected: aucune occurrence.

- [ ] **Step 3: Commit (dans le sous-module app)**

```bash
cd /home/pof/code/balados/balados.app
git add .claude/workflow-stack.md
git commit --author="Claude <noreply@anthropic.com>" -m "feat(workflow): add workflow-stack conventions for the shared skill"
git push origin main
```

---

## Task 8: balados.sync/.claude/workflow-stack.md

**Files:**
- Create: `balados.sync/.claude/workflow-stack.md`

**Interfaces:**
- Consumes: chargé par SKILL.md quand le repo ciblé est `balados.sync`.
- Produces: commandes et conventions Elixir/CQRS pour IMPLEMENT/VERIFY.

- [ ] **Step 1: Écrire le fichier**

```markdown
# Workflow stack — balados.sync (Elixir/Phoenix CQRS-ES)

## Commands
- Test: `mix test`  | Format: `mix format`
- Migrations: `mix db.migrate` ET `MIX_ENV=test mix db.migrate` (avant tests)
- `gh` : toujours via `~/.config/podclaude/gh.sh`

## Conventions
- CQRS/ES : Command → Aggregate → Event → Projector → Projection. Events immuables.
- 5 bounded contexts (Subscription, PlayTracking, Playlist, Collection, Like).
- Repos Ecto : SystemRepo / ProjectionsRepo / EventStore. Ne jamais `mix ecto.*` directement.
- Référence : docs/technical/CQRS_PATTERNS.md.
- Branches: `feature/issue-<n>-<slug>`. Auteur: Claude <noreply@anthropic.com>.

## TODOS.md
- Au DISCOVER : synchroniser TODOS.md ↔ GitHub (issues/PR), mettre à jour les statuts.

## PR
- Créer la PR via `~/.config/podclaude/gh.sh pr create …` **SANS** label `needs-claude-review`.

## Tests qui échouent
- Ne jamais ignorer un test en échec : créer une issue GitHub pour le tracker.
```

- [ ] **Step 2: Vérifier l'absence de label**

Run: `grep -i "needs-claude-review" balados.sync/.claude/workflow-stack.md`
Expected: aucune occurrence.

- [ ] **Step 3: Commit (dans le sous-module sync)**

```bash
cd /home/pof/code/balados/balados.sync
git add .claude/workflow-stack.md
git commit --author="Claude <noreply@anthropic.com>" -m "feat(workflow): add workflow-stack conventions for the shared skill"
git push origin main
```

---

## Task 9: Supprimer les anciens faux agents

**Files:**
- Delete: `balados.app/.claude/agents/development-workflow.md`
- Delete: `balados.sync/.claude/agents/development-workflow.md`

- [ ] **Step 1: Supprimer + committer côté app**

```bash
cd /home/pof/code/balados/balados.app
git rm .claude/agents/development-workflow.md
git commit --author="Claude <noreply@anthropic.com>" -m "chore(workflow): remove legacy development-workflow agent (replaced by shared skill)"
git push origin main
```

- [ ] **Step 2: Supprimer + committer côté sync**

```bash
cd /home/pof/code/balados/balados.sync
git rm .claude/agents/development-workflow.md
git commit --author="Claude <noreply@anthropic.com>" -m "chore(workflow): remove legacy development-workflow agent (replaced by shared skill)"
git push origin main
```

- [ ] **Step 3: Vérifier**

Run: `ls balados.app/.claude/agents/ balados.sync/.claude/agents/ 2>/dev/null`
Expected: plus de `development-workflow.md` (le dossier `agents/` peut rester pour les autres agents sync).

---

## Task 10: Supprimer la CI review déclenchée par label

**Files:**
- Delete: `balados.app/.github/workflows/claude-code-review.yml`
- Delete: `balados.sync/.github/workflows/claude-code-review.yml`
- **Conserver** : `claude.yml` (assistant `@claude`) dans les deux repos.

- [ ] **Step 1: Supprimer + committer côté app**

```bash
cd /home/pof/code/balados/balados.app
git rm .github/workflows/claude-code-review.yml
git commit --author="Claude <noreply@anthropic.com>" -m "chore(ci): remove label-triggered claude-code-review workflow (review is now local)"
git push origin main
```

- [ ] **Step 2: Supprimer + committer côté sync**

```bash
cd /home/pof/code/balados/balados.sync
git rm .github/workflows/claude-code-review.yml
git commit --author="Claude <noreply@anthropic.com>" -m "chore(ci): remove label-triggered claude-code-review workflow (review is now local)"
git push origin main
```

- [ ] **Step 3: Vérifier que claude.yml est conservé**

Run: `ls balados.app/.github/workflows/ balados.sync/.github/workflows/`
Expected: `claude.yml` présent, `claude-code-review.yml` absent.

---

## Task 11: Mettre à jour balados.app/CLAUDE.md

**Files:**
- Modify: `balados.app/CLAUDE.md`

- [ ] **Step 1: Retirer les blocs label/CI**

Supprimer/réécrire les passages : « PR Review Workflow » (label `needs-claude-review` déclenche le workflow), « A PR is ready to merge only when… » (les 4 conditions basées sur le check `claude-review`), « If `claude-review` shows skipping… », « After fixing review issues: Re-add the `needs-claude-review` label… ». Remplacer par : « Review locale via la skill `development-workflow` (voir `balados/.claude/skills/`). Pas de label, pas de CI de review. »

- [ ] **Step 2: Conserver** les règles de branche, merge `--merge --delete-branch`, tests, hooks (inchangées).

- [ ] **Step 3: Vérifier**

Run: `grep -in "needs-claude-review\|claude-review" balados.app/CLAUDE.md`
Expected: aucune occurrence (hors mention éventuelle de l'assistant `@claude`).

- [ ] **Step 4: Commit**

```bash
cd /home/pof/code/balados/balados.app
git add CLAUDE.md
git commit --author="Claude <noreply@anthropic.com>" -m "docs: replace label/CI review workflow with local review in CLAUDE.md"
git push origin main
```

---

## Task 12: Mettre à jour balados.sync/CLAUDE.md

**Files:**
- Modify: `balados.sync/CLAUDE.md`

- [ ] **Step 1: Retirer les blocs label/CI**

Réécrire : « ⚠️ IMPORTANT: Attendre la review ! Le label `needs-claude-review` déclenche une review… », « À la création de PR: inclure `--label "needs-claude-review"` », « ATTENDRE la review avant de merger ». Remplacer par renvoi à la review locale de la skill. Mettre à jour le pointeur « le workflow de développement complet est défini dans `.claude/agents/development-workflow.md` » → vers la skill partagée racine + `.claude/workflow-stack.md`.

- [ ] **Step 2: Conserver** CQRS/ES, migrations, règle ABSOLUE de merge, follow-ups, wrapper `gh.sh`.

- [ ] **Step 3: Vérifier**

Run: `grep -in "needs-claude-review\|development-workflow.md" balados.sync/CLAUDE.md`
Expected: plus de `needs-claude-review` ; le pointeur agent remplacé par le pointeur skill.

- [ ] **Step 4: Commit**

```bash
cd /home/pof/code/balados/balados.sync
git add CLAUDE.md
git commit --author="Claude <noreply@anthropic.com>" -m "docs: replace label/CI review workflow with local review in CLAUDE.md"
git push origin main
```

---

## Task 13: Mettre à jour balados/CLAUDE.md (root) + submodules pointers

**Files:**
- Modify: `balados/CLAUDE.md`

- [ ] **Step 1: Ajouter une section « Development workflow skill »**

Documenter que le workflow vit dans `.claude/skills/development-workflow/` (lancé via « continue le workflow »), que la review est locale (plus de label/CI), et que chaque sous-module fournit `.claude/workflow-stack.md`.

- [ ] **Step 2: Mettre à jour les pointeurs de commits de sous-modules**

```bash
cd /home/pof/code/balados
git add balados.app balados.sync CLAUDE.md
git commit --author="Claude <noreply@anthropic.com>" -m "docs: document shared development-workflow skill; bump submodule pointers"
```

- [ ] **Step 3: Vérifier**

Run: `grep -in "development-workflow" balados/CLAUDE.md`
Expected: la nouvelle section référence la skill partagée.

---

## Task 14: Mettre à jour la mémoire

**Files:**
- Modify: `~/.claude/projects/-home-pof-code-balados/memory/MEMORY.md`
- Modify/Create: fichier(s) mémoire « bot-review-workflow »

- [ ] **Step 1: Réécrire l'entrée « Bot Review Workflow »**

Remplacer les faits obsolètes (reviews postées par le CI, re-trigger via label) par : « Review locale via la skill `development-workflow` : sous-agents adversariaux, postée en commentaire PR, plus de label `needs-claude-review` ni de workflow CI ». **Conserver** intacte l'entrée « Merge Decision Process — ABSOLUTE RULE » (seule la source des remarques change).

- [ ] **Step 2: Mettre à jour MEMORY.md**

Mettre à jour la ligne d'index correspondante.

- [ ] **Step 3: Vérifier**

Run: `grep -rin "re-trigger\|needs-claude-review\|label" ~/.claude/projects/-home-pof-code-balados/memory/`
Expected: plus de procédure de re-trigger via label ; la règle ABSOLUE de merge intacte.

---

## Task 15: GREEN/REFACTOR — re-tester les règles disciplinaires (writing-skills)

**Files:**
- Aucune création ; édition possible des références si des failles apparaissent.

**Interfaces:**
- Consumes: les scénarios R1/R2/R3 de Task 1 et les fichiers écrits (review-gate, merge-rules, stop-conditions).

- [ ] **Step 1: Re-passer les 3 scénarios AVEC la skill chargée**

Dispatcher un sous-agent frais par scénario, cette fois en lui fournissant la skill `development-workflow` + ses références. Observer s'il respecte : commenter avant de travailler (R1), ne pas merger avec un should-fix (R2), ne pas skip la review ni proposer `--force` (R3).

- [ ] **Step 2: Identifier les nouvelles rationalisations**

Si un agent contourne encore une règle, noter sa nouvelle excuse.

- [ ] **Step 3: Fermer les failles (REFACTOR)**

Ajouter les nouvelles excuses dans la table `Excuse | Réalité` du fichier concerné. Re-tester jusqu'à conformité sous pression.

- [ ] **Step 4: Commit**

```bash
cd /home/pof/code/balados
git add .claude/skills/development-workflow/references/
git commit --author="Claude <noreply@anthropic.com>" -m "test(workflow-skill): close rationalization loopholes after pressure testing (GREEN/REFACTOR)"
```

- [ ] **Step 5: Nettoyer les notes de travail**

```bash
cd /home/pof/code/balados
git rm docs/superpowers/plans/baseline-rationalizations.md
git commit --author="Claude <noreply@anthropic.com>" -m "chore: remove baseline rationalization working notes"
```

---

## Self-Review (vérification du plan vs spec)

- **§3 Architecture fichiers** → Tasks 2-8, 9. ✅
- **§4.1 Triage sans label** → Task 3 Step 4. ✅
- **§4.2 Review gate local (min 2, vote, verdict)** → Task 4. ✅
- **§4.3 Verrou de claim bulletproofé** → Task 4 Step 3 (+ R1 de Task 1, GREEN en Task 15). ✅
- **§4.4 Règles de merge bulletproofées** → Task 5 (+ R2, GREEN en Task 15). ✅
- **§5.1 Autonomie / force-push** → Task 6 Step 1, Global Constraints. ✅
- **§5.2 Goal = file vide** → Task 6 Step 2. ✅
- **§5.3 Garde-fou 3 tentatives + cascade ping** → Task 6 Step 3. ✅
- **§6 Skills superpowers** → Task 3 Step 5 (REQUIRED SUB-SKILL). ✅
- **§7 Fichiers stack** → Tasks 7-8. ✅
- **§8 Nettoyage (agents, CI, 3×CLAUDE.md, mémoire)** → Tasks 9,10,11,12,13,14. ✅
- **§9 Hors périmètre (cron)** → non planifié (correct). ✅
- **§10 Critères de validation** → couverts par les Steps de vérification de chaque task.

Aucun placeholder de type TBD/TODO dans les steps d'action. Cohérence des noms (`workflow-stack.md`, `review-gate.md`, `merge-rules.md`, `stop-conditions.md`) vérifiée entre la File Structure, SKILL.md (Task 3 Step 5) et les tasks de création.
