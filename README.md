# Вклад Абубукара Цароева

Снимок файлов, которые я писал и менял в **tricky-exchanger / Меняйка**.

## Вводная

В команде **1 фронтенд и 3 бэкенда**. Я один из бэкендов.

В проект я **пришёл позже остальных**. К этому моменту команда уже сама начала разработку **алгоритма поиска/матчинга цепочек**, поэтому в эту зону я сознательно **не включался**: не перехватывал чужой ownership и не переписывал алгоритм «с нуля». Алгоритм остался зоной коллег; моя зона — архитектура, инфраструктура, качество кода и продуктовые фичи вне matching-ядра (auth, users, items, CI/CD, рефакторинг).

Когда я вошёл в команду, у бэкенда была **слабая архитектура** и **не было общих правил**. Поэтому я взял на себя роль **техлида**: выстроить структуру, зафиксировать договорённости и не давать репо расползаться дальше.

Правила, которые я выставил и за которыми следил:

1. **Layer-first**: `handler` → `service` → `repository` → `entity` (не entity-first каша).
2. **Интерфейсы у потребителя**, узкие по поведению — без fat Repository на десятки методов.
3. **Единые ошибки**: БД через `DBErrToErr`/`MapDBErr`; user-facing API на русском; логи на английском.
4. **Единый HTTP-стиль**: `api.SendOk` / `SendError`, общие helpers (`BindJSON`, current user, path id).
5. **Комментарии только про неочевидное**; длинный rationale → ADR, не простыни в `.go`.
6. **Сначала баги, потом красота**; не тащить DI/query-builder/микрофайлы без нужды.
7. **CI обязателен**: `gofmt`, unit, frontend checks, e2e; в `main` только через проверки.
8. **Мердж в `main` только после ревью**.

Отдельно: я **проверял абсолютно все PR** команды перед мерджем в `main` — следил, чтобы новый код не ломал архитектуру, не плодил дубли helpers/ошибок и не растаскивал стиль.

Список путей в снимке: [`MANIFEST.md`](./MANIFEST.md). Удалённые: [`MISSING_OR_DELETED.txt`](./MISSING_OR_DELETED.txt).

## CI/CD и автодеплой — сделал полностью сам

Весь CI/CD и автодеплой на **VPS + GitHub Actions** настроил я.

### CI (проверки на каждый PR и push в `main`)

Файл: `.github/workflows/ci.yml`

- На **pull_request** и **push в `main`** автоматически гоняются:
  - backend: `gofmt` + `go test ./...`
  - frontend: `npm ci` → typecheck → lint → test
  - e2e: подъём `docker-compose.e2e.yml`, wait на `/healthz`, `go test -tags=e2e`
- Есть concurrency: параллельные прогоны на одном ref отменяются.

То есть после мерджа/пуша в `main` тесты и проверки **не вручную** — пайплайн сам.

### CD / автодеплой на VPS (после `main`)

Файл: `.github/workflows/deploy.yml`  
Триггер: **push в `main`** (+ ручной `workflow_dispatch`).

Цепочка, которую я собрал:

1. **Снова прогон тестов** (backend + frontend + e2e) — деплой не стартует, если красное.
2. **Build & push образов** backend/frontend в **GHCR** (`ghcr.io/.../backend|frontend:latest` и `:sha`).
3. **SSH на VPS** (`VPS_HOST` / `VPS_USER` / `VPS_SSH_KEY`):
   - копирую `docker-compose.prod.yml`, `Caddyfile`, миграции;
   - на сервере `docker login ghcr.io` → `pull` → `migrate` → `up -d`;
   - force-recreate **Caddy**;
   - smoke: `curl https://menyayka.tech/healthz` должен ответить JSON бэкенда, не HTML фронта;
   - prune старых образов.

Прод-окружение: compose + Caddy (TLS/прокси) + `.env` на сервере из `.env.deploy.example`. Деплой-путь: `/opt/menyayka`.

Итого: **мердж в `main` → CI → сборка образов → выкат на VPS** без ручного ssh «на глаз».

---

## Что я сделал в каждом файле

Формат: **путь** — что именно я здесь сделал.

### Корень / инфра

