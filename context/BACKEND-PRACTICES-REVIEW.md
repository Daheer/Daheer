# Backend practices extracted from your projects

**AI context — historical comparative review:** this file records observed backend work patterns, not mandatory requirements for every project. Project-specific instructions and current product decisions take precedence. Current Getwell/Linkmeds rules are in [the project reference](../../getwell/getwell-backend/docs/PROJECT-REFERENCE.md). Older password/phone-OTP, manual-only refund or proof recommendations here are not current Getwell launch instructions. Findings and source line references below retain their 15 October 2026 review date and have not been re-audited during this move.

Reviewed on 15 October 2026. Scope: `/Users/dahiribrahim/Desktop/Deedax`, with detailed inspection of the FastAPI backends. This is a static source review: implementation, route wiring, models, migrations, workers, tests, deployment files, and selected documentation. It does not establish that a feature is enabled in a deployed environment or that the tests currently pass. No application source was changed and no external services were contacted.

Your strongest recurring habit is to build the operating system around a product feature: identity, permissions, money, background work, staff workflows, notifications, reporting, and recovery. Your strongest individual implementations then add concurrency control, auditability, and explicit handling of uncertain outcomes.

“Recurring” below means observed in multiple projects. “Standout” means a particularly strong implementation in one or a few projects. Neither means present uniformly throughout every repository.

## Projects examined

| Project | Backend inspected | What it contributes most clearly |
|---|---|---|
| Case management / The Law App | `case_mgmt` | Workspace permissions, AI credit accounting, payment reconciliation, separate platform staff, document processing, live notifications, billing tests. |
| DSI / hotel management | `dsi/backend` | Capability permissions, bookings and inventory, payment exception handling, Redis caching, rotating sessions, operational reports, scheduled maintenance. |
| MoveIn | `movein/realestate` | Explicit API contract, injected providers, OTP and session lifecycle, listing revisions and moderation, media pipeline, signed cursor pagination. |
| Nasarawa election monitoring | `nasarawa_monitor/nasarawa` | Async PostgreSQL, device approval, Redis dashboard and session caches, atomic rate limiting, evidence integrity, backup workflows, load-test tooling. |
| PrepTab | `preptab/preptab_backend` | Multiple user roles, device subscriptions, activation codes, payment fulfillment, agent commissions, analytics, bulk allocation and email tasks. |
| SizeMet | `sizemet` | API-key-protected upload/job flow, durable processing status, separate compute workers, cloud/local storage options. |
| OrbitalsAI | `orbitalsai/new-backend` | Customer JWT/API-key boundaries, metered AI services, dedicated task queues, streaming, support tickets, service status and public API documentation. |
| OrbitalsAI admin | `orbitalsai/admin-app/admin-backend` | Separate operations, finance, usage, security, audit, settings, status and support surfaces. |
| OmniVoice server | `orbitalsai/omnivoice-server` | Model lifecycle, concurrency limits, request batching, warm prompt caches, readiness and latency metrics. |
| OmniVoice vLLM gateway | `orbitalsai/omnivoice-vllm-experiments/vllm_gateway` | Narrow validated gateway, mandatory configured credentials, bounded text chunks, upstream timeout and WAV integrity checks. |

Other parent-folder projects were inventoried for context. `beam` uses Flutter/Firebase; `beam.me` has Next.js waitlist handlers; `zo_mu_je` has Flutter clients and a Firebase-backed admin application. `charanchi` contains design/web iterations; `playground` contains experiments and clients. `getwell` contained architecture and contract artifacts at the review date; a FastAPI foundation was added afterward. `flutter` is an SDK checkout and was excluded as evidence of your own backend decisions. Dependency directories, virtual environments, generated build output, SSH material, and environment-secret files were excluded.

PrepTab's quiz generator and annotation tool contain Python scripts and dependency lists, but were not counted as FastAPI services merely because FastAPI appears in a requirements file. Likewise, inference research, SDK, frontend, and design folders were not counted as separate production backends.

## Your modus operandi

1. **You model the whole business workflow.** A booking has inventory, holds, expiration, payment, check-in, cancellation, receipts and staff operations. A listing has revisions, media readiness, moderation, publication and availability. An AI call has identity, usage, pricing, credits, job state and support.
2. **You usually start with a modular application.** Domain routers, schemas, models, services or CRUD modules, shared configuration, dependencies and workers recur. These are primarily modular monoliths with separate worker processes; the inference and admin services provide additional boundaries where useful.
3. **You keep authoritative business decisions on the server.** Ownership, permitted state changes, prices, subscription access, quotas and provider verification are backend concerns in the stronger implementations.
4. **You design for several actors.** Customers, agents, administrators, managers, auditors, firm members, parents, teachers and students get different capabilities and workflows.
5. **You move expensive work out of the request path.** Email, media conversion, document indexing, transcription and longer text/TTS jobs go to workers. Slow model startup also gets explicit treatment.
6. **You preserve the business record when infrastructure or providers misbehave.** Examples include pending/review states, payment reconciliation, manual refund alerts, orphan cleanup and ledger entries.
7. **You account for the actual hosting environment.** Connection recycling, PgBouncer compatibility, worker memory/concurrency, cheap health probes, Redis timeouts, cold starts and deployment entrypoints recur.
8. **You provide operational controls.** Admin endpoints, seed/bootstrap commands, reports, exports, runtime settings, backups and handoff documents make the application manageable after launch.
9. **Your strongest tests target failure modes.** Concurrent fulfillment, shared-balance overspending, access-boundary mistakes, replayed tokens, delayed payments, cache invalidation and malformed media receive explicit tests in the more developed projects.

These are inferences from the reviewed code, not claims about the order in which you personally wrote it.

## Extracted backend checklist

### Application structure and API contracts

| # | Practice found | Evidence and value |
|---|---|---|
| 1 | Domain-based routing | Recurring across the seven main backends: authentication, billing, documents, bookings, inventory, agents, listings and other domains have their own routers. See [Case app][c-main] and [DSI router][d-router]. |
| 2 | Separation of API, business logic and persistence | Recurring `api`, `services`/`crud`, `schemas`, and `models` modules. Strong examples are [MoveIn listing service][m-listings], [Case billing][c-billing] and PrepTab's `app/crud` modules. Some routes still contain substantial business logic; separation is not uniform. |
| 3 | Typed request and response schemas | Pydantic schemas, enum values, field ranges and declared response models constrain public input and serialization. [MoveIn schemas][m-schemas] and [SizeMet jobs][s-jobs]. |
| 4 | Dependency-based authentication and database access | Shared `Depends` dependencies resolve sessions, callers and roles rather than duplicating the entire authentication flow in each handler. [Case dependencies][c-deps], [MoveIn dependencies][m-deps], [Nasarawa dependencies][n-deps]. |
| 5 | Replaceable external providers | Standout: MoveIn injects OTP, email, storage and dispatchers through its app factory, with in-memory providers for tests. Nasarawa similarly exposes cache, limiter and storage interfaces/fallbacks. [MoveIn factory][m-main], [providers][m-providers]. |
| 6 | Central environment-driven configuration | Recurring settings modules keep service URLs, feature switches, pool parameters, limits and provider configuration together. Pydantic settings appear across the main backends. [MoveIn config][m-config], [Nasarawa config][n-config]. |
| 7 | Versioned and documented APIs | `/api/v1` or configured API prefixes, OpenAPI metadata, tags and contract documents recur. Case management uses domain `/api/...` prefixes rather than a uniform version prefix. [MoveIn factory][m-main], [PrepTab app][p-main], [Orbitals app][o-main]. |
| 8 | Stable structured errors and request correlation | Shared exception handlers produce machine-readable errors in Case, MoveIn and Nasarawa. MoveIn also returns `X-Request-ID` and logs unexpected failures with that identifier while returning a generic message. [MoveIn errors][m-errors], [factory][m-main]. |
| 9 | Explicit consumer-facing API boundaries | Standout: OrbitalsAI generates a selected public OpenAPI document instead of publishing every internal operation; CI checks the snapshot. MoveIn has a written canonical contract and an export script. [Public schema][o-openapi], [Orbitals CI][o-ci], [MoveIn contract][m-contract]. |

