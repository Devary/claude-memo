---
name: project-anime-vote
description: "Current state of quar-anime-vote + ui-anime-vote — local JWT auth (Keycloak/redzone-api removed), user-management branch, prod request flow"
metadata: 
  node_type: memory
  type: project
  originSessionId: 60ff7dd0-0020-44d3-8bd4-183536fd3902
---

As of 2026-07-04, both repos are on branch `feature/user-management` (branched off `feature/voting-level`, which is included). Working trees clean.

## MAJOR CHANGE: auth is now fully local (commit 4553486)
Keycloak and redzone-api delegation were REMOVED from anime-vote. It is now a self-contained auth system:
- `AppUser` / `AppRole` entities (JPA, `user_role` join table), UUID string ids, unique username+email
- `PasswordUtil` hashing; register requires 8+ char password + confirmPassword match
- `TokenService`: signs access (1h) + refresh (7d) JWTs locally with RSA keys `privateKey.pem`/`publicKey.pem` in resources; issuer `anime-vote`; roles as `groups` claim; `token_use` claim distinguishes access/refresh
- `AuthResource`: `/auth/login`, `/auth/register`, `/auth/refresh`, `/auth/username-availability/{u}` — all `@PermitAll`
- `RedzoneAuthClient.java` still exists but is DEAD CODE (no references, no `redzone.url` in properties)
- Prod: `quarkus.http.auth.proactive=${GATEWAY_AUTH_ENABLED:false}`; `mp.jwt.verify.issuer=${JWT_ISSUER:anime-vote}`

## User-management / content-ownership feature (commits 13509f2, ff15b14)
- `ContentStatus` enum: PENDING / APPROVED / REJECTED on characters, polls, multi-polls
- Content has `ownerId`; users create content as PENDING; `isPrivate=true` content is auto-APPROVED
- `UserContentService`: per-user CRUD with ownership checks; daily limit 5 per type (429); deleting APPROVED public poll/multi-poll sets `deletePending=true` (admin must approve deletion), otherwise immediate delete
- `ApprovalService` + `AdminApprovalResource`: admin approves/rejects pending content and deletion requests
- `UserService`: profile update (email/profilePicture), admin user CRUD, role CRUD (`AdminUserResource`, `AdminRoleResource`)
- Other resources: `UserProfileResource`, `UserContentResource`, `UploadResource` (image upload), `HistoryResource` (dedupes multi-poll votes, adds char imageUrl), `ServerTimeResource`

## ui-anime-vote (Angular 21, signals, standalone; dev port 4209)
- `app.routes.ts` is EMPTY — single-page app, view switching via signals in `app.ts`
- `AuthService`: signal-based session in localStorage key `anime_auth`, auto-refresh 60s before expiry, `isAdmin` computed from roles
- Management area: anime, character, poll, multi-poll, user, role, approval, my-content management components; shared `crud-modal` + `confirm-modal` (CRUD modal pattern applied)
- Bracket auto level generation in multi-poll management; VoteHistory uses `HistoryItemDto` from API
- `environment.prod.ts`: `apiUrl: '/anime-vote'`; dev: `/api` → proxy → `localhost:5556`

## Backend config
- Port 5556; dev datasource: PostgreSQL `192.168.178.41:31432/animevote`, `drop-and-create` in dev
- Prod: Vault-backed `db.*` keys at `kv/anime-vote` (role `anime-vote`), `hibernate-generation:update`

## Infra — Traefik (NOT nginx ingress)
- IngressRoute rule: `Host(<app>.192.168.178.41.nip.io) || PathPrefix(/<deployment>)`; StripPrefix middleware
- K8s services expose port 80 → container port
- Prod flow: Browser `POST /anime-vote/auth/login` → Traefik StripPrefix → anime-vote pod:5556 → local JWT issued (no external auth calls)
- Deploy: Jenkinsfile → Harbor `192.168.178.41:30002/library` → Rundeck job → infra `deploy.sh` (Vault bootstrap, sed-template k8s manifests, rollout check)