- **`.env.example`** — завёл/поддерживал шаблон локальных env.
- **`.env.deploy.example`** — завёл шаблон прод-env для VPS (секреты только на сервере, не в git).
- **`.gitignore`** — настроил игнор секретов/мусора.
- **`.github/workflows/ci.yml`** — **написал CI полностью**: unit/frontend/e2e на PR и на `main`, плюс `gofmt`.
- **`.github/workflows/deploy.yml`** — **написал автодеплой полностью**: тесты → GHCR → SSH на VPS → migrate/up → healthz.
- **`Caddyfile`** — настроил reverse-proxy/TLS для `menyayka.tech` на VPS.
- **`Makefile`** — завёл единые команды run/migrate/test.
- **`README.md`** — написал/вёл корневую документацию запуска.
- **`docker-compose.yml`** — собрал локальный стек.
- **`docker-compose.prod.yml`** — собрал прод-compose под GHCR-образы и автодеплой на VPS.
- **`docker-compose.e2e.yml`** — собрал стек для CI e2e.
- **`frontend/Dockerfile`** — упаковал фронт в образ.
- **`frontend/.dockerignore`** — ужал контекст сборки фронта.
- **`frontend/nginx.conf`**, **`frontend/nginx.local.conf`** — настроил раздачу фронта.
- **`frontend/README.md`** — описал фронт-запуск.
- **`loadtest/k6/auth_item_flow.js`** — написал нагрузочный сценарий auth+item.
- **`docs/adr/0001-matching-and-search.md`** — вынес длинные matching/search комментарии из кода в ADR.
- **`docs/adr/0002-smtp-encryption.md`** — вынес SMTP/TLS rationale из mailer в ADR.

### Backend bootstrap

- **`backend/README.md`** — описал запуск бэкенда и структуру.
- **`backend/go.mod`**, **`backend/go.sum`** — завёл/обновлял зависимости модуля.
- **`backend/cmd/api/main.go`** — ужал до тонкой точки входа, вынес wiring.
- **`backend/cmd/api/server.go`** — вынес сборку зависимостей/сервера из `main`.

### API / core / middleware

- **`internal/api/response.go`** — сделал единый формат ошибок/успеха (`SendOk`/`SendError`).
- **`internal/api/request.go`** — вынес общие HTTP helpers (`CurrentUser*`, `PathInt64`, `BindJSON`) из хендлеров.
- **`internal/core/config/config.go`** — сделал fail-fast на невалидный env.
- **`internal/core/database/postgres.go`** — завёл подключение/пул Postgres.
- **`internal/core/database/transaction.go`** — завёл TransactionManager.
- **`internal/core/logger/logger.go`** — завёл общий логгер.
- **`internal/core/router/router.go`** — завёл layer-first роутинг и регистрацию фич.
- **`internal/core/router/ping_handler.go`** — добавил ping/health каркас.
- **`internal/middleware/auth.go`** — сделал JWT middleware.

### Entity

- **`entity/user.go`** — завёл/правил модель пользователя под auth.
- **`entity/item.go`** — правил модель товара под CRUD/image/category.
- **`entity/exchange_offer.go`** — правил модель заявки под общий layout (фичу писали коллеги).
- **`entity/chain.go`**, **`entity/chain_draft.go`**, **`entity/vote.go`** — правил сущности под общий layout/ошибки (логика цепочек — коллеги).
- **`entity/errors.go`** — завёл/наращивал доменные ошибки.

### Handler chain — рефакторинг чужого кода

- **`handler/chain/handler.go`** — вырезал монолит, оставил только каркас `Handler`.
- **`handler/chain/read.go`** — вынес list/get из толстого `handler.go`.
- **`handler/chain/vote.go`** — вынес vote/confirm/unconfirm/think/decline из `handler.go` + выровнял стиль/ошибки.
- **`handler/chain/fulfillment.go`** — вынес handoff/receipt из `handler.go`.
- **`handler/chain/response.go`** — вынес DTO ответов из монолита.
- **`handler/chain/errors.go`** — вынес маппинг ошибок и перевёл user-facing сообщения на русский.

### Handler exchange_offer — рефакторинг чужого кода (Linempy)

- **`handler/exchange_offer/handler.go`** — ужал до каркаса после сплита.
- **`handler/exchange_offer/read.go`** — вынес list/get из монолита.
- **`handler/exchange_offer/write.go`** — вынес create/update/delete из монолита.
- **`handler/exchange_offer/request.go`** — вынес request DTO.
- **`handler/exchange_offer/response.go`** — вынес response DTO и перевёл ответы на `api.SendOk/SendError`.

### Handler item — писал сам

- **`handler/item/handler.go`** — написал каркас Handler.
- **`handler/item/contracts.go`** — написал интерфейс сервиса для хендлера.
- **`handler/item/read.go`** — написал read-ручки.
- **`handler/item/write.go`** — написал write-ручки.
- **`handler/item/request.go`** — написал request DTO.
- **`handler/item/response.go`** — написал response DTO.
- **`handler/item/errors.go`** — написал маппинг ошибок хендлера.

### Handler user — писал сам

- **`handler/user/handler.go`** — написал каркас Handler.
- **`handler/user/contracts.go`** — написал интерфейс сервиса для хендлера.
- **`handler/user/auth.go`** — написал register/login/logout/me.
- **`handler/user/password.go`** — написал смену пароля.
- **`handler/user/recovery.go`** — написал recovery-флоу.
- **`handler/user/response.go`** — написал ответы user API.