### Authentication, sessions and authorization

| # | Practice found | Evidence and value |
|---|---|---|
| 10 | Password hashing | Bcrypt and Argon2 implementations appear in the backends rather than storing plaintext passwords. [Case security][c-security], [Nasarawa security][n-security]. |
| 11 | Token expiry and purpose checking | Access and refresh tokens have distinct roles; protected dependencies reject inappropriate token types. [Case dependencies][c-deps], [MoveIn authentication][m-auth], [Nasarawa dependencies][n-deps]. |
| 12 | Email verification and account recovery | Verification, resend, forgot-password and reset-password paths recur in Case, PrepTab and OrbitalsAI; Nasarawa also has password-reset services and tests. [Case auth][c-auth], [PrepTab auth][p-auth], [Orbitals auth][o-auth]. |
| 13 | Hash-only storage for bearer secrets | Standout implementations store hashes of refresh, reset, verification, invitation or device tokens; the exact token categories differ by project. [Case security][c-security], [DSI sessions][d-auth-service], [Nasarawa security][n-security], [MoveIn auth][m-auth]. |
| 14 | Refresh-token rotation and replay detection | DSI revokes a token on refresh and burns its family on reuse. MoveIn additionally locks the refresh-session row and records reuse against the family. [DSI sessions][d-auth-service], [MoveIn auth][m-auth]. |
| 15 | Logout and revocation | DSI revokes refresh tokens; MoveIn revokes sessions; Case implements JTI revocation for account authentication. [DSI auth routes][d-auth], [MoveIn auth][m-auth], [Case Redis helper][c-redis]. Case's legacy user dependency does not apply the same JTI check. |
| 16 | Browser-aware refresh cookies | DSI uses HttpOnly cookies, production Secure settings, an auth-only path and environment-specific SameSite behavior. This is a concrete implementation choice, not evidence of a full CSRF design. [DSI sessions][d-auth-service]. |
| 17 | OTP lifecycle controls | MoveIn implements expiry, resend cooldown, per-recipient and per-IP hourly limits, a verification-attempt ceiling, hashed codes and a consumed challenge state. [MoveIn auth][m-auth]. |
| 18 | Identity normalization | MoveIn normalizes phone numbers to E.164 and email identities to lowercase; other auth paths also normalize addresses. This reduces inconsistent identities and duplicate-account surprises. [MoveIn auth][m-auth]. |
| 19 | Device-aware access and notifications | Nasarawa requires an approved device plus active agent and supports device lifecycle management. Case records known devices and alerts on new-device logins. [Nasarawa dependencies][n-deps], [device service][n-devices], [Case auth][c-auth]. |
| 20 | Roles and fine-grained capabilities | Recurring role guards; standout DSI maps job roles to explicit capabilities such as booking modification, reports, inventory and staff management. [DSI permissions][d-permissions], [MoveIn dependencies][m-deps]. |
| 21 | Object and workspace authorization | Ownership checks appear for listings, support tickets, case resources, firm billing, devices and agent submissions. These checks go beyond having a valid login. [MoveIn listing ownership][m-listings], [Case billing authorization][c-billing], [Orbitals tickets][o-tickets], [PrepTab device scope][p-device-students]. |
| 22 | Customer/staff authentication separation | Standout: Case platform staff have separate authentication, sessions and permission maps; firm administration does not confer platform administration. Nasarawa separates admin JWTs from agent device tokens. [Case platform service][c-platform], [Case billing tests][c-billing-tests], [Nasarawa dependencies][n-deps]. |
| 23 | API keys with account-management boundaries | OrbitalsAI supports product requests via API key or JWT, stores key hashes and active flags, and has a dedicated account-route authentication dependency. A regression test rejects API keys on account routes. [Orbitals security][o-security], [API keys][o-api-keys], [boundary test][o-boundary]. |
| 24 | Alternative sign-in integrations | OrbitalsAI implements Google OAuth authorization, exchange and identity-token verification services. This is a product-specific capability rather than a universal baseline. [OAuth service][o-oauth]. |

### Abuse protection and deployment-facing security

| # | Practice found | Evidence and value |
|---|---|---|
| 25 | Endpoint-specific rate limits | Case limits signup, login, reset, resend, staff login and AI queries; DSI limits auth and public booking/order/payment writes; OrbitalsAI has default and stricter chat limits. [Case auth][c-auth], [Case limiter][c-limiter], [DSI cache/limiter][d-cache], [Orbitals limiter][o-limiter]. |
| 26 | Limits keyed to both network and identity | DSI login uses both IP and normalized username limits. Nasarawa and MoveIn also apply identity-specific scopes. This addresses attacks spread across more than one IP. [DSI auth][d-auth], [Nasarawa auth API][n-auth], [MoveIn auth][m-auth]. |
| 27 | Shared atomic Redis counters | Nasarawa and OrbitalsAI use Lua to increment and set expiry together. Shared storage coordinates counts across processes. This is stronger than the separate `INCR`/`EXPIRE` calls found elsewhere. [Nasarawa limiter][n-rate], [Orbitals limiter][o-limiter]. |
| 28 | Explicit backoff signals | Several limiters return HTTP 429 with `Retry-After` or structured retry details. [Nasarawa limiter][n-rate], [Orbitals limiter][o-limiter], [MoveIn OTP][m-auth]. |
| 29 | Bounded expensive login work | Standout: Nasarawa offloads Argon2 work to threads and gates concurrent hashes with a semaphore, accounting for CPU and memory during login bursts. [Nasarawa security][n-security]. |
| 30 | Explicit CORS origins | Case, DSI, MoveIn, Nasarawa and OrbitalsAI configure origin lists. This is a recurring browser-integration decision; SizeMet and PrepTab retain wildcard fallbacks. [Case app][c-main], [DSI app][d-main], [MoveIn factory][m-main]. |
| 31 | Production configuration and documentation controls | MoveIn rejects selected unsafe production settings; Nasarawa validates JWT secret length and requires Redis when production/staging limiter/cache dependencies resolve. DSI hides docs when debug is off, while OrbitalsAI gates internal docs. [MoveIn config][m-config], [Nasarawa config][n-config], [DSI app][d-main], [Orbitals app][o-main]. |

