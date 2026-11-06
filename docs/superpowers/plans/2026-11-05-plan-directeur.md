# Plan directeur Balados — novembre 2026

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.
>
> Version de référence éditable : https://claude.ai/artifact/2ozSDUAisi7wP8K2rw3Cr1 (Claude Docs). Ce fichier en est la copie dans le dépôt ; tenir les deux à jour.

**Goal:** Fermer les dettes connues (sécurité, dépendances, hygiène), puis rouvrir le développement produit sur #262 et #263.

**Architecture:** Chaque phase se découpe en tâches exécutables par le skill `development-workflow` : une issue GitHub par tâche, branche `feature/issue-N-slug`, TDD, revue locale adversariale, merge sans relecture humaine. Les tâches donnent fichiers, signatures et tests exacts ; l'exécutant écrit le corps.

**Tech Stack:** balados.app : React 19, Vite 7, TypeScript 5.9, Vitest 4, Dexie 4, Workbox 7. balados.sync : Elixir umbrella (core, projections, web, jobs), Phoenix 1.7.21, LiveView 0.20.17, Commanded 1.4, EventStore 1.4, Hammer 6, Cachex 3.

**Spec:** état des lieux du 2026-11-05 (deux dépôts verts : app 429 tests, sync 890 tests ; à jour avec origin ; aucune PR ouverte ; dormants depuis le 2026-07-02). Sources : `balados.sync/TODOS.md` (spec proxy hardening), `balados.app/docs/ROADMAP.md` §4.6, issues podCloud/balados.sync#262 et #263, sorties de `npm audit`, `npm outdated`, `mix hex.outdated`.

## Global Constraints

- Jamais de code directement sur `main` : branche `feature/issue-<n>-<slug>`, commits conventionnels, auteur `Claude <noreply@anthropic.com>`.
- Toute PR passe la revue locale ; aucune remarque n'est ignorée (fix dans la PR ou issue de suivi, jamais de suivi de suivi).
- Claim lock : commenter l'issue ou la PR avant de commencer.
- Encodage des identifiants RSS : base64 URL-safe sans padding, décodage `lastIndexOf(",")` côté item.
- Doc et code dans le même commit : `docs/`, `CLAUDE.md`, `TODOS.md` se mettent à jour avec la feature.
- Tests verts avant chaque commit : `npm run lint && npm run build && npx vitest run` côté app, `mix format --check-formatted && mix compile --warnings-as-errors && mix test --exclude e2e` côté sync (ce que la CI exécute depuis la tâche 3.0 ; les tests navigateur sont tagués `:e2e`, pas `:wallaby`).

## Review Focus

1. **Refresh concurrent (1.3).** Dix 401 simultanés ne doivent déclencher qu'un refresh en vol. Test `proxyManager.test.ts` : deux `fetchWithProxy` concurrents sur 401, `tryRefresh` appelé une fois, les deux rejouées avec le nouveau token.
2. **Quota partagé entre appareils (1.2).** Rate limit par `user_id` : 60 requêtes avec `device_id` A puis une avec `device_id` B du même user donnent 429 avec `retry-after`. Si trop bas, passer `limit` à 120 dans la même PR.
3. **429 du proxy côté app (1.3).** Sur 429, `fetchWithProxy` lève `SyncProxyRateLimitError` portant `retryAfterSeconds`, sans appel aux proxies publics.
4. **Service Worker et token rafraîchi (1.3).** Après mise à jour de `syncToken` en IndexedDB, la requête suivante du SW porte le nouveau Bearer.
5. **Show notes après DOMPurify 3.4 (1.1).** Une description avec `<a href target="_blank" rel="noopener">` et `<script>` rend le lien avec ses attributs et aucun script.

---

## Phase 1 — Court terme, sans débat

Trois correctifs de sécurité indépendants, une PR chacun. Ordre de merge : 1.1 n'importe quand ; 1.3 avant 1.2 si possible, pour que les clients déployés gèrent le 401 le jour où le serveur l'exige.

### Task 1.1: Bump DOMPurify (balados.app)

`npm audit` remonte une faille high sur dompurify 3.3.1, corrigée en 3.4.17. Seul `src/components/player/EpisodePlayer.tsx` l'utilise.

**Files:** Modify `package.json`, `package-lock.json`. Test `src/components/player/EpisodePlayer.test.tsx`.

- [ ] Créer l'issue `fix: bump dompurify to 3.4.17 (npm audit high)`, claim lock, branche `feature/issue-N-bump-dompurify`.
- [ ] Écrire le test Review Focus 5 (lien `target="_blank"` conservé, `<script>` retiré) ; `npx vitest run src/components/player` : attendu vert déjà (régression protégée).
- [ ] `npm install dompurify@^3.4.17` ; `grep '"dompurify"' package.json` affiche `^3.4.17`.
- [ ] `npm audit --omit=dev` : attendu `found 0 vulnerabilities`.
- [ ] `npm run build && npx vitest run` : attendu build OK, 430 tests verts.
- [ ] Commit `fix(deps): bump dompurify to 3.4.17 for GHSA advisories`, PR, revue locale, merge.