### Repository common + chain

- **`repository/errors.go`** — реализовал `DBErrToErr`, добавил `MapDBErr`.
- **`repository/chain/repository.go`** — сжал монолит до `Postgres` + `NewRepository`, убрал обратный `var _` на service.
- **`repository/chain/candidates.go`** — вынес `SaveCandidates` из монолита.
- **`repository/chain/listing.go`** — вынес list/get/`HasDeadlineEvent`, подчистил комменты.
- **`repository/chain/votes.go`** — вынес vote-методы, подчистил комменты.
- **`repository/chain/proposal.go`** — вынес propose/confirm/decline/expire; вернул куски логики, потерянные при сплите; убрал лишние комменты.
- **`repository/chain/replacement.go`** — вынес replacement-методы; подключил общий eligibility SQL.
- **`repository/chain/replacement_sql.go`** — вынес общий eligibility SQL и починил битые `$4`/`$7`.
- **`repository/chain/freeze.go`** — вынес freeze/release; вернул `invalid_reason = NULL`; убрал лишние комменты.
- **`repository/chain/fulfillment.go`** — вынес handoff/receipt/complete.
- **`repository/chain/lifecycle.go`** — вынес delete/list для matcher.

### Repository exchange_offer — рефакторинг чужого кода

- **`repository/exchange_offer/repository.go`** — ужал до каркаса после сплита.
- **`repository/exchange_offer/create.go`** — вынес create.
- **`repository/exchange_offer/read.go`** — вынес read/list; подчистил комменты.
- **`repository/exchange_offer/mutate.go`** — вынес update/delete; подчистил комменты.
- **`repository/exchange_offer/helpers.go`** — вынес helpers и провёл ошибки через `MapDBErr`.

### Repository cluster / search / item / user

- **`repository/cluster/repository.go`** — ужал до каркаса, убрал лишние комменты.
- **`repository/cluster/create.go`** — вынес create из монолита, убрал лишние комменты.
- **`repository/cluster/find.go`** — вынес find, убрал лишние комменты.
- **`repository/cluster/members.go`** — вынес members, убрал лишние комменты.
- **`repository/cluster/membership.go`** — вынес membership, убрал лишние комменты.
- **`repository/cluster/refresh.go`** — вынес refresh, убрал лишние комменты.
- **`repository/cluster/vectors.go`** — вынес vectors, убрал лишние комменты.
- **`repository/search/repository.go`** — ужал до каркаса, вычистил простыню комментариев.
- **`repository/search/find.go`** — вынес find, убрал лишние комменты.
- **`repository/search/frontier.go`** — вынес frontier, убрал лишние комменты.
- **`repository/search/queries.go`** — вынес SQL queries, сильно почистил комменты (часть → ADR).
- **`repository/search/scan.go`** — вынес scan helpers, убрал лишние комменты.
- **`repository/search/vector.go`** — правил vector-часть, убрал лишние комменты.
- **`repository/item/repository.go`** — правил item repo + выравнивание `DBErrToErr`/комментов.
- **`repository/user/repository.go`** — писал/правил user repo под auth + чистка комментов.

### Service chain

- **`service/chain/contracts.go`** — разделил большой `Repository` на подинтерфейсы (`CandidateStore`, `VoteStore`, `ProposalStore`, `FreezeStore`, …).
- **`service/chain/service.go`** — оставил каркас после сплита.
- **`service/chain/freeze_service.go`** — сузил зависимость с `Repository` до `FreezeStore`.
- **`service/chain/candidates.go`** — вынес save-candidates сценарий из монолита.
- **`service/chain/read.go`** — вынес list/get сценарии.
- **`service/chain/votes.go`** — вынес vote/withdraw.
- **`service/chain/proposal.go`** — вынес confirm/unconfirm/think/decline.
- **`service/chain/replacement.go`** — вынес replacement-сценарии.
- **`service/chain/fulfillment.go`** — вынес handoff/receipt.
- **`service/chain/lifecycle.go`** — вынес lifecycle-методы для matcher.
- **`service/chain/score.go`** — вынес score/ranker вызовы.

### Service exchange_offer — рефакторинг чужого кода

- **`service/exchange_offer/service.go`** — выровнял под общий layout/ошибки; сценарии заявок не писал.

### Service item / user / cluster / matching

