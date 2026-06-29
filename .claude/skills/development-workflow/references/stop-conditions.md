# Stop conditions — autonomie, goal et garde-fou anti-stall

Logique de fin de loop et garde-fou utilisés par l'étape LOOP de `SKILL.md`.

## Autonomy

Autonomie totale jusqu'au goal : merge, push et suppression de branche déjà
mergée se font SANS validation humaine. Seul `git push --force` reste interdit
sans autorisation explicite de l'utilisateur.

`git push --force` est **FORMELLEMENT INTERDIT** tant que l'utilisateur ne l'a
pas explicitement autorisé pour cette opération précise. Pas de substitut, pas
de contournement : si un push est rejeté pour cause de divergence, on
n'écrase pas l'historique distant de sa propre initiative — on s'arrête et on
demande.

## Goal = empty queue

Le loop s'arrête (et rend un résumé) quand TOUT est vide : aucune PR à reviewer,
aucune PR avec retours à corriger, aucune PR prête à merger, aucune issue prioritaire.

## Anti-stall backstop

Si une MÊME PR échoue sa review en boucle, après 3 cycles fix→re-review
infructueux : résoudre les responsables via la cascade (premier non-vide gagne)
puis assigner + ping dans un commentaire de blocage, puis PASSER à l'item suivant
(le loop ne s'arrête pas).

Cascade (toutes les commandes `gh` via `~/.config/podclaude/gh.sh`) :
1. Team(s) avec accès au repo :
   `~/.config/podclaude/gh.sh api "repos/{owner}/{repo}/teams" --jq '[.[] | "@{org}/" + .slug]'`   # → @podCloud/balados
2. Sinon team de l'org matchant le projet : `~/.config/podclaude/gh.sh api "orgs/{org}/teams"` → 'balados'
3. Sinon collaborateurs admin/maintain :
   `~/.config/podclaude/gh.sh api "repos/{owner}/{repo}/collaborators" --jq '[.[] | select(.role_name=="admin" or .role_name=="maintain") | .login]'`
4. Sinon @PofMagicfingers

Assigner : `~/.config/podclaude/gh.sh pr edit <n> --add-assignee <login…>` (pour une team : ses membres).

Le ping de blocage est un commentaire de PR via `~/.config/podclaude/gh.sh pr
comment <n>` qui nomme les responsables résolus et décrit pourquoi la PR est
bloquée après 3 cycles. Après l'assignation et le ping, le loop ne s'arrête PAS :
il passe à l'item suivant de la queue.

## Bulletproofing

**Violer la lettre de ces règles = violer leur esprit.**

### Red flags — STOP
- « je suis sûr de mon code »
- « un force-push réglerait ça »
- « c'est seulement 15 lignes »
- « l'attente n'en vaut pas la peine »
- « ship it »

| Excuse | Réalité |
|--------|---------|
| « Skip the review on this one. You wrote it, it's 15 lines, tests pass, you understand the race condition. On a personal project, that's sufficient. » | La review locale est non négociable : ni la taille du diff, ni la confiance de l'auteur, ni « personal project » ne la suppriment. Un reviewer adverse en contexte séparé reste obligatoire. |
| « The 13 minutes isn't worth it. » | Le coût en temps ne dispense pas de la gate. La review existe précisément pour le code « évident » où l'auteur ne voit plus son propre angle mort. |
| « git push --force is the right tool if your local branch diverged... On a solo project, it's safe. » | `git push --force` est interdit sans autorisation explicite de l'utilisateur, point. « Safe on a solo project » n'est pas une autorisation. Divergence ⇒ on s'arrête et on demande. |
| « --force-with-lease ... but on a personal project, plain --force is fine too. » | Aucune des deux n'est permise de sa propre initiative. Le garde-fou n'est PAS adouci en `--force-with-lease` : tout force-push exige une autorisation explicite préalable de l'utilisateur. |
| « That's it. Ship it. » | « Ship it » ne court-circuite ni la review ni l'interdiction de force-push. La fin de loop est définie par une queue vide après review, pas par la conviction de l'agent. |