### Database correctness and query efficiency

| # | Practice found | Evidence and value |
|---|---|---|
| 32 | PostgreSQL and SQLAlchemy as the main persistence stack | Recurring across all seven primary application backends. Async SQLAlchemy/asyncpg is prominent in DSI, MoveIn, Nasarawa, PrepTab and SizeMet; Case and OrbitalsAI use synchronous sessions. [DSI database][d-db], [Nasarawa database][n-db], [Case database][c-db], [Orbitals database][o-db]. |
| 33 | Schema migrations | Alembic histories exist in Case, DSI, MoveIn, Nasarawa, PrepTab and SizeMet. OrbitalsAI uses SQL migrations and runner scripts, including documented down migrations. [DSI migrations][d-migrations], [Orbitals migration guide][o-migrations]. |
| 34 | Session lifetime, rollback and cleanup | Dependencies and worker helpers manage sessions; stronger examples distinguish expected HTTP failures from database failures and roll back/close accordingly. [Case database][c-db], [PrepTab database][p-db], [MoveIn worker sessions][m-workers]. |
| 35 | Connection-pool resilience | Pre-ping, recycle intervals, checkout timeouts and bounded pools recur. Case and PrepTab retry stale initial connections once; Nasarawa exposes pool settings; DSI supports disabling asyncpg's statement cache for PgBouncer. [Case database][c-db], [PrepTab database][p-db], [Nasarawa database][n-db], [DSI database][d-db]. |
| 36 | Database-enforced invariants | Check and unique constraints reject invalid state even outside a particular endpoint: nonnegative stock/credits, valid vote totals, valid ranges, unique membership and event identities. [Case billing models][c-billing-models], [DSI inventory models][d-inventory-models], [Nasarawa result models][n-result-models], [MoveIn models][m-models]. |
| 37 | Targeted and partial indexes | Nasarawa indexes device tokens and submission lookups. DSI uses partial unique indexes to permit only one pending payment per target. DSI's space migration adds a PostgreSQL exclusion constraint for overlapping active reservations. [Nasarawa identity][n-identity], [DSI pending-payment migration][d-pending], [DSI spaces migration][d-space-migration]. |
| 38 | Row locking and concurrent allocation | `FOR UPDATE` protects important billing, inventory, booking, media and session changes. PrepTab uses `skip_locked` in bulk activation-code allocation. [Case billing][c-billing], [DSI inventory][d-inventory], [MoveIn auth][m-auth], [PrepTab allocation][p-super]. This is path-specific, not a claim that every write is race-safe. |
| 39 | Deliberate monetary arithmetic | DSI and MoveIn use Decimal/Numeric for prices. Case uses integer kobo and scaled integer credit/provider-cost units. [DSI pricing][d-pricing], [MoveIn schemas][m-schemas], [Case credits][c-credits]. OrbitalsAI still has float-based billing paths. |
| 40 | Explicit time semantics | UTC-aware expiry and session logic recur; DSI has a configured business timezone helper for local dates. [DSI time][d-time], [MoveIn auth][m-auth], [Case billing][c-billing]. Time handling is not uniform throughout older code. |
| 41 | Query-aware lists and relationships | Eager loading and batched lookups appear in DSI, PrepTab, Case and MoveIn; list endpoints provide filtering/pagination. MoveIn's feed uses a signed keyset cursor bound to the active filters. [PrepTab student CRUD][p-students], [MoveIn feed][m-listings], [Case security regressions][c-security-tests]. |

### Caching and graceful degradation

| # | Practice found | Evidence and value |
|---|---|---|
| 42 | Redis caching for hot reads | DSI caches public rooms, spaces, menus and daily menus. Nasarawa caches election dashboard views. OrbitalsAI caches status/history data. [DSI cache helper][d-cache], [DSI rooms][d-rooms], [Nasarawa dashboard][n-dashboard], [Orbitals status][o-status]. |
| 43 | Expiring, parameter-aware cache keys | TTLs bound staleness. Nasarawa dashboard keys include election, version and relevant query parameters; dedicated tests check parameter isolation. [Nasarawa cache][n-cache], [dashboard][n-dashboard], [cache tests][n-cache-tests]. |
| 44 | Invalidation on writes | DSI invalidates affected prefixes after edits. Nasarawa bumps an election dashboard version on submission/review writes, with optional bump coalescing. [DSI rooms][d-rooms], [Nasarawa submissions][n-submissions], [Nasarawa cache][n-cache]. |
| 45 | Authentication-path caching with a database fallback | Nasarawa caches device/agent validation with a configured TTL and delete operations. A miss or Redis failure falls through to the indexed PostgreSQL lookup. [Agent session cache][n-session-cache], [dependencies][n-deps]. |
| 46 | Explicit cache failure behavior | DSI and Nasarawa return cache misses and continue against the source database when cache operations fail. OrbitalsAI status history also has a memory fallback. This keeps optional acceleration from becoming an availability dependency. [DSI cache][d-cache], [Nasarawa cache][n-cache], [Orbitals status][o-status]. |
| 47 | Small local caches for expensive metadata and model preparation | OrbitalsAI caches pricing and status data; OmniVoice warms reusable voice-clone prompts. This is selective caching of repeated work, separate from durable business state. [Orbitals billing][o-billing], [OmniVoice inference][v-inference]. |

### Background work and recovery

| # | Practice found | Evidence and value |
|---|---|---|
| 48 | Celery/Redis job processing | Six of the seven main backends use Celery: Case, DSI, MoveIn, PrepTab, SizeMet and OrbitalsAI. Jobs cover email, documents, media, reports, audio and text. [Case tasks config][c-celery], [SizeMet jobs][s-jobs], [Orbitals workers][o-workers]. |
| 49 | Isolation of workloads with different latency | Case routes email separately. OrbitalsAI separates audio, LLM and TTS queues/workers, so a cold text model need not hold up transcription. [Case tasks config][c-celery], [Orbitals workers][o-workers], [worker container][o-worker-docker]. |
| 50 | Retry policies and backoff | Case email tasks retry; PrepTab email tasks configure automatic retries and backoff; MoveIn notification/media workers use explicit bounded retries. [Case email tasks][c-email-tasks], [PrepTab email tasks][p-email-tasks], [MoveIn notifications][m-email-worker]. |
| 51 | Worker reliability and resource settings | Selected workers configure JSON serialization, late acknowledgements, worker-loss behavior, prefetch, time limits, result expiry, memory or concurrency limits. [Case Celery][c-celery], [PrepTab Celery][p-celery], [Orbitals worker][o-workers]. These settings differ by project. |
| 52 | Scheduled business maintenance | Beat jobs handle booking expiry, night audit, pre-arrival email, archives, billing sweeps, hearing reminders, stale uploads and availability reminders. [DSI schedules][d-celery], [Case schedules][c-celery], [MoveIn schedules][m-workers]. |
| 53 | Durable job state and result retrieval | SizeMet creates a pending job, returns HTTP 202 and exposes status/results. OrbitalsAI persists audio/text/TTS work and generation history. [SizeMet jobs][s-jobs], [Orbitals TTS migration guide][o-migrations]. |
| 54 | Reconciliation and cleanup paths | Case cleans pending documents and reconciles uncertain payment/AI operations. MoveIn reconciles media. DSI has periodic expiry cleanup as a safety net in addition to individual expiry tasks. [Case Celery][c-celery], [Case credits][c-credits], [MoveIn media worker][m-media-worker], [DSI schedules][d-celery]. |