## Wizard + dashboard round (2026-07-09, pushed)
- `shared/multi-poll-wizard/multi-poll-wizard.component.ts` consolidates admin multi-poll-management AND my-content multi-poll form into ONE 3-step stepper (Properties/Groups/Summary), `mode: 'admin'|'user'` input controls bracket support + structural-edit-while-editing. Both callers use ViewChild to read `.dirty` for the confirm-before-discard pattern (kept in parent, not the wizard)
- `shared/poll-time.ts` (unit tested): pollTimeStatus/pollRemainingLabel — used by both multi-poll tables for the "Xd Yh left"/"Finished" badge, computed from earliest start/latest end across ALL groups incl. brackets
- New `GET /admin/dashboard` (ADMIN/MODERATOR) + `DashboardManagementComponent` tab: totals, 7d trends, visibility/voting-mode breakdowns, top-5 most liked/voted/commented (polls+multipolls combined)
- Gotcha found+fixed: 3 single polls (seeded 2026-07-06) had gone missing from DataSeeder despite DatabaseSeedTest requiring >=10 — always verify seed counts actually match test expectations after seeder edits, don't trust prior session summaries
- Cypress gotcha: backend H2 persists across repeated `npx cypress run` invocations in one session — specs creating named content (e.g. multi-poll questions) MUST suffix with `Date.now().toString(36)` or duplicate-conflict 409s appear on reruns; dashboard total assertions must use `.gte()` bounds not exact counts
- Tests: backend 96, vitest 36, Cypress 47 (14 specs)

## Vote-by-group + seed pipeline (2026-07-06, pushed)
- `MultiPoll.votingByGroup` (IMMUTABLE at creation, flat/level-0 only): vote = whole group; POST vote body `{groupId}`, PUT `{newGroupId}`; one vote per poll; results per group (`groupTotal` = group votes, `myVotedGroupId`); `MultiPollVote.characterId` now NULLABLE (live PG altered + voting_by_group column added)
- Seed pipeline order: 10 Anime rows → 64 characters (slug ids, linked by series name) → 10 single polls → 20 multi-polls (10 votingByGroup PUBLIC flat + 10 by-character incl. bracket + all visibility anchors). Anchor questions unchanged — tests depend on them
- Frontend: third card mode `.choice-board` (isGroupVote before isSimple/knockout); VoteStore group choice under `${groupId}|@group` count key + `getMyGroupChoice(pollId)`; creation checkbox in both multi forms (hidden on edit)
- Cypress gotcha: anime mgmt table paginates at 10 — specs must use the search filter before asserting rows
- Tests: backend 93, vitest 30, Cypress 40

## Validation + schema round (2026-07-06, pushed)
- Comments: 6..349 chars, control chars stripped, `<`/`>` rejected (server CommentService.sanitizeAndValidate + client comment-validation.ts — keep in sync)
- Names/questions (anime, character, poll, multi-poll) capped at 254 via `ContentValidation.requireName`; maxlength=254 on all form inputs
- Multi-poll group `level` is MANDATORY (Integer @NotNull admin path, explicit check user path; user My Content groups send level:0)
- SCHEMA GOTCHA: NOT NULL boolean columns need `@ColumnDefault` or hibernate `update` fails silently on populated PG tables ("column X does not exist" at runtime). Live dev DB patched with ALTERs for comments_enabled
- z-index ladder: header 500 < vote-sheet 550 < comment drawer 600 < auth 700 < management 800 < toast 9999
- `public/logo-4k.png` = 4096×4096 transparent social-media logo

## Social layer + UI polish (2026-07-06, pushed)
- Text-only comments (≤500 chars, URL rejection mirrored client `shared/social/comment-validation.ts` + server `CommentService.LINK_PATTERN`); newest first; creator/admin/mod can pause via PUT /{kind}/{id}/comments-enabled (existing comments stay readable). Likes: ContentLike toggle per user/IP. DTOs enriched: likes/likedByMe/commentCount/commentsEnabled
- TikTok action rail `app-social-actions` (Like/Comment/Share) bottom-right on both cards; comment drawer `app-comment-drawer` right panel / mobile bottom sheet. Rail resyncs via effect on poll input (component reuse!)
- Header + auth card = 5% opaque glass with FIXED light text (backdrop is colorful in both themes — never use --rz-ink there); header needed `position:relative; z-index:100` (backdrop-filter stacking context) and overlays above it need z>100
- Language selector = flag dropdown (🇺🇸/🇫🇷/🇹🇳), LANGS in translations.ts has {code,label,flag}
- Auth modal: 72px icon-only logo, no wordmark; header icon-only too

