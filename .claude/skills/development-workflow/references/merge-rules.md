# Merge rules — merge gate, follow-ups et bulletproofing

Règles appliquées par l'étape COMMIT de `SKILL.md`. Une review locale produit
des verdicts `must-fix` / `should-fix` / `nice-to-have` (voir `review-gate.md`) ;
ce fichier décide quand et comment merger en fonction de ces verdicts.

## Merge gate

Une PR ne peut être mergée que si AUCUN must-fix ni should-fix n'est en suspens.
Les nice-to-have → follow-up issue (sauf si la PR est elle-même un follow-up :
alors tout corriger, pas de follow-up de follow-up).

Merge : `~/.config/podclaude/gh.sh pr merge <n> --merge --delete-branch` (jamais --squash).

`--merge` préserve l'historique des commits de la branche ; `--delete-branch`
nettoie la branche mergée. On n'utilise jamais `--squash` (perte de granularité).

## Post-merge follow-ups

Chaque nice-to-have non corrigé doit avoir une issue de suivi créée AVANT le
merge. Catégorisation :

| Verdict de review | Action au merge | Labels de l'issue de suivi | Priorité |
|-------------------|-----------------|----------------------------|----------|
| `must-fix`        | corriger dans la PR (bloque le merge) | — (aucune issue) | — |
| `should-fix`      | corriger dans la PR (bloque le merge) | — (aucune issue) | — |
| `nice-to-have`    | créer un follow-up (ou corriger) | `follow-up`, `from-pr-<N>` | basse |

Format de titre des issues de suivi :

```
[Follow-up #<PR>] <type>: <description>
```

Exemple : `[Follow-up #64] refactor: extract useLike hook polling logic`.

Exception : si la PR est elle-même un follow-up, on ne crée PAS de follow-up de
follow-up — tout est corrigé directement dans la PR avant merge. Cette exception
s'applique **même si** le constat porte sur un fichier hors du diff de la PR, ou
sur un bug préexistant (pas une régression introduite par la PR) : « ce n'est pas
dans mon diff » et « c'était déjà comme ça avant » ne sont pas des motifs pour
retomber sur un follow-up quand la PR en cours est déjà un follow-up.

## Post-merge doc sync

Dernière action d'un item qui se termine par un merge (ou par la fermeture
d'une issue devenue obsolète/doublon) — **avant** de reboucler sur DISCOVER :

1. Se placer dans le repo racine `balados/` (le superprojet qui contient les
   submodules `balados.app` et `balados.sync`).
2. Synchroniser : `git fetch origin main`, mettre à jour le checkout local
   (et les pointeurs de submodule si le merge a fait avancer `balados.app`
   ou `balados.sync`).
3. Vérifier l'état réel de **tout le projet**, pas seulement l'item qu'on
   vient de traiter : PR ouvertes/récemment mergées et issues ouvertes/
   récemment fermées sur `balados.app` ET `balados.sync` (`gh pr list`,
   `gh issue list`, avec `--repo` pour chaque submodule).
4. Mettre à jour `balados/CLAUDE.md` (racine — principalement la section
   « Current Work ») pour refléter cet état réel : déplacer vers
   « Completed » ce qui est mergé/fermé, retirer les entrées obsolètes,
   corriger la liste « Open Work » avec le backlog effectivement ouvert.
5. Commit + push directement sur `main` du superprojet — une mise à jour de
   documentation racine est une des exceptions au « jamais de commit direct
   sur main » (voir `balados.app/CLAUDE.md` § RÈGLES CRITIQUES).

Ce n'est pas une étape à faire « à un moment » dans la boucle : c'est la
dernière chose qu'on fait pour un item avant de reboucler, à chaque fois
qu'un merge (ou une fermeture d'issue) change l'état réel du projet.

## Bulletproofing

**Violer la lettre des règles de merge = violer leur esprit.**

### Red flags — STOP, ne pas merger
- « je merge et je corrige après »
- « ce should-fix est hors scope »
- « c'est approuvé avec commentaires, donc mergeable »
- « c'est juste cosmétique »
- « ça ne débloque pas le ship »
- « ce n'est pas dans le diff de cette PR »
- « c'est préexistant, pas une régression de cette PR »
- « je synchroniserai la doc racine plus tard / au prochain item »

| Excuse | Réalité |
|--------|---------|
| « Skip the test IDs — pure cosmetics. » | « Cosmétique » n'est pas une catégorie de merge. Un nice-to-have non corrigé exige une follow-up issue avant merge — pas un abandon silencieux. |
| « Adding these now won't make the feature ship better, it just makes the test harness smoother *someday*. Defer it or skip it entirely. » | « Defer or skip entirely » est exactement la violation : différer est permis SEULEMENT via une follow-up issue tracée. « Skip entirely » ne l'est jamais. |
| « Skip the hook extraction — classic over-engineering impulse. » | Juger un nice-to-have « over-engineering » n'autorise pas à l'effacer. Soit on le corrige, soit on le trace ; le mérite technique se débat dans l'issue. |
| « Extract when you have *proven* reuse, not when you *predict* you might use it. » | Bonne maxime d'ingénierie, mauvaise raison de ne rien tracer. La décision de ne pas corriger se documente dans une follow-up issue, elle ne se prend pas en silence. |
| « Everything else is nice-to-have that doesn't unblock shipping. Rename the prop, merge, ship it. That's what I'd actually do. » | « Doesn't unblock shipping » ne dispense pas du suivi. Chaque nice-to-have non touché = une follow-up issue, sinon pas de merge. |
| « The math: Feature works. Tests pass. Sprint deadline is now. » | La deadline ne change pas la gate. Tests verts + feature OK n'autorisent pas à sauter le tracking des remarques restantes. |
| « Ce finding touche PlayerProvider.tsx, pas les fichiers de mon diff — hors scope. » | Le fichier touché par la review n'a pas à être dans le diff pour que la règle s'applique. Si la PR en cours est déjà un follow-up, un nice-to-have trouvé n'importe où par la review se corrige dans la PR, il ne devient pas un nouveau follow-up sous prétexte qu'il vit ailleurs. |
| « Ce comportement existait déjà avant la PR, ce n'est pas une régression que j'ai introduite. » | Préexistant ne veut pas dire hors périmètre de correction. La règle « pas de follow-up de follow-up » porte sur ce que la review trouve, pas sur qui l'a introduit. |
| « La doc racine, je la mettrai à jour groupée plus tard, ça ira plus vite. » | Groupée = jamais faite. Chaque merge qui change l'état réel du projet doit se refléter dans `CLAUDE.md` avant de reboucler, pas dans un futur batch hypothétique. |