### Uploads, object storage and media

| # | Practice found | Evidence and value |
|---|---|---|
| 55 | Object storage behind service modules | R2/S3-compatible and Azure Blob helpers keep binary data handling separate from API/business modules. Database rows retain ownership, state and metadata. [MoveIn providers][m-providers], [Nasarawa storage][n-storage], [Orbitals tickets][o-tickets]. |
| 56 | Presigned upload and finalize lifecycle | MoveIn allocates an owned media record and expiring upload URL, then checks object metadata before advancing its state. Case and Nasarawa also provide direct object-storage upload workflows. [MoveIn media][m-media], [Case document service][c-documents], [Nasarawa storage][n-storage]. |
| 57 | File type, size and count limits | Case shares chunked size/type validation across upload paths; MoveIn limits media types, bytes and per-listing counts; support tickets cap attachment count/size. [Case uploads][c-uploads], [MoveIn media][m-media], [Orbitals tickets][o-tickets]. |
| 58 | Checks against actual file content | MoveIn decodes images and probes video format/duration instead of relying only on filenames. OrbitalsAI support attachments check signatures or text samples. [MoveIn processing][m-processing], [Orbitals tickets][o-tickets]. These checks are stronger than declared-MIME checks elsewhere. |
| 59 | Ownership-bound evidence and integrity checks | Nasarawa constructs evidence keys from election/scope/agent identity and provides streamed SHA-256 verification; submission services validate the assigned scope. [Nasarawa storage][n-storage], [submissions][n-submissions]. |
| 60 | Private authorized downloads | Case document access and OrbitalsAI support attachment downloads check access before providing download URLs. Expiring storage links avoid making private attachments public by default. [Case document routes][c-document-routes], [Orbitals tickets][o-tickets]. |
| 61 | Derived media and failed-upload cleanup | MoveIn generates optimized images/thumbnails and video derivatives, strips EXIF from exported images and reconciles media states. OrbitalsAI removes uploaded blobs if attachment persistence fails. [MoveIn processing][m-processing], [media worker][m-media-worker], [Orbitals tickets][o-tickets]. |

### Money, subscriptions and commercial completeness

| # | Practice found | Evidence and value |
|---|---|---|
| 62 | Payment initialization plus server verification | Paystack/Monnify flows include backend initialization, verification and fulfillment. Case verifies reference, integer amount, currency, environment, payer and transaction identity against the invoice. [Case payments][c-paystack], [DSI payments][d-payments], [PrepTab payments][p-payments]. |
| 63 | Webhook authentication over the raw body | HMAC signature checks occur before trusting payment payloads, with constant-time digest comparison in provider helpers. [Case payments][c-paystack], [DSI webhooks][d-webhooks], [PrepTab Monnify][p-monnify]. |
| 64 | Idempotency at consequential boundaries | Examples include invoice/request keys, AI request fingerprints, provider transaction uniqueness, submission client UUIDs and analytics event keys. Reusing a key with different input is rejected in Case AI billing. [Case credits][c-credits], [Case billing models][c-billing-models], [Nasarawa submissions][n-submissions], [MoveIn models][m-models]. |
| 65 | Durable intent before external money side effects | Standout: Case commits a payment attempt before provider dispatch. An uncertain result enters reconciliation instead of automatically issuing another charge. [Case payments][c-paystack]. |
| 66 | Prepaid holds, settlement and credit ledger | Standout: Case reserves credit allocations, settles measured provider usage, releases unused holds, records ledger entries and routes unknown outcomes to review. [Case credits][c-credits]. |
| 67 | Versioned commercial terms | Case invoices snapshot terms and reference versioned catalog prices; subscription/invoice history is preserved when a new price is published. [Case billing models][c-billing-models], [payments][c-paystack], [tests][c-billing-tests]. |
| 68 | Subscription lifecycle and entitlement enforcement | Case models trialing/active/past-due/expired/suspended access and renewals. PrepTab checks device subscription expiry; OrbitalsAI maintains balances and usage. [Case billing][c-billing], [PrepTab subscriptions][p-subscriptions], [Orbitals billing][o-billing]. |
| 69 | Quotas, capacity and spend warnings | Case enforces firm seats, storage reservations and AI credit availability, with threshold notices and overrun review. PrepTab limits student profiles per device and allocates activation codes. [Case billing][c-billing], [credits][c-credits], [PrepTab device scope][p-device-students]. |
| 70 | Payment exception workflows | DSI handles underpayment, duplicate payment and payment after a reservation expired, including staff alerts/refund handling when fulfillment is impossible. Case places mismatched, late or excess payments into review. [DSI webhooks][d-webhooks], [Case payments][c-paystack], [DSI regressions][d-webhook-tests]. |
| 71 | Related commercial operations | PrepTab implements agent commissions, activation-code purchases and approval/allocation flows. DSI handles group/deferred bookings and inventory movement. These are signs of domain completeness beyond checkout alone. [PrepTab commission CRUD][p-commissions], [allocation][p-super], [DSI group service][d-groups], [inventory][d-inventory]. |
| 72 | Auditable business and financial history | Case has billing audit and credit ledger rows; DSI has actor-based audits and inventory movements; Nasarawa records administrative/device/submission activity. [Case billing models][c-billing-models], [DSI audit][d-audit], [DSI inventory models][d-inventory-models], [Nasarawa identity][n-identity]. |

### Administration, communication and business workflows