## Branding (2026-07-05)
- Product name is **VoteScroll** (repos keep anime-vote names). Logo: `public/logo.svg` — gradient VS lightning tile, used in header + favicon. No success toasts on voting (state-only feedback); error toasts remain

## Audit + UX round (2026-07-05, pushed)
- `audit_event` table: CREATED/UPDATED/DELETED/APPROVED/REJECTED/RESTORED per entity with actor (AuditService resolves JWT itself via `Instance<JsonWebToken>` — no service signature churn) + JSON snapshot of restorable fields; `/admin/audit` (mods): recent, per-entity history, POST /{id}/restore
- Approval queue: moderators can EDIT pending items (type-aware modal, Save / Save & Approve) before approving; multi-poll admin update now also edits question/anime
- Cards show `by : username` chip (ownerUsername ?? 'admin')
- Feed navigates vertically TikTok-style: wheel (500ms cooldown), touch swipe (60px), ArrowUp/Down, vertical chevrons, direction-aware slide animation; interactive areas excluded via isInsideScrollable
- GOTCHA: Cypress `.trigger('wheel', {deltaY})` mangles options — dispatch `new win.WheelEvent(...)` via cy.window() instead
- Test counts: backend 78, vitest 24, Cypress 30

## Open content editing (2026-07-05, pushed)
- ALL users create/update animes + characters (any entity, wiki-style); simple-user changes → status PENDING until ADMIN/MODERATOR validates; moderator changes publish directly. `/user/anime` endpoints; Anime has status+ownerId; anime approvals in the queue (type ANIME)
- Duplicates: anime (name, case-insens.) / character (name+anime) → 409 with `conflictId` in error body → UI confirm modal "already exists, modify it instead?" switches form to edit. PUBLIC polls can't reuse a fighter set, public multi-polls can't reuse a question; PRIVATE may duplicate
- `content_edit_log` + `content.modification.daily-limit` (default 50, test profile 3) caps simple-user anime/char modifications/day
- Polls editable only by creator + moderators (403 otherwise)
- GOTCHA: dev PostgreSQL 192.168.178.41:31432 can be unreachable — run backend on H2 with `-Dquarkus.datasource.db-kind=h2 -Dquarkus.datasource.jdbc.url='jdbc:h2:mem:animevote;DB_CLOSE_DELAY=-1' -Dquarkus.datasource.reactive=false` for local/Cypress work
- Test counts now: backend 71, vitest 22, Cypress 24 (content-editing.cy.ts covers the duplicate-modal flow)

## Moderation model (2026-07-05, pushed)
- Content (anime/characters/polls/multi-polls) must be validated by ADMIN **or MODERATOR** before publication; those roles share all content-management + approval endpoints; Users/Roles admin stays ADMIN-only. Frontend: `AuthService.canModerate` gates the management tabs
- Guard rule: PENDING/REJECTED polls are 404 on direct fetch/vote for everyone except creator + moderators (was a leak: 200 by id!)
- Users can only compose polls from APPROVED characters or their own pending ones
- Seeded demo users: admin/admin123 (ADMIN), mod/mod12345 (MODERATOR), otaku/otaku123 (USER)