- **`service/user/service.go`** — писал/правил auth/recovery бизнес-логику.
- **`service/user/contracts.go`** — завёл/правил контракты user service.
- **`service/item/service.go`** — оставил каркас после сплита, убрал лишние комменты.
- **`service/item/contracts.go`** — правил контракты, убрал лишние комменты.
- **`service/item/create.go`** — вынес create.
- **`service/item/read.go`** — вынес read/pagination.
- **`service/item/mutate.go`** — вынес update/delete.
- **`service/item/embed.go`** — вынес embedding-часть.
- **`service/item/image.go`** — вынес image-часть, убрал лишние комменты.
- **`service/cluster/service.go`** — правил вызовы под сплит repo/контракты.
- **`service/cluster/contracts.go`** — правил/сужал контракты cluster.
- **`service/matching/cycle_finder.go`** — ужал точку входа после сплита.
- **`service/matching/find.go`** — вынес find из монолита.
- **`service/matching/graph.go`** — вынес graph helpers.
- **`service/matching/dedupe.go`** — вынес dedupe.
- **`service/matching/validator.go`** — правил validator, вычистил лишние комменты.

### Infrastructure / pkg

- **`infrastructure/mailer/mailer.go`** — правил mailer; длинный TLS-текст вынес в ADR, сократил комменты.
- **`infrastructure/reservation/stub_checker.go`** — правил stub под общий layout.
- **`pkg/token/token.go`** — написал JWT issue/parse.
- **`pkg/codestore/codestore.go`** — написал/правил хранилище recovery-кодов.
- **`pkg/storage/minio_storage.go`** — правил MinIO storage под item images.
- **`pkg/validator/validator.go`** — сделал единый BindJSON/validation.
- **`pkg/validator/custom.go`** — добавил custom-валидации.
- **`pkg/validator/error.go`** — унифицировал ошибки валидатора.
- **`pkg/utils/ranker/ranker_state.go`** — подчистил лишние комменты.

### Migrations

- **`migrations/000001_*`** — добавил enable pgvector.
- **`migrations/000002_*`** — добавил таблицу users.
- **`migrations/000006_*`** — добавил image для item.
- **`migrations/000009_*`**, **`000010_*`** — перевёл item category на text.
- **`migrations/000011_*`**, **`000012_*`** — перевёл wanted category offer на text.
- **`migrations/000013_*`** — добавил unique living offer.

### Tests

- **`tests/api/request_test.go`** — написал тесты на HTTP helpers.
- **`tests/core/config/ranker_mode_test.go`** — перенёс тест конфига в `tests/` и правил под fail-fast.
- **`tests/db/postgres_test.go`** — правил/поддерживал DB-тесты.
- **`tests/router/router_test.go`** — написал/правил тесты роутера.
- **`tests/e2e/e2e_test.go`** — завёл/правил e2e.
- **`tests/handler/user/*`** — написал тесты auth/recovery/password.
- **`tests/handler/item/handler_test.go`**, **`write_test.go`** — писал/дописывал тесты item handler под сплит.
- **`tests/handler/exchange_offer/handler_test.go`**, **`write_test.go`** — подогнал тесты под сплит чужого handler.
- **`tests/handler/chain/handler_test.go`** — подогнал тесты под сплит.
- **`tests/handler/chain/errors_test.go`** — добавил тесты маппинга ошибок.
- **`tests/handler/chain/vote_decline_test.go`** — добавил тесты decline-ветки.
- **`tests/service/user/user_service_test.go`** — написал/правил тесты user service.
- **`tests/service/item/*`** — писал/дописывал тесты после сплита item.
- **`tests/service/exchange_offer/*`** — подогнал тесты под рефакторинг чужого service.
- **`tests/service/cluster/*`**, **`matching/*`** — правил тесты под сплит/чистки.
- **`tests/service/chain/lifecycle_test.go`** — добавил тесты lifecycle после сплита.
- **`tests/repository/errors_test.go`** — написал тесты `DBErrToErr`/`MapDBErr`.
- **`tests/repository/chain/replacement_test.go`** — добавил контракт-тесты eligibility SQL.
- **`tests/repository/chain/repository_test.go`** — добавил smoke после сплита.
- **`tests/repository/cluster/*`**, **`exchange_offer/*`**, **`item/*`**, **`search/*`**, **`user/*`** — добавил/правил smoke после сплитов.
- **`tests/infrastructure/*`**, **`tests/pkg/*`** — правил/перенёс тесты mailer/codestore/validator/ranker в `tests/`.

---

## Последний большой рефакторинг — что делал

1. Починил SQL placeholders в replacement и вынес общий SQL.
2. Разрезал толстые `handler`/`repository`/`service` по файлам ответственности.
3. Разделил fat `Repository` в `service/chain/contracts.go` на подинтерфейсы.
4. Вынес HTTP helpers в `api/request.go`, убрал дубли.
5. Вычистил лишние комментарии, длинное → ADR.
6. Добавил `gofmt` в CI и fail-fast config.
7. Не делал: DI framework, query builder, микрофайлы на каждый метод, тотальный uuid-рефактор.