| # | Practice found | Evidence and value |
|---|---|---|
| 73 | Admin tooling as part of the product | User/device management, reporting, finance, support, moderation and configuration have backend surfaces. OrbitalsAI even separates an admin application. [Orbitals admin][a-main], [Case platform service][c-platform], [DSI permissions][d-permissions]. |
| 74 | Runtime settings with controlled changes | Standout: Case checks staff permissions, validates setting values, requires expected versions and audits before/after values and reasons. Feature gates protect relevant operations. [Case platform service][c-platform], [Case billing tests][c-billing-tests]. |
| 75 | Explicit lifecycle transitions and moderation | MoveIn separates drafts/revisions, submission, approval/rejection and publication, and checks media readiness. DSI separates booking and payment status. Nasarawa validates active elections and reviews submissions. [MoveIn listings][m-listings], [DSI webhooks][d-webhooks], [Nasarawa submissions][n-submissions]. |
| 76 | Durable in-app notification feeds | Case and Nasarawa persist notifications and recipient/read state; DSI has notifications linked to business entities. [Case notification service][c-notifications], [Nasarawa notifications][n-notifications], [DSI notification model][d-notification-model]. |
| 77 | Multiple delivery channels | Case uses Redis pub/sub plus SSE and Web Push; Nasarawa sends mobile push through registered FCM devices. These are project-specific implementations, not a shared universal notification platform. [Case notification routes][c-notification-routes], [delivery tasks][c-notification-tasks], [Nasarawa notifications][n-notifications]. |
| 78 | Transactional email and reminders | Verification, reset, booking confirmations, receipts, cancellation, pre-arrival, hearing reminders and digest flows are implemented across projects. [Case email tasks][c-email-tasks], [DSI tasks][d-tasks], [PrepTab email tasks][p-email-tasks]. |
| 79 | Customer support lifecycle | OrbitalsAI has owned tickets, replies, categories, closure, private attachments and admin support routes. [Ticket service][o-tickets], [admin application][a-main]. |
| 80 | Exports, reports, bulk operations and bootstrap tools | DSI provides downloadable PDF reports and revenue/occupancy/staff reporting. Nasarawa provides CSV exports, import/seed commands and admin bootstrap tools. MoveIn has an admin CLI. [DSI reports][d-reports], [Nasarawa dashboard API][n-dashboard-api], [MoveIn CLI][m-cli]. |

### Deployment, observability and disaster recovery

| # | Practice found | Evidence and value |
|---|---|---|
| 81 | Containerized runtime and worker topology | Docker and Compose/Procfiles recur, with separate API/worker/scheduler roles. Nasarawa and PrepTab use multistage images; DSI, Nasarawa, PrepTab and SizeMet include non-root runtime users. [DSI Dockerfile][d-docker], [Nasarawa Dockerfile][n-docker], [PrepTab Dockerfile][p-docker], [Case Compose][c-compose]. |
| 82 | Service health checks, dependency ordering and persistent volumes | Compose definitions check PostgreSQL/Redis/API/worker health, wait for dependencies and retain state in volumes. Nasarawa also includes PgBouncer. [Case Compose][c-compose], [DSI Compose][d-compose], [Nasarawa Compose][n-compose], [MoveIn Compose][m-compose]. |
| 83 | Deployment-time migrations and environment-specific startup | Case/DSI Compose runs Alembic before the web process; Nasarawa and PrepTab entrypoints handle startup. There are deployment definitions for several hosting environments. [Case Compose][c-compose], [DSI Compose][d-compose], [Nasarawa Dockerfile][n-docker], [PrepTab Dockerfile][p-docker]. |
| 84 | Cheap liveness separated from deeper checks | PrepTab and Nasarawa separate `/health` from `/health/db`. OrbitalsAI exposes `/live`, `/ready` and `/startup`; OmniVoice reports model readiness and memory. The exact meaning of “ready” differs by service. [PrepTab app][p-main], [Nasarawa health][n-health], [Orbitals app][o-main], [OmniVoice health][v-health]. |
| 85 | Operational status, latency and independent worker heartbeat | OrbitalsAI records component status/history and uses a Redis heartbeat emitted outside the task queue. OmniVoice exposes success/error/timeout and latency metrics. Request duration logging exists in the main Orbitals backend. [Orbitals status][o-status], [worker][o-workers], [OmniVoice metrics][v-metrics], [Orbitals app][o-main]. |
| 86 | Backup jobs and restore procedures | Case schedules PostgreSQL dumps to private R2. DSI has scheduled database backup and archive tasks. Nasarawa implements dump timeout, duplicate-run coordination, retention pruning and backup/restore documentation. [Case backup][c-backup], [DSI tasks][d-tasks], [Nasarawa backups][n-backup], [restore guide][n-restore]. These are code/configuration findings, not proof of a successful restore drill. |

### Verification, documentation and specialist AI work

| # | Practice found | Evidence and value |
|---|---|---|
| 87 | Automated tests aimed at important failures | Case, DSI, MoveIn, Nasarawa and OrbitalsAI contain substantial tests for access, sessions, concurrency, payment replay, media and cache invalidation. See the static inventory below and [Case billing tests][c-billing-tests], [DSI webhook tests][d-webhook-tests], [Nasarawa cache tests][n-cache-tests]. |
| 88 | CI and contract checks | Nasarawa's CI runs pytest, Ruff and mypy. DSI's runs pytest. OrbitalsAI's runs unit/integration/ASR regression tests, a public OpenAPI check and coverage reporting; its lint steps are commented out. [Nasarawa CI][n-ci], [DSI CI][d-ci], [Orbitals CI][o-ci]. |
| 89 | Realistic load-test tooling and runbooks | Nasarawa supplies Locust scenarios for agent login/submission bursts, dashboard polling, distinct credentials and latency/error SLOs. Projects also contain deployment, go-live, seeding and handoff documents. [Load-test guide][n-load], [DSI go-live plan][d-go-live], [MoveIn contract][m-contract]. The guide's targets are not measured capacity claims. |
| 90 | Domain-aware AI processing and output validation | Case extracts and token-chunks documents for embeddings/retrieval and persists chat context/sources. OrbitalsAI adds token-aware chunking, structured-output validation and checks for unsupported facts in summaries. OmniVoice adds concurrency gating, compatible-request batching and warm prompt caches. Its vLLM gateway validates chunk sizes, WAV integrity and upstream errors. [Case processing][c-processing], [Case AI][c-ai], [Orbitals validators][o-validators], [chunking][o-chunking], [OmniVoice inference][v-inference], [gateway][g-app]. |

## How feature-complete the codebases are

The source shows broad business scope, especially in DSI, Case management, PrepTab and OrbitalsAI. Feature breadth and operational hardening are separate measurements: a project can have many endpoints while still lacking consistent rate limits, tests or failure handling.

Static inventory of seven main applications, excluding dependencies and generated code:

| Backend | Decorated route handlers | `test_*.py` files under `tests/` | Named test functions | Alembic version files |
|---|---:|---:|---:|---:|
| Case management | 99 | 17 | 119 | 23 |
| DSI | 140 | 12 | 85 | 38 |
| MoveIn | 57 | 8 | 39 | 2 |
| Nasarawa | 73 | 33 | 196 | 13 |
| PrepTab | 118 | 0 | 0 | 17 |
| SizeMet | 8 | 0 | 0 | 3 |
| OrbitalsAI main backend | 74 | 30 | 315 | Uses SQL/runner migrations |
| **Total** | **669** | **100** | **754** | **96 Alembic files, plus OrbitalsAI's SQL migrations** |

Counts come from static Python AST inspection. Route counts include aliases/multiple route decorators and are not unique public API operation counts. Named test functions are not collected pytest case counts: parametrization, fixtures, skips and runtime availability can change what actually executes. Tests outside the conventional `tests/` directory were not included. None of these counts establishes code coverage or a passing suite.