### Task 1.2: Fermer l'open proxy RSS, Phase A (balados.sync)

Le scope `/api/v1/rss` passe par le pipeline `:rss_api` (un simple `accepts`). Le proxy est public et ignore le JWT que l'app envoie déjà. On enforce JWT et rate limit par utilisateur au niveau du contrôleur, sur le modèle de `LikeController`.

**Files:**
- Modify `apps/balados_sync_web/lib/balados_sync_web/controllers/rss_proxy_controller.ex` (plugs en tête de module)
- Modify `apps/balados_sync_web/test/balados_sync_web/controllers/rss_proxy_controller_test.exs`
- Modify `TODOS.md` (cocher Phase A), `docs/FEATURES.md`, `docs/technical/AUTH_SYSTEM.md`
- Ne pas toucher : `RssCache`, `UrlValidator`, décodage base64, header `cache-control`.

**Interfaces:**
- Consumes `BaladosSyncWeb.Plugs.JWTAuth` (`scopes_any`), `BaladosSyncWeb.Plugs.RateLimiter` (`limit`, `window_ms`, `key: :user_id`, `namespace`), `JwtTestHelper.authenticate_conn(conn, user_id, scopes: [...])`.
- Produces : `GET /api/v1/rss/proxy/*` répond 401 sans JWT, 403 sans scope couvrant, 429 au-delà de 120 req/min/user (décision d'exécution : 120 plutôt que 60, quota partagé entre appareils).

- [ ] Issue `security: enforce JWT and per-user rate limit on RSS proxy (TODOS Phase A)` avec la section Phase A de `TODOS.md`. Claim lock, branche `feature/issue-N-rss-proxy-jwt`.
- [ ] Vérifier dans `balados.app/src` quels scopes l'app demande à l'autorisation. Si aucun ne couvre le proxy, ajouter le scope côté app dans la tâche 1.3 avant de merger celle-ci.
- [ ] Tests qui échouent, nouveau `describe "authentication"` :

```elixir
test "returns 401 without JWT", %{conn: conn} do
  conn = get(conn, "/api/v1/rss/proxy/#{encode_feed("https://example.com/feed.xml")}")
  assert json_response(conn, 401)
end

test "returns 401 with JWT lacking a sync or subscriptions scope", %{conn: conn} do
  conn =
    conn
    |> JwtTestHelper.authenticate_conn(user_id, scopes: ["user.likes.read"])
    |> get("/api/v1/rss/proxy/#{encode_feed("https://example.com/feed.xml")}")
  assert json_response(conn, 401)
end

test "proceeds past auth with user.sync scope", %{conn: conn} do
  conn =
    conn
    |> JwtTestHelper.authenticate_conn(user_id, scopes: ["user.sync"])
    |> get("/api/v1/rss/proxy/#{encode_feed("https://example.com/nonexistent-feed.xml")}")
  assert json_response(conn, 502)["error"] == "BAD_GATEWAY"
end

test "returns 429 after 60 requests in a minute", %{conn: conn} do
  authed = JwtTestHelper.authenticate_conn(conn, user_id, scopes: ["user.sync"])
  path = "/api/v1/rss/proxy/#{encode_feed("https://example.com/nonexistent-feed.xml")}"
  for _ <- 1..60, do: get(authed, path)
  conn = get(authed, path)
  assert json_response(conn, 429)
  assert get_resp_header(conn, "retry-after") != []
end

test "quota is shared across devices of the same user", %{conn: conn} do
  # 60 requêtes avec device_id A, puis une avec device_id B du même user_id -> 429
end
```

- [ ] `mix test apps/balados_sync_web/test/balados_sync_web/controllers/rss_proxy_controller_test.exs` : attendu nouveaux tests en échec.
- [ ] Ajouter en tête de `RssProxyController`, après `use BaladosSyncWeb, :controller` :

```elixir
plug BaladosSyncWeb.Plugs.JWTAuth, scopes_any: ["user.sync", "user.subscriptions.read"]
plug BaladosSyncWeb.Plugs.RateLimiter,
  limit: 60, window_ms: 60_000, key: :user_id, namespace: "rss_proxy"
```

- [ ] Passer les tests existants du fichier par `authenticate_conn(user_id, scopes: ["user.sync"])` dans le `setup`.
- [ ] Fichier de test vert, puis `mix test --exclude e2e` : 0 échec.
- [ ] Cocher Phase A dans `TODOS.md` ; documenter JWT requis (scopes `user.sync` ou `user.subscriptions.read`) et quota 60/min/user dans `docs/FEATURES.md` et `docs/technical/AUTH_SYSTEM.md`.
- [ ] Commit `fix(security): require JWT and rate limit RSS proxy (Closes #N)`, PR, revue locale, merge.

### Task 1.3: Gestion du 401 du proxy serveur (balados.app)

`fetchWithProxy` (`src/services/rss/proxyManager.ts`) traite un 401 du serveur comme tout échec et retombe en silence sur les proxies publics. Après 1.2, un token expiré ferait fuir tous les flux vers AllOrigins. On distingue 401 (refresh via `SyncClient`, sinon erreur typée affichée « reconnexion requise », jamais de fallback public) et 429 (erreur typée avec `retryAfterSeconds`, pas de fallback public).

**Files:**
- Modify `src/services/rss/proxyManager.ts`, `src/services/rss/proxyManager.test.ts`
- Modify `src/services/sync/client.ts` (point d'entrée public de refresh, dédupliqué)
- Modify `src/hooks/useSync.ts` ou le composant d'état de sync (surfacer l'erreur)
- Modify `src/workers/sw.ts` + ses tests (relire le token après refresh)
- Modify `docs/ROADMAP.md` §4.6 (cocher Phase A), `docs/SYNC.md`

**Interfaces:**
- Produces dans `proxyManager.ts` : `export class SyncProxyAuthError extends Error { readonly status = 401 }` et `export class SyncProxyRateLimitError extends Error { readonly status = 429; readonly retryAfterSeconds: number }`.
- Produces dans `client.ts` : `async tryRefresh(): Promise<boolean>` (wrapper public de `refreshAccessToken`, une seule promesse en vol partagée entre appelants concurrents).
- Contrat : sur 401, `fetchWithProxy` appelle `tryRefresh()` une fois ; succès → rejoue la requête serveur avec le nouveau token (relu via `getSettings()`) ; échec → lève `SyncProxyAuthError` sans proxies publics. Sur 429 → `SyncProxyRateLimitError`. Autres non-ok et erreurs réseau : cascade actuelle vers les proxies publics.

- [ ] Issue `fix: handle 401/429 from sync server RSS proxy (refresh or re-auth, no silent public fallback)`. Claim lock, branche `feature/issue-N-proxy-401`.
- [ ] Tests qui échouent dans `proxyManager.test.ts`, avec les mocks `fetch` déjà en place :

```ts
it("throws SyncProxyAuthError on 401 and does not call public proxies", async () => {
  await expect(fetchWithProxy(FEED_URL)).rejects.toBeInstanceOf(SyncProxyAuthError);
  expect(fetchMock).not.toHaveBeenCalledWith(expect.stringContaining("allorigins"), expect.anything());
});

it("retries sync proxy once with the refreshed token after a 401", async () => {
  const result = await fetchWithProxy(FEED_URL);
  expect(result.proxyUsed).toBe("Sync Server");
  expect(fetchMock.mock.calls.at(-1)?.[1]?.headers).toMatchObject({ Authorization: "Bearer new-token" });
});

it("dedupes concurrent refreshes", async () => {
  await Promise.all([fetchWithProxy(FEED_A), fetchWithProxy(FEED_B)]);
  expect(tryRefreshMock).toHaveBeenCalledTimes(1);
});

it("throws SyncProxyRateLimitError with retryAfterSeconds on 429", async () => {
  await expect(fetchWithProxy(FEED_URL)).rejects.toMatchObject({ status: 429, retryAfterSeconds: 30 });
});

it("still falls back to public proxies on a 500 from the sync proxy", async () => {
  expect((await fetchWithProxy(FEED_URL)).proxyUsed).toBe("AllOrigins");
});
```

- [ ] `npx vitest run src/services/rss/proxyManager.test.ts` : attendu nouveaux tests en échec.
- [ ] Implémenter les deux classes d'erreur et les branches 401/429 ; `tryRefresh()` dans `SyncClient` avec promesse partagée.
- [ ] Test SW (Review Focus 4) : après mise à jour de `syncToken` en base, la requête suivante du SW porte le nouveau Bearer. Implémenter.
- [ ] Dans la couche qui appelle `fetchWithProxy` (rafraîchissement des abonnements), attraper `SyncProxyAuthError` → état sync `error` avec message « Session expirée, reconnectez le serveur de sync » via l'affichage existant de `SyncStatusIcon` ; attraper `SyncProxyRateLimitError` → replanifier après `retryAfterSeconds`.
- [ ] `npm run build && npx vitest run` : tout vert.
- [ ] Cocher Phase A de `docs/ROADMAP.md` §4.6 ; décrire 401/429 dans `docs/SYNC.md`.
- [ ] Commit `fix(sync): surface 401/429 from server RSS proxy instead of silent public fallback (Closes #N)`, PR, revue locale, merge.

---

## Phase 2 — Hygiène des dépôts et docs

Sans PR de code. Parallélisable avec la Phase 1 par un agent sur le dépôt racine, jamais pendant qu'un autre agent travaille dans le même sous-module.

### Task 2.1: Nettoyage git

- [x] balados.app : pour chacun des 7 worktrees `worktree-agent-*` (commit 0393199), `git -C <chemin> status --short` : attendu vide.
- [x] `git worktree remove --force <chemin>` puis `git branch -D worktree-agent-<id>` pour chacun ; `git worktree list` n'affiche plus que main.
- [x] balados.app : `git branch -d` sur `feature/issue-58-migrate-to-biome`, `feature/issue-60-eslint-biome-coverage-doc`, `feature/issue-62-listening-history` (si refus, `git log main..<branche>` avant tout `-D`).
- [x] balados.app : supprimer `origin/feature/issue-62-listening-history` (mergée par PR #67).
- [x] balados.sync : supprimer la branche locale `feature/likes-followups-256-257-260` (PR #261).
- [x] balados.sync : pour chacune des 11 branches distantes hors main, `git log origin/main..origin/<branche>` ; si vide, `git push origin --delete <branche>` ; sinon lister les commits dans une issue de suivi.
- [x] Supprimer `balados.sync/erl_crash.dump` (13 Mo, février 2026, gitignoré).
- [x] `git branch -a` dans les deux sous-modules : main plus les seules branches non mergées.

### Task 2.2: Rafraîchir les docs racine

**Files:** `CLAUDE.md` (racine, Current Work), `docs/INTEGRATION_STATUS.md`, `docs/ROADMAP.md`.

- [x] `docs/ROADMAP.md` : `Last updated` du jour ; Phase « Sécurité et dette » reprenant 1.1 à 1.3 et la Phase 3 avec numéros d'issue ; #262 et #263 dans discovery.
- [x] `docs/INTEGRATION_STATUS.md` : proxy RSS « JWT requis depuis PR #265 » ; ligne « Open proxy fermé » dans l'historique.
- [x] `CLAUDE.md` racine : remplacer « Open Work » par l'état réel (lien vers ce plan, issues ouvertes dont #262 et #263) ; remplacer la date du 2 juillet.
- [x] Ajout (décision Pof 2026-11-05) : moderniser la skill `development-workflow` — plan directeur comme source de travail, worktrees de review détachés avec leurs propres dépendances, re-review, follow-ups et tests flaky, synchronisation du plan après merge ; `workflow-stack.md` des deux sous-modules enrichis des commandes exactes et des pièges connus (#272, #269, #266, #76) ; section workflow du `CLAUDE.md` de balados.sync condensée (7 phases obsolètes → renvoi à la skill), encodage et endpoints likes corrigés dans celui de balados.app.
- [x] Commit direct sur main (docs uniquement) : `docs: refresh root status docs after security fixes and link master plan`.
- [x] Tenir ce fichier à jour à chaque phase terminée (miroir du Claude Doc).

---

## Phase 3 — Mises à jour de dépendances

Cinq majeures bloquées côté backend (Phoenix 1.8, LiveView 1.2, Hammer 7, gettext 1.0, httpoison 3), une dizaine côté frontend. Aucun dépôt n'exécute ses tests en CI (seul `claude.yml`). Filet d'abord, puis une majeure par PR, mineures groupées.

### Task 3.0: Filet de sécurité : tests en CI et versions pinnées

**Files:** Create `balados.sync/.github/workflows/test.yml`, `balados.sync/.tool-versions`, `balados.app/.github/workflows/test.yml`, `balados.app/.nvmrc`.

- [x] balados.sync : `.tool-versions` avec `elixir 1.20.1-otp-28` et `erlang 28.5.0.2`. OTP 28 n'a pas bloqué ; Elixir 1.20 a révélé 17 warnings (`--warnings-as-errors`) et 3 diffs de formateur, corrigés dans la PR sans changement de comportement (issue #273, PR #275).
- [x] balados.sync : `test.yml` : postgres 16, `erlef/setup-beam` (`.tool-versions`, strict), cache `deps`/`_build` (restore/save séparés), `mix deps.get`, `mix format --check-formatted`, `mix compile --warnings-as-errors`, `mix db.create`, `mix db.init`, `mix test --exclude e2e`, `MIX_ENV=test` global. L'EventStore a reçu une config de test (`balados_sync_test`) : sans elle `db.init` ciblait la base dev. Vert sur la PR (942 tests).
- [x] balados.app : `.nvmrc` et `.tool-versions` = `24.22.2`, `engines.node >= 24` ; `test.yml` : `npm ci`, `npm run lint`, `npm run build`, `npx vitest run`, les trois contrôles tournent même si l'un échoue, concurrence par commit sur `main`. `asyncUtilTimeout` 4 s / `testTimeout` 10 s pour ne pas rendre #76 bloquant. Vert sur la PR (issue #78, PR #79, 491 tests).
- [x] Workflows documentés dans les `CLAUDE.md` (section CI, règle de merge : le check doit être vert sur le dernier commit, GitHub ne l'impose pas encore → décision ruleset dans app #80). Suivis : app #80, #81 ; sync #274 (flaky RateLimiterTest), #276 (HTTP sortant dans les tests), #277 (500 latent sur item sans enclosure), #278 (warnings des fichiers de test).

### Task 3.1: Backend, mineures sans risque

- [x] `mix deps.update` de tous les candidats « Update possible » (40 entrées du lock, aucune contrainte modifiée). joken_jwks reste en 1.7 : 1.8 exige hackney 4, bloqué par httpoison 2 (débloqué en 3.3). Entrée `:mail` obsolète retirée du lock ; `elixir: "~> 1.16"` dans les quatre apps (exigé par joken 2.7, swoosh 1.29, eventstore 1.4.10 ; la suite de tests demande 1.17 via wallaby 0.31).
- [x] `mix compile --warnings-as-errors` puis `mix test --exclude e2e` : 942 tests, 0 échec (local sur base vierge et CI).
- [x] `mix test --include e2e` : 3/7, identique sur `main` avec les anciennes dépendances → préexistant, issue #282 (la connexion ne trouve pas le lien « Log out », assertion dépendante de la locale).
- [x] PR #283 mergée le 5 nov. (issue #281, 2 rounds de revue) avec notes de changelog : la plage contient une quinzaine de correctifs de sécurité (bandit, plug, mint, phoenix, phoenix_html).

### Task 3.2: Backend, majeures hors Phoenix (une PR chacune)

| Lib | De | Vers | Usage | Point d'attention |
| --- | --- | --- | --- | --- |
| gettext | 0.26 | 1.0 | `gettext.ex` + `use Gettext, backend:` | Déjà la forme `backend:`, migration quasi vide |
| hammer | 6.2 | 7.5 | `plugs/rate_limiter.ex` + 1 fichier | Backend ETS déclaré comme module, `check_rate/3` → `hit/3` |
| httpoison | 2.3 | 3.0 | 3 fichiers (RssCache, UrlValidator, ownership) | Candidat au remplacement par `Req`/`Finch` ; décider dans la PR |
| timex | 3.7 | 4.0 | 1 fichier | Vérifier si `DateTime`/`Calendar` natifs suffisent, supprimer la dep |
| cachex | 3.6 | 4.1 | 2 fichiers (RssCache) | Options renommées (`expiration:` → `expire:`) |
| dns_cluster | 0.1 | 0.3 | config release | Suivre la doc Phoenix 1.8 |

- [x] Hammer 7 : PR #289 (issue #286), module `BaladosSyncWeb.RateLimit`, `Retry-After` exact, limiteur démarré avant l'endpoint, quota `auth_refresh` 100/h réellement appliqué ; test `limit: 1` → 200 puis 429 ajouté ; flaky #266 corrigé.
- [x] timex : supprimé (PR #287, issue #285), `BaladosSyncWeb.Rfc1123` natif. La revue de CI a trouvé un vrai bug : sans tzdata, Quantum sortait les trois tâches planifiées du planning (`timezone: "UTC"`) → `:utc` + `SchedulerConfigTest`. Suivis #290, #295.
- [x] cachex 4 : PR #291 (issue #288). Sans migration, `ttl:` et `limit:` étaient ignorés en silence (plus d'expiration ni de borne) ; `expire:` + hook `Cachex.Limit.Scheduled`, test de configuration.
- [x] gettext 1.0 : PR #296 (issue #284), aucune rupture, débloqué par le retrait de timex. Issue #299 ouverte en marge (locale des LiveView).
- [x] dns_cluster 0.3 : PR #294 (issue #293), mergée le 2026-11-06 après 4 tours. Corrige aussi une clé de config qui empêchait `DNS_CLUSTER_QUERY` d'être lue (valeur vide ou blanche = pas de cluster, valeur rognée).
- [x] httpoison : remplacé par Req derrière `BaladosSyncCore.SafeHttp` (PR #298, issue #292), mergée le 2026-11-06 après 7 tours. Trouvaille de sécurité : les redirections n'étaient pas revalidées (SSRF), et le validateur laissait passer `0.0.0.0`, `[::ffff:169.254.169.254]`, `localhost.`… ; tout est corrigé. Les tours 3 à 6 ont aussi fermé des fuites d'identifiants et de jetons de flux privés dans les journaux. Suivis #297 (résolution DNS), #300 (taille de corps, délai global, hygiène des journaux restante), #301 (joken_jwks sans hackney en prod). joken_jwks 1.8 n'est plus bloqué que par le lock (Wallaby garde httpoison 2).

**Tâche 3.2 terminée le 2026-11-06** : six PR mergées (#287, #289, #291, #294, #296, #298).

### Task 3.3: Phoenix 1.8 et LiveView 1.x

LiveView 0.20 → 1.2 change `live_patch`/`live_redirect` (8 occurrences), `phx-feedback-for`, `core_components` et les formulaires. 5 LiveViews concernées, plus les 37 tests LiveView déjà cassés une fois (#246).

- [x] Issue `chore(deps): upgrade to Phoenix 1.8 and LiveView 1.2`, branche ; générer un projet Phoenix 1.8 vierge dans le scratchpad pour diff `core_components.ex`, `router.ex`, `endpoint.ex`, `app.js`, config esbuild/tailwind.
- [x] Étape 1, LiveView 1.0 seul (Phoenix 1.7 l'accepte) : `~> 1.0`, `<.link patch/navigate>` à la place de `live_patch`/`live_redirect`, `used_input?/1` à la place de `phx-feedback-for`. Tests verts, commit.
- [x] Étape 2, Phoenix 1.8 : `~> 1.8`, `phoenix_live_dashboard ~> 0.9`, aligner `core_components.ex` sur le diff en gardant les composants maison, layout racine (`Layouts.app` composant fonction). `mix test --exclude e2e` et `--include e2e` verts, commit.
- [x] Étape 3, LiveView 1.2 : `~> 1.2`, changelog, `mix compile --warnings-as-errors` propre.
- [x] `docs/technical/ARCHITECTURE.md` (versions) et `CLAUDE.md` du sous-module. PR, revue locale (demander une revue des templates), merge.

Terminée le 2026-11-06 (PR sync #303, issue #302, 3 tours de revue). Phoenix 1.8.16, LiveView 1.2.13, LiveDashboard 0.9.1, floki remplacé par lazy_html. Écarts au plan : plus aucun `live_patch`/`live_redirect` à migrer ; les layouts explicites et le `core_components` daisyUI du générateur 1.8 ne sont pas adoptés (optionnels, toucheraient toutes les pages) ; e2e à parité avec main (3/7, #282). La revue a trouvé une régression introduite par la montée : l'`id` exigé sur le formulaire de filtres activait la restauration de formulaire, qui ramenait en page 1 après reconnexion (`phx-auto-recover="ignore"`). Suivis #304, #306 ; trouvé en passant #305.

### Task 3.4: Frontend, mineures groupées

- [x] `npm update` (24 paquets dans leurs plages, `package.json` inchangé ; React 19.3, Vite 7.3, Vitest 4.1, Tailwind 4.3, Dexie 4.4, TanStack Query 5.106, vite-plugin-pwa 1.3…), lint 0 nouveau, build OK, 493 tests. `npm audit` complet 27 → 6 (tout sous workbox-build, attend vite-plugin-pwa 2 en 3.5).
- [x] Vérification navigateur par Playwright (profil vierge, SW activé sur le build preview, page Historique, abonnement et lecture d'un épisode). Elle a révélé un bug préexistant : sur profil vide, le seeding du flux par défaut ouvrait une transaction d'écriture dans un `liveQuery` et faisait planter la Bibliothèque (issue #83, corrigé + test de régression dans la même PR). PR #84 mergée le 5 nov. (issue #82, 3 rounds) ; suivi #85 (`includeAssets`, précache en double, course migration localStorage/seeding). Majeures restantes pour 3.5 : ajouter `@types/node` 26 et `react-i18next` 17 à la liste.

### Task 3.5: Frontend, majeures (une PR chacune, dans cet ordre)

| Lib | De | Vers | Point d'attention |
| --- | --- | --- | --- |
| @vitejs/plugin-react | 5 | 6 | Impact limité à `vite.config.ts` |
| vite | 7 | 8 | Vérifier `vite-plugin-pwa` compatible avant ; `injectManifest` sur `src/workers/sw.ts` |
| vite-plugin-pwa | 1 | 2 | `strategies: injectManifest`, `srcDir` ; retester le background sync |
| vitest + coverage-v8 | 4 | 5 | API des mocks ; suite complète |
| i18next + react-i18next | 25/16 | 26/17 | 31 fichiers ; `compatibilityJSON`, pluriels |
| lucide-react | 0.562 | 1.x | 20 fichiers ; icônes renommées, la compilation les trouve |
| marked | 17 | 18 | `EpisodePlayer.tsx` seul ; retester les show notes avec DOMPurify |
| jsdom, jest-dom | 27/6 | 29/7 | Tests uniquement |
| typescript | 5.9 | 7.0 | En dernier, seul : compilateur Go, `tsc -b` et `verbatimModuleSyntax` à revalider ; si Biome ou Vite ne suivent pas, rester en 5.9 et noter la date de réessai |

- [ ] Pour chaque ligne : issue, branche, `npm install <lib>@<majeure>`, `npm run lint && npm run build && npx vitest run` vert, vérification navigateur quand indiqué, PR, merge.
- [ ] Après la dernière : `npm outdated` ne liste plus que des mineures ; `npm audit --omit=dev` à 0 ; section Stack du `CLAUDE.md` du sous-module à jour.

---

## Phase 4 — Applications tierces et specs produit

#262 et #263 sont des captures Discord, pas des specs. Chacune passe par `superpowers:brainstorming` avec Pof, produit un design doc, puis un plan séparé via `superpowers:writing-plans`. Ce plan fixe les questions à trancher et les livrables, pas les tâches de code.

Décision du 2026-11-05 : la phase s'ouvre par le chantier des applications tierces, car PodCloud sera la première application à importer des données dans un compte Balados, ce qui exige que le contrat client soit terminé.

### Task 4.0: Terminer le contrat d'application tierce

Le serveur attend `/authorize?token=<JWT d'app portant métadonnées et clé publique>` et des requêtes signées RS256 par l'app ; la popup de balados.app appelle `/authorize?redirect_uri=…&response_type=token`, ne persiste aucun matériel de renouvellement, et seul le token manuel fonctionne (#73). Avant qu'une application externe puisse s'y brancher, ce contrat doit être unique, documenté et implémenté des deux côtés.

**Livrables :** `balados.sync/docs/technical/AUTH_SYSTEM.md` comme contrat de référence, guide « intégrer une application tierce » dans `balados.sync/docs/api/`, balados.app conforme (#73 fermée), kit de test (clés de dev, script générant un JWT d'app valide).

Questions à trancher :

1. **Modèle de jeton** : jeton auto-signé par l'app (clé privée locale, `exp` court renouvelé via `POST /auth/refresh`, modèle documenté côté serveur) ou jeton émis par le serveur. Recommandation : garder le modèle serveur, faire converger l'app.
2. **Stockage côté app** : clé privée dans IndexedDB (WebCrypto non extractible si possible), `fromSettings()` la recharge, le chemin 401 → refresh → rejeu de `proxyManager` devient effectif.
3. **Scopes** : liste publique et stable ; scopes écriture seule utilisables par un importateur (`user.subscriptions.write`, `user.likes.write`, `user.playlists.write`, `user.plays.write`) sans `user.sync`.
4. **Provenance** : `app_id` déjà dans les métadonnées de commande ; décider s'il est projeté (« importé depuis PodCloud », révocation propre).
5. **Page d'autorisation** : libellés par scope, mention « cette application ne pourra rien lire » quand seuls des scopes d'écriture sont demandés.

- [ ] Brainstorming avec Pof sur 1 à 3, design doc court.
- [ ] Implémentation backend (doc, page d'autorisation, kit de test) puis frontend (#73) via le `development-workflow`.
- [ ] Critère de sortie : une application de démonstration externe obtient une autorisation, pousse un abonnement, et un 401 sur le proxy est récupéré sans action utilisateur.

### Task 4.1: Import PodCloud à sens unique

PodCloud s'enregistre comme application tierce (4.0) et pousse une fois, pour un utilisateur qui l'accepte, ses abonnements, likes, playlists et positions d'écoute dans le compte Balados. Il ne lit rien et ne modifie plus son propre état : le compte PodCloud est figé pour la partie écoute, Balados reprend la main.

**Livrables :** design doc côté balados.sync (ce qui manque à l'API pour un import idempotent), spec du job d'export côté PodCloud, commentaires croisés sur les issues.

Questions à trancher :

1. **Endpoints** : `POST /sync` en une passe avec horodatages d'origine (recommandé : profite de la résolution de conflit) ou endpoints unitaires.
2. **Idempotence** : rejouer l'import ne duplique rien (à vérifier pour les playlists).
3. **Positions d'écoute** : `POST /play` avec `updated_at` d'origine si PodCloud les connaît.
4. **Rapport** : envoyés / ignorés par conflit / en erreur, affiché côté PodCloud.
5. **Identité des flux** : conserver l'identifiant PodCloud du podcast pour suivre un flux dont l'URL change (lien avec l'OPML étendu, Phase 5).

- [ ] Brainstorming, design doc, plan ; dépend de 4.0.

### Task 4.2: Spec des tags communautaires (#262)

**Livrables :** `balados.sync/docs/superpowers/specs/2026-MM-DD-community-tags-design.md`, puis `docs/superpowers/plans/…-community-tags.md`, commentaire sur #262.

Questions à trancher, dans l'ordre :

1. **Agrégat** : Subscription existant (piggyback `Subscribe`) ou nouvel agrégat `PodcastTagging` par user/feed ? Recommandation : agrégat dédié, événement `TagsChanged {user_id, feed, added, removed, at}`.
2. **Vocabulaire partagé** : table `tags` globale, slug normalisé (minuscules, sans accents, tirets) ; qui crée un tag, limite par jour.
3. **Seuils** : 10 % des taggeurs ET > 25 taggeurs. Projections `podcast_tag_counts` (feed, tag, count) et `podcast_tagger_counts` (feed, total) ; visibilité calculée à la lecture.
4. **Top 5 / top 20** : cache Cachex 5 min, `GET /api/v1/public/feed/:feed/tags` et `GET /api/v1/public/tags/top`.
5. **Modération** : commande admin `HideTag`, `hidden_at` sur `tags`, exclusion de toutes les lectures publiques ; UI admin LiveView existante.
6. **Sync app** : `TagSync {rss_source_feed, tags[], updated_at}` dans `POST /sync`, LWW sur `updated_at` pour la liste entière.
7. **UI app** : fiche podcast, message explicatif, retrait de ses propres tags.

- [ ] Brainstorming avec Pof, une question à la fois.
- [ ] Design doc relu par Pof.
- [ ] Plan d'implémentation : backend (agrégat, projections, API, admin) puis frontend (sync, UI).
- [ ] Fermer #158 comme remplacée par #262, ou la garder pour les hashtags libres.

### Task 4.3: Spec des réactions d'épisodes (#263)

**Livrables :** même trio (design doc, plan, commentaire sur #263).

1. **Agrégat** : `EpisodeReaction` par user/épisode, événement `ReactionsSet {user_id, feed, item, reactions[], at}` (liste complète, max 5). Recommandation : liste complète.
2. **Set de réactions** : set par défaut (5 emojis à choisir avec Pof), set custom par flux revendiqué via `PodcastOwnershipClaim`, commande `SetFeedReactions {feed, reactions[]}` réservée au propriétaire vérifié. Sort des réactions existantes quand le set change.
3. **Anonymat** : projections `episode_reaction_counts` (feed, item, reaction, count) et `episode_reaction_combos` (feed, item, combo_bitmask 1..31, count).
4. **Pentagone** : calcul côté app à partir des 31 combos ; normalisation par épisode ; `GET /api/v1/public/episode/:item/reactions`.
5. **Modération** : `HideReaction` global ou par flux, même forme que `HideTag` (4.2) ; mutualiser l'admin.
6. **Sync app** : `ReactionSync {rss_source_feed, rss_source_item, reactions[], updated_at}`.
7. **Namespace RSS** `balados:reactions` : hors périmètre, annexe du design doc.

- [ ] Brainstorming, design doc, plan, comme en 4.2.
- [ ] Faire 4.2 avant 4.3 (modération et projections publiques posées une fois).

### Task 4.4: Exécution

- [ ] Issues d'implémentation depuis les plans (une par tâche), étiquetées, passées dans le `development-workflow`.
- [ ] Mettre à jour ce document : lier design docs et plans, cocher.

---

## Phase 5 — Backlog non planifié

Rien d'engagé. Repris un par un par un brainstorming dédié après les Phases 1 à 4.

| Item | Dépôt | Nature | Prérequis | Note |
| --- | --- | --- | --- | --- |
| Proxy d'images HMAC, Phase B | sync + app | Privacy | 1.2 livrée | Design figé dans `TODOS.md` : `POST /api/v1/rss/sign`, URLs `?sig=` stables, audio jamais proxifié. App : regex cache images du SW, modal trackers au play |
| #153 Playlists collaboratives | sync | Social | Aucun | Multi-contributeurs sur l'agrégat Playlist |
| #155 Suivre d'autres utilisateurs | sync | Social | Modèle de confidentialité | Graphe social, timeline des suivis |
| #158 Hashtags et topics | sync | Discovery | 4.1 | Probablement absorbée par #262 |
| #201 Recommandations MinHash | sync | Discovery | Volume d'utilisateurs | Job périodique dans `balados_sync_jobs` |
| Import OPML, recherche iTunes/PodcastIndex | app | Discovery | Aucun | `docs/ROADMAP.md` Phase 5 |
| UI de gestion des proxies, logs de sync | app | Confort | Aucun | `docs/ROADMAP.md` Phases 1 et 4 |
| OPML étendu import/export (position, lu, liké, playlists, date d'abonnement) | app + sync | Portabilité | Aucun | Idée Pof du 2026-11-05 ; namespace `balados:` sur les `<outline>`, mêmes règles de conflit que le sync |

---

## Suivi d'avancement

| Tâche | Dépôt | Issue | PR | Statut |
| --- | --- | --- | --- | --- |
| 1.1 Bump DOMPurify | app | #68 | #69 (mergée 2026-11-05) | Terminé — suivi #71 |
| 1.2 Proxy RSS : JWT + rate limit | sync | #264 | #265 (mergée 2026-11-05) | Terminé — suivis #267 à #272 |
| 1.3 Gestion 401/429 du proxy | app | #70 | #72 (mergée 2026-11-05) | Terminé — suivis #73 à #77 |
| 2.1 Nettoyage git | racine | | | Terminé (2026-11-05) : 7 worktrees orphelins, 4 branches locales et 1 branche distante mergées supprimés, crash dump effacé |
| 2.2 Docs racine | racine | | | Terminé (2026-11-05) : ROADMAP, INTEGRATION_STATUS, CLAUDE.md racine, skill `development-workflow` modernisée, stacks et CLAUDE.md des sous-modules |
| 3.0 CI tests + versions pinnées | les deux | app #78, sync #273 | app #79, sync #275 | Terminé (2026-11-05) : Node 24.22.2, Elixir 1.20.1 / OTP 28, suites vertes en CI ; suivis #80, #81, #274, #276–#278 |
| 3.1 Backend mineures | sync | #281 | #283 | Terminé (2026-11-05) : 40 entrées du lock, Elixir ~> 1.16, 942 tests ; e2e préexistant #282 |
| 3.2 Backend majeures hors Phoenix | sync | #284, #285, #286, #288, #292, #293 | #287, #289, #291, #294, #296, #298 | Terminé |
| 3.3 Phoenix 1.8 + LiveView 1.x | sync | #302 | #303 | Terminé (2026-11-06) : Phoenix 1.8.16, LiveView 1.2.13 ; suivis #304, #306 ; #305 |
| 3.4 Frontend mineures | app | #82, #83 | #84 | Terminé (2026-11-05) : 24 paquets, bug #83 trouvé par Playwright et corrigé ; suivi #85 |
| 3.5 Frontend majeures | app | | | À faire |
| 4.0 Contrat d'application tierce | les deux | #73 | | À faire |
| 4.1 Import PodCloud à sens unique | sync + PodCloud | | | À faire |
| 4.2 Spec tags communautaires #262 | sync | #262 | | À faire |
| 4.3 Spec réactions #263 | sync | #263 | | À faire |
| 4.4 Exécution des plans 4.x | les deux | | | À faire |

Prochaine revue de ce plan : 2026-11-19.