## Visibility + sharing + i18n (2026-07-05, pushed)
- `Visibility` enum on Poll/MultiPoll: PUBLIC / AUTHENTICATED / RESTRICTED (allowed user list) / PRIVATE (creator + ADMIN/MODERATOR). `VisibilityPolicy.canView` used by listings AND result/vote endpoints (hidden → 404). PRIVATE/RESTRICTED auto-approved; PUBLIC/AUTHENTICATED need approval
- CRITICAL gotcha fixed: principal name = JWT `upn` (username); ownership/audience compare against AppUser.id → resources must use `jwt.getSubject()`. Also `quarkus.http.auth.proactive=true` needed so @PermitAll endpoints see the Bearer identity (was false → all "authenticated" votes were silently IP-keyed)
- Public polls: anonymous same-IP re-vote allowed only after 20 min (PollService.PUBLIC_IP_COOLDOWN_MINUTES, applies per multi-poll group too); authenticated users strictly once
- `GET /users/directory` (authenticated) feeds the RESTRICTED audience picker (shared `app-visibility-field` in all 3 forms)
- Share links: `?p=<pollId>` deep link + share button (public content only, Web Share API/clipboard)
- i18n: custom signal-based `I18nService` (src/app/i18n/), EN default/FR/AR, header cycle toggle, AR sets dir=rtl; management screens still English-only
- Seeds: demo user otaku/otaku123; poll-members-hashira (AUTHENTICATED), poll-vip-kage (RESTRICTED→otaku), mp-members-mha (AUTHENTICATED)

## Voting rules (confirmed by user 2026-07-04)
- One vote per user per voting group; switchable while the group is open (before endDate). Backend `changeVote` + `checkPeriod` enforce it; UI keeps other candidates tappable after voting ("Tap to switch")
- Simple polls (1v1) and single-group multi-polls render as a plain PrimeNG org chart (gold Winner root); the symmetric SofaScore knockout tree is ONLY for multi-polls with >1 group
- Single-group polls whose group startDate is in the future are hidden from the public listing — filtered in the BACKEND (`MultiPollService.getAll`), per user's explicit instruction
- Management export (poll + multi-poll) downloads JPEG (canvas-rasterized SVG, 2x). ALL exports use the knockout style: 1v1 polls render fighters as side match-boxes with a single-slot gold Winner center (leader from live results); single-group multis use the org chart; multi-group multis the full symmetric tree
- Backend tests run on in-memory H2 (application-test.properties); PG profile kept for manual runs; seed-count assertions must stay lower-bound (exact counts broke Jenkins)
- ALL ids (seeded polls/multi-polls/groups too) are now random UUIDs — never reference content by slug; locate seeds by question text or structure (bracket = poll with a level-2 group). refreshBracketTournamentDates works structurally by level+groupOrder
- Test suites: backend 63 tests = 35 @QuarkusTest/RestAssured API tests + 28 Mockito unit tests (quarkus-junit5-mockito + quarkus-panache-mock deps; GOTCHA: PanacheMock intercepts Lombok static builder() too — construct entities with `new` before/instead of builders inside stubs; varargs Panache statics need `Mockito.<Object[]>any()`). Frontend: 22 vitest specs + 19 Cypress e2e tests (cypress/e2e/, `npm run e2e`, needs stack on 5556+4209; custom commands register fresh users per test to avoid IP-cooldown flakiness). Run: `./mvnw test`, `npx ng test`, `npm run e2e`

## SofaScore knockout redesign (2026-07-04, pushed to feature/user-management)
- Poll + multi-poll cards render as symmetric knockout trees (SofaScore style): outer avatars, TBD shield placeholders, connector rails, gold Final center, champion crown; tap a match → bottom vote sheet
- Backend: `winnerCharId` on MultiPollGroupDto/GroupResultDto (set when group ended; null on tie/no votes); `getAll()` resolves winners; `DataSeeder.seedVotes()` seeds demo votes from TEST-NET IPs (idempotent)
- VoteStore counts are ALSO keyed `groupId|charId` (`getGroupCount`) — flat charId map alone breaks once a char advances to a higher bracket level
- Gotcha: the app.ts carousel `@else-if` reuses MultiPollCardComponent across consecutive multi-polls → per-poll init must be an `effect()` on the poll input, not ngOnInit
- Gotcha: carousel next()/prev() skip a whole multi-poll once ANY group is voted (myVotes[pollId] set on first group vote) — brackets disappear from rotation after one QF vote
- anyComponentStyle budget warning raised to 12kB in angular.json

## Voting-level / bracket feature (merged into this branch)
- `MultiPollGroup.level` + `feederGroupIds`; winners auto-populate next level when all feeders expire; tie/no-votes → TBD "?"
- Bracket UI: levels DESCENDING (Grand Final top), per-group voting, `hasVotedInGroup()`
- Tournament poll id `mp-anime-tournament`; `DataSeeder` refreshes expired bracket dates on restart
