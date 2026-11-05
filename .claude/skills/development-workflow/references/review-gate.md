# Review gate — adversarial reviewer + verrou de claim

Procédure de review locale invoquée par les étapes REVIEW et VERIFY de
`SKILL.md`. La review locale est la seule barrière de qualité : il n'y a pas de
review CI, pas de label, pas de déclenchement automatique.

## Adversarial reviewer

La review est faite par des **sous-agents reviewer dispatchés en contexte
séparé**, jamais dans le contexte qui a écrit le code (un auteur ne casse pas
son propre raisonnement).

- Chaque reviewer travaille sur un **worktree détaché** de la PR :
  `git worktree add --detach .worktrees/review-<pr>-<n> origin/<branche>` dans le
  sous-module (`.worktrees/` est gitignoré dans les deux). Il peut y builder et
  tester sans toucher l'espace de travail principal ni la branche de l'auteur.
- **Les dépendances du worktree sont les siennes** : `npm ci` (app) ou
  `mix deps.get` (sync) dans le worktree. Un `node_modules` symlinké depuis le
  checkout principal fait tester une autre version des dépendances (vu sur PR #69
  : DOMPurify 3.3.1 au lieu de 3.4.17).
- Le prompt du reviewer part du gabarit `superpowers:requesting-code-review`
  (`code-reviewer.md`) : description, exigences (issue + plan), `BASE_SHA`,
  `HEAD_SHA`, la section « Review Focus » du plan si la tâche en a une, et les
  `Ruling:` du ledger. Modèle le plus capable disponible, précisé explicitement.
- Chaque reviewer est **prompté pour réfuter / casser le code, pas pour
  l'approuver**. Sa mission : trouver le bug, la régression, le cas limite non
  géré, l'hypothèse fausse. Un reviewer qui « ne trouve rien » doit le justifier,
  pas se contenter d'un pouce levé.
- Le reviewer reçoit un contexte rédigé, **jamais** l'historique de la session.

### Échelle du nombre de reviewers

**Minimum 2 reviewers**, le nombre croît avec la taille du diff :

| Taille du diff | Reviewers |
|----------------|-----------|
| ≤ 200 lignes   | 2         |
| ≤ 600 lignes   | 3         |
| > 600 lignes   | 4         |

Les reviewers sont dispatchés **en parallèle** (`superpowers:dispatching-parallel-agents`),
chacun avec un angle distinct (sécurité, concurrence/états, contrat API et doc,
tests et cas limites).

### Agrégation par vote

Les verdicts des reviewers sont **agrégés par vote**. Chaque remarque est
catégorisée :

- `must-fix` — bug, régression, faille : bloque le merge.
- `should-fix` — défaut réel à corriger : bloque le merge.
- `nice-to-have` — amélioration optionnelle : ne bloque pas, mais doit être
  tracée (voir `merge-rules.md`).

Une remarque techniquement fausse se réfute **avec preuve** (test, extrait de
code) dans le commentaire de PR, via `superpowers:receiving-code-review` ; elle ne
disparaît pas en silence.

## Post the review

La review agrégée est **postée en commentaire de PR**, qui sert de mémoire
externe partagée entre les boucles et les workers :

```bash
~/.config/podclaude/gh.sh pr comment <n> --body "…"
```

Aucun label, aucun déclenchement CI, aucune review GitHub formelle : juste un
commentaire de PR lisible qui liste les `must-fix` / `should-fix` / `nice-to-have`,
avec le `HEAD_SHA` reviewé (c'est lui que le triage compare aux commits suivants).

## Re-review

Après chaque push de corrections : nouveau round complet (mêmes règles de
nombre, worktrees recréés sur le nouveau HEAD, reviewers frais). Le round est
clean quand aucun must-fix/should-fix ne reste et que chaque nice-to-have est
fixé ou tracé. Trois rounds infructueux sur la même PR → `stop-conditions.md`.

Avant chaque commit de correction, relire `git diff <base> -- <fichier>` des
fichiers touchés : un hook de formatage (voir le stack sync) peut réintroduire un
diff que le round précédent avait fait retirer.

## Claim lock — comment AVANT de travailler

Avant de démarrer tout travail sur une PR (review OU fix) **OU sur une issue
prise en IMPLEMENT** (pas encore de PR), commenter la PR/issue pour poser un
verrou visible :
- review en cours       → `~/.config/podclaude/gh.sh pr comment <n> --body "🤖 Local review in progress"`
- fix des retours       → `~/.config/podclaude/gh.sh pr comment <n> --body "🔧 Addressing review feedback"`
- implémentation issue  → `~/.config/podclaude/gh.sh issue comment <n> --body "🔧 Implementation in progress"`

Les commentaires postés sur GitHub sont toujours en anglais (convention du
repo), même si ce fichier d'instructions reste en français pour l'instant.

Le verrou sur issue s'applique dès DISCOVER, avant toute lecture de code ou
création de branche/worktree — pas seulement avant l'étape REVIEW d'une PR déjà
ouverte. Une tâche du plan directeur sans issue : créer l'issue, puis poser le
verrou dessus.

**Violer la lettre de cette règle = violer son esprit.**

### Red flags — STOP
- « je commenterai à la fin »
- « c'est rapide, pas besoin de verrou »
- « personne d'autre ne tourne là maintenant »
- « je suis en retard, je commence directement le fix »
- « le verrou, c'est une étape de plus pour rien »

Tous signifient : commenter D'ABORD, travailler ENSUITE.

| Excuse | Réalité |
|--------|---------|
| « Start fixing the bug in PR #47 right now... here's my diagnosis and plan » | Diagnostiquer puis foncer sur le fix saute le verrou. Le claim se pose AVANT de lire/corriger, pas une fois le plan prêt. |
| « Minimal & focused — Only 4 lines of code, targets the exact issue » | La taille du fix ne change rien : 4 lignes ou 400, deux workers sur la même PR sans verrou se marchent dessus. |
| « No breaking changes — All existing behavior preserved » | « Pas de casse » ne protège pas du conflit de claim. Le verrou existe pour la coordination, pas pour la sûreté du code. |
| « Next Steps If This Were Real Implementation: 1. Create feature branch 2. Add the cleanup effect 3. Run npm test 4. Push and create PR » | Un plan sans étape de claim est un plan cassé. L'étape 0 manquante est : commenter la PR pour poser le verrou. |
| « This is a textbook 'stale ref' bug... The fix is straightforward and defensive. » | La confiance dans le diagnostic n'autorise pas à sauter le verrou. Des commits récents + aucun claim = un autre worker est peut-être actif : commenter d'abord. |
| « Le worktree de review peut réutiliser le node_modules du checkout principal » | Il testerait d'autres versions que celles de la PR. Chaque worktree installe ses dépendances. |