The strongest examples by concern are:

- **Financial correctness:** Case management's invoice snapshots, credit holds/settlement, ledger, reconciliation and concurrent fulfillment tests.
- **Booking and staff operations:** DSI's inventory, holds, groups, payments, reports, capabilities and exception workflows.
- **Cache and burst-traffic design:** Nasarawa's dashboard versions, session cache, atomic limits, password-hash gate, connection controls and realistic load tooling.
- **Contract and media lifecycle:** MoveIn's injected providers, revision workflow, content checks, OTP/session design and signed filtered cursors.
- **Developer-facing AI operations:** OrbitalsAI's JWT/API-key boundary, queues, status page, support, typed public API and model-focused regression tests.
- **Multi-actor educational commerce:** PrepTab's student/parent/teacher/school/admin/agent flows, subscriptions, activation inventory and commissions.
- **Focused compute job service:** SizeMet's protected upload → durable job → worker → result workflow.

## Boundaries on what can be called your standard

1. **Rate limiting and response caching are uneven.** Redis exists in many projects as a broker, but that alone does not mean API response caching. I verified response caching most clearly in DSI, Nasarawa and OrbitalsAI status flows. I did not find comparable shared rate-limit wiring in the reviewed PrepTab or SizeMet application code. MoveIn's concrete abuse controls are concentrated in its OTP flows.
2. **Fail-open is an availability tradeoff.** Cache fallback is a useful recurring pattern. Several rate limiters also fail open, and Case's revocation helper permits requests if Redis is unavailable. These are not equivalent security guarantees. A revocation outage can leave an otherwise revoked access token usable until expiry.
3. **Concurrency protection is path-specific.** The row locks and database constraints are real, but their presence does not make all endpoints atomic. For example, DSI's refresh rotation reads a row without the explicit lock that MoveIn uses. OrbitalsAI's billing service also has float-based balance-update paths rather than Case's stronger integer/hold approach.
4. **Hardening is not uniform across generations.** SizeMet allows wildcard CORS with credentials; PrepTab can fall back to that combination. Case's legacy user and newer account dependencies differ on revocation. These should not be copied as one consistent security baseline.
5. **Tests and CI are substantial in some repositories, absent or unverified in others.** No conventional `tests/` suites were found in PrepTab or SizeMet during this review. Configured CI checks were inspected, not executed. Some test suites require PostgreSQL, Redis, containers or model endpoints.
6. **Observability is selective.** Request IDs, status pages, metrics and worker heartbeats exist in specific services. I did not establish a common distributed-tracing, alerting or metrics platform across the whole portfolio.
7. **GetWell is currently design evidence.** Its contract covers consequential-command idempotency/versioning, explicit money/status semantics and operational workflows, but its own architecture README states that the SQL is not a production migration and that transaction/access controls still need implementation. It belongs in your design approach, not the implemented-feature inventory.

The reusable standard supported by this review is: **a modular, typed, permission-aware backend with authoritative business rules, migration-managed data, bounded public operations, selective caching, background jobs, auditable money/state changes, staff tooling and a documented deployment/recovery path.** The strongest additions to carry forward consistently are Case's money-boundary discipline, Nasarawa's burst/cache controls and MoveIn's contract/provider/media boundaries.

## Source references

[c-main]: /Users/dahiribrahim/Desktop/Deedax/case_mgmt/app/main.py:30
[c-security]: /Users/dahiribrahim/Desktop/Deedax/case_mgmt/app/core/security.py:11
[c-deps]: /Users/dahiribrahim/Desktop/Deedax/case_mgmt/app/api/deps.py:19
[c-db]: /Users/dahiribrahim/Desktop/Deedax/case_mgmt/app/db/session.py:27
[c-auth]: /Users/dahiribrahim/Desktop/Deedax/case_mgmt/app/api/routes/auth.py:76
[c-limiter]: /Users/dahiribrahim/Desktop/Deedax/case_mgmt/app/core/limiter.py:15
[c-redis]: /Users/dahiribrahim/Desktop/Deedax/case_mgmt/app/core/redis_client.py:34
[c-billing]: /Users/dahiribrahim/Desktop/Deedax/case_mgmt/app/services/billing_service.py:45
[c-credits]: /Users/dahiribrahim/Desktop/Deedax/case_mgmt/app/services/credit_service.py:41
[c-paystack]: /Users/dahiribrahim/Desktop/Deedax/case_mgmt/app/services/paystack_service.py:136
[c-billing-models]: /Users/dahiribrahim/Desktop/Deedax/case_mgmt/app/db/models/billing.py:20
[c-platform]: /Users/dahiribrahim/Desktop/Deedax/case_mgmt/app/services/platform_admin_service.py:41
[c-celery]: /Users/dahiribrahim/Desktop/Deedax/case_mgmt/app/core/celery_app.py:36
[c-email-tasks]: /Users/dahiribrahim/Desktop/Deedax/case_mgmt/app/tasks/email_tasks.py:19
[c-notification-tasks]: /Users/dahiribrahim/Desktop/Deedax/case_mgmt/app/tasks/notification_tasks.py:30
[c-notification-routes]: /Users/dahiribrahim/Desktop/Deedax/case_mgmt/app/api/routes/notifications.py:102
[c-notifications]: /Users/dahiribrahim/Desktop/Deedax/case_mgmt/app/services/notification_service.py:197
[c-uploads]: /Users/dahiribrahim/Desktop/Deedax/case_mgmt/app/core/uploads.py:30
[c-documents]: /Users/dahiribrahim/Desktop/Deedax/case_mgmt/app/services/document_service.py:151
[c-document-routes]: /Users/dahiribrahim/Desktop/Deedax/case_mgmt/app/api/routes/documents.py:254
[c-processing]: /Users/dahiribrahim/Desktop/Deedax/case_mgmt/app/services/document_processor.py:152
[c-ai]: /Users/dahiribrahim/Desktop/Deedax/case_mgmt/app/services/ai_service.py:1
[c-billing-tests]: /Users/dahiribrahim/Desktop/Deedax/case_mgmt/tests/test_billing.py:130
[c-security-tests]: /Users/dahiribrahim/Desktop/Deedax/case_mgmt/tests/api/test_security_fixes.py:219
[c-compose]: /Users/dahiribrahim/Desktop/Deedax/case_mgmt/docker-compose.yaml:18
[c-backup]: /Users/dahiribrahim/Desktop/Deedax/case_mgmt/scripts/backup.sh:11
[d-main]: /Users/dahiribrahim/Desktop/Deedax/dsi/backend/app/main.py:6
[d-router]: /Users/dahiribrahim/Desktop/Deedax/dsi/backend/app/api/router.py:1
[d-cache]: /Users/dahiribrahim/Desktop/Deedax/dsi/backend/app/services/cache.py:31
[d-auth]: /Users/dahiribrahim/Desktop/Deedax/dsi/backend/app/api/v1/auth.py:39
[d-auth-service]: /Users/dahiribrahim/Desktop/Deedax/dsi/backend/app/services/auth.py:29
[d-permissions]: /Users/dahiribrahim/Desktop/Deedax/dsi/backend/app/core/permissions.py:10
[d-db]: /Users/dahiribrahim/Desktop/Deedax/dsi/backend/app/database.py:9
[d-migrations]: /Users/dahiribrahim/Desktop/Deedax/dsi/backend/alembic/versions/844f1a7b37dc_initial_revision.py:1
[d-pending]: /Users/dahiribrahim/Desktop/Deedax/dsi/backend/alembic/versions/h7i8j9k0l1m2_one_pending_payment_per_target.py:70
[d-space-migration]: /Users/dahiribrahim/Desktop/Deedax/dsi/backend/alembic/versions/e4f5a6b7c8d9_add_bookable_spaces.py:118
[d-inventory]: /Users/dahiribrahim/Desktop/Deedax/dsi/backend/app/services/inventory.py:28
[d-inventory-models]: /Users/dahiribrahim/Desktop/Deedax/dsi/backend/app/models/inventory.py:62
[d-pricing]: /Users/dahiribrahim/Desktop/Deedax/dsi/backend/app/services/pricing.py:45
[d-time]: /Users/dahiribrahim/Desktop/Deedax/dsi/backend/app/core/time.py:7
[d-rooms]: /Users/dahiribrahim/Desktop/Deedax/dsi/backend/app/api/v1/rooms.py:48
[d-celery]: /Users/dahiribrahim/Desktop/Deedax/dsi/backend/app/celery_app/celery.py:29
[d-tasks]: /Users/dahiribrahim/Desktop/Deedax/dsi/backend/app/celery_app/tasks.py:1862
[d-payments]: /Users/dahiribrahim/Desktop/Deedax/dsi/backend/app/api/v1/payments.py:132
[d-webhooks]: /Users/dahiribrahim/Desktop/Deedax/dsi/backend/app/api/v1/webhooks.py:16
[d-webhook-tests]: /Users/dahiribrahim/Desktop/Deedax/dsi/backend/tests/test_webhook_late_payment.py:63
[d-groups]: /Users/dahiribrahim/Desktop/Deedax/dsi/backend/app/services/booking_group.py:27
[d-audit]: /Users/dahiribrahim/Desktop/Deedax/dsi/backend/app/core/audit.py:1
[d-notification-model]: /Users/dahiribrahim/Desktop/Deedax/dsi/backend/app/models/notification.py:1
[d-reports]: /Users/dahiribrahim/Desktop/Deedax/dsi/backend/app/api/v1/reports.py:67
[d-docker]: /Users/dahiribrahim/Desktop/Deedax/dsi/backend/Dockerfile:53
[d-compose]: /Users/dahiribrahim/Desktop/Deedax/dsi/backend/docker-compose.yaml:77
[d-ci]: /Users/dahiribrahim/Desktop/Deedax/dsi/backend/.github/workflows/ci.yml:26
[d-go-live]: /Users/dahiribrahim/Desktop/Deedax/dsi/backend/docs/GO_LIVE_TEST_PLAN.md
[m-main]: /Users/dahiribrahim/Desktop/Deedax/movein/realestate/app/main.py:30
[m-deps]: /Users/dahiribrahim/Desktop/Deedax/movein/realestate/app/core/dependencies.py:19
[m-providers]: /Users/dahiribrahim/Desktop/Deedax/movein/realestate/app/providers.py:13
[m-config]: /Users/dahiribrahim/Desktop/Deedax/movein/realestate/app/core/config.py:79
[m-errors]: /Users/dahiribrahim/Desktop/Deedax/movein/realestate/app/core/exceptions.py:43
[m-auth]: /Users/dahiribrahim/Desktop/Deedax/movein/realestate/app/services/auth_v1.py:77
[m-listings]: /Users/dahiribrahim/Desktop/Deedax/movein/realestate/app/services/listing_v1.py:50
[m-schemas]: /Users/dahiribrahim/Desktop/Deedax/movein/realestate/app/schemas/v1.py:148
[m-models]: /Users/dahiribrahim/Desktop/Deedax/movein/realestate/app/models/canonical.py:268
[m-media]: /Users/dahiribrahim/Desktop/Deedax/movein/realestate/app/services/media_v1.py:53
[m-processing]: /Users/dahiribrahim/Desktop/Deedax/movein/realestate/app/services/media_processing.py:38
[m-workers]: /Users/dahiribrahim/Desktop/Deedax/movein/realestate/app/workers/celery_app.py:10
[m-email-worker]: /Users/dahiribrahim/Desktop/Deedax/movein/realestate/app/workers/notifications.py:8
[m-media-worker]: /Users/dahiribrahim/Desktop/Deedax/movein/realestate/app/workers/media.py:1
[m-contract]: /Users/dahiribrahim/Desktop/Deedax/movein/realestate/docs/backend-v1-contract.md
[m-cli]: /Users/dahiribrahim/Desktop/Deedax/movein/realestate/app/cli.py:1
[m-compose]: /Users/dahiribrahim/Desktop/Deedax/movein/realestate/docker-compose.yaml:16
[n-main]: /Users/dahiribrahim/Desktop/Deedax/nasarawa_monitor/nasarawa/app/main.py:10
[n-config]: /Users/dahiribrahim/Desktop/Deedax/nasarawa_monitor/nasarawa/app/core/config.py:164
[n-deps]: /Users/dahiribrahim/Desktop/Deedax/nasarawa_monitor/nasarawa/app/api/deps.py:22
[n-db]: /Users/dahiribrahim/Desktop/Deedax/nasarawa_monitor/nasarawa/app/db/session.py:14
[n-rate]: /Users/dahiribrahim/Desktop/Deedax/nasarawa_monitor/nasarawa/app/services/rate_limit_service.py:28
[n-cache]: /Users/dahiribrahim/Desktop/Deedax/nasarawa_monitor/nasarawa/app/services/cache_service.py:37
[n-dashboard]: /Users/dahiribrahim/Desktop/Deedax/nasarawa_monitor/nasarawa/app/services/dashboard_service.py:281
[n-submissions]: /Users/dahiribrahim/Desktop/Deedax/nasarawa_monitor/nasarawa/app/services/submission_service.py:28
[n-security]: /Users/dahiribrahim/Desktop/Deedax/nasarawa_monitor/nasarawa/app/core/security.py:23
[n-auth]: /Users/dahiribrahim/Desktop/Deedax/nasarawa_monitor/nasarawa/app/api/auth.py:1
[n-devices]: /Users/dahiribrahim/Desktop/Deedax/nasarawa_monitor/nasarawa/app/services/device_service.py:1
[n-session-cache]: /Users/dahiribrahim/Desktop/Deedax/nasarawa_monitor/nasarawa/app/services/agent_session_service.py:70
[n-storage]: /Users/dahiribrahim/Desktop/Deedax/nasarawa_monitor/nasarawa/app/services/storage_service.py:108
[n-identity]: /Users/dahiribrahim/Desktop/Deedax/nasarawa_monitor/nasarawa/app/models/identity.py:126
[n-result-models]: /Users/dahiribrahim/Desktop/Deedax/nasarawa_monitor/nasarawa/app/models/result.py:119
[n-notifications]: /Users/dahiribrahim/Desktop/Deedax/nasarawa_monitor/nasarawa/app/services/notification_service.py:53
[n-dashboard-api]: /Users/dahiribrahim/Desktop/Deedax/nasarawa_monitor/nasarawa/app/api/dashboard.py:109
[n-health]: /Users/dahiribrahim/Desktop/Deedax/nasarawa_monitor/nasarawa/app/api/health.py:13
[n-backup]: /Users/dahiribrahim/Desktop/Deedax/nasarawa_monitor/nasarawa/app/services/backup_service.py:281
[n-restore]: /Users/dahiribrahim/Desktop/Deedax/nasarawa_monitor/nasarawa/docs/instructions/DATABASE_BACKUP_RESTORE.md
[n-cache-tests]: /Users/dahiribrahim/Desktop/Deedax/nasarawa_monitor/nasarawa/tests/test_dashboard_cache.py:124
[n-ci]: /Users/dahiribrahim/Desktop/Deedax/nasarawa_monitor/nasarawa/.github/workflows/ci.yml:26
[n-load]: /Users/dahiribrahim/Desktop/Deedax/nasarawa_monitor/nasarawa/loadtest/README.md:3
[n-docker]: /Users/dahiribrahim/Desktop/Deedax/nasarawa_monitor/nasarawa/Dockerfile:24
[n-compose]: /Users/dahiribrahim/Desktop/Deedax/nasarawa_monitor/nasarawa/docker-compose.yaml:18
[p-main]: /Users/dahiribrahim/Desktop/Deedax/preptab/preptab_backend/app/main.py:41
[p-db]: /Users/dahiribrahim/Desktop/Deedax/preptab/preptab_backend/app/core/database.py:16
[p-auth]: /Users/dahiribrahim/Desktop/Deedax/preptab/preptab_backend/app/api/v1/endpoints/auth.py:1
[p-super]: /Users/dahiribrahim/Desktop/Deedax/preptab/preptab_backend/app/api/v1/endpoints/super_user.py:458
[p-students]: /Users/dahiribrahim/Desktop/Deedax/preptab/preptab_backend/app/crud/student.py:28
[p-device-students]: /Users/dahiribrahim/Desktop/Deedax/preptab/preptab_backend/app/api/v1/endpoints/devices_students.py:41
[p-subscriptions]: /Users/dahiribrahim/Desktop/Deedax/preptab/preptab_backend/app/core/subscription.py:15
[p-payments]: /Users/dahiribrahim/Desktop/Deedax/preptab/preptab_backend/app/api/v1/endpoints/payments.py:528
[p-monnify]: /Users/dahiribrahim/Desktop/Deedax/preptab/preptab_backend/app/services/monnify.py:293
[p-commissions]: /Users/dahiribrahim/Desktop/Deedax/preptab/preptab_backend/app/crud/commission.py:18
[p-celery]: /Users/dahiribrahim/Desktop/Deedax/preptab/preptab_backend/app/core/celery_app.py:53
[p-email-tasks]: /Users/dahiribrahim/Desktop/Deedax/preptab/preptab_backend/app/tasks/email_tasks.py:24
[p-docker]: /Users/dahiribrahim/Desktop/Deedax/preptab/preptab_backend/Dockerfile:15
[s-jobs]: /Users/dahiribrahim/Desktop/Deedax/sizemet/app/api/routes/jobs.py:17
[o-main]: /Users/dahiribrahim/Desktop/Deedax/orbitalsai/new-backend/app/src/main.py:66
[o-db]: /Users/dahiribrahim/Desktop/Deedax/orbitalsai/new-backend/app/src/db/session.py:19
[o-security]: /Users/dahiribrahim/Desktop/Deedax/orbitalsai/new-backend/app/src/core/security.py:1
[o-api-keys]: /Users/dahiribrahim/Desktop/Deedax/orbitalsai/new-backend/app/src/models/api_key.py:13
[o-boundary]: /Users/dahiribrahim/Desktop/Deedax/orbitalsai/new-backend/tests/integration/test_account_auth_boundary.py:4
[o-auth]: /Users/dahiribrahim/Desktop/Deedax/orbitalsai/new-backend/app/src/api/v1/endpoints/auth.py:1
[o-oauth]: /Users/dahiribrahim/Desktop/Deedax/orbitalsai/new-backend/app/src/services/oauth.py:16
[o-limiter]: /Users/dahiribrahim/Desktop/Deedax/orbitalsai/new-backend/app/src/middleware/rate_limiter.py:18
[o-billing]: /Users/dahiribrahim/Desktop/Deedax/orbitalsai/new-backend/app/src/services/billing.py:13
[o-workers]: /Users/dahiribrahim/Desktop/Deedax/orbitalsai/new-backend/app/src/celery_worker.py:11
[o-worker-docker]: /Users/dahiribrahim/Desktop/Deedax/orbitalsai/new-backend/Dockerfile.worker:28
[o-tickets]: /Users/dahiribrahim/Desktop/Deedax/orbitalsai/new-backend/app/src/services/support_ticket.py:184
[o-status]: /Users/dahiribrahim/Desktop/Deedax/orbitalsai/new-backend/app/src/services/status_service.py:28
[o-openapi]: /Users/dahiribrahim/Desktop/Deedax/orbitalsai/new-backend/app/src/public_openapi.py:1
[o-ci]: /Users/dahiribrahim/Desktop/Deedax/orbitalsai/new-backend/.github/workflows/ci.yml:71
[o-migrations]: /Users/dahiribrahim/Desktop/Deedax/orbitalsai/new-backend/app/migrations/README.md
[o-validators]: /Users/dahiribrahim/Desktop/Deedax/orbitalsai/new-backend/app/src/services/llm_validators.py:144
[o-chunking]: /Users/dahiribrahim/Desktop/Deedax/orbitalsai/new-backend/app/src/services/chunking.py:69
[a-main]: /Users/dahiribrahim/Desktop/Deedax/orbitalsai/admin-app/admin-backend/main.py:49
[v-inference]: /Users/dahiribrahim/Desktop/Deedax/orbitalsai/omnivoice-server/omnivoice_server/services/inference.py:227
[v-health]: /Users/dahiribrahim/Desktop/Deedax/orbitalsai/omnivoice-server/omnivoice_server/routers/health.py:14
[v-metrics]: /Users/dahiribrahim/Desktop/Deedax/orbitalsai/omnivoice-server/omnivoice_server/services/metrics.py:11
[g-app]: /Users/dahiribrahim/Desktop/Deedax/orbitalsai/omnivoice-vllm-experiments/vllm_gateway/app.py:69
