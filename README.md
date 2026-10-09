# BaseProject

A reusable .NET 8 Web API starter built around Clean Architecture, CQRS, PostgreSQL, Redis, and Docker.

## Solution structure

```text
src/
  BaseProject.Api/             HTTP endpoints, authentication, middleware
  BaseProject.Application/     CQRS requests, validation, contracts, behaviors
  BaseProject.Domain/          Domain entities and rules
  BaseProject.Infrastructure/  EF Core, Dapper, PostgreSQL, Redis
```

Dependency direction: API -> Application; Infrastructure -> Application and Domain; Application -> Domain.

## Included foundations

- `.editorconfig`, `Directory.Build.props`, and `Directory.Packages.props` for naming, compiler defaults, and centrally versioned NuGet packages.
- MediatR CQRS with validation, request logging, slow-request warnings, and opt-in transaction behaviors.
- Shared `Result<T>`/`Error` types and an `ApiResponse<T>` response envelope.
- Global exception/status-code responses: validation 400, authentication 401, authorization 403, not found 404, conflict/concurrency 409, and sanitized 500.
- FluentValidation validators for Product create/update/delete.
- PostgreSQL persistence through EF Core for writes and Dapper for Product list reads.
- Audit fields (`CreatedBy`, `CreatedDate`, `UpdatedBy`, `UpdatedDate`) and centrally applied soft-delete fields.
- `RepositoryBase<TEntity>` and `UnitOfWork`.
- JWT bearer validation with `Products.Read` and `Products.Write` role policies.
- Redis-backed Product list caching with version-based invalidation after a committed write.
- Serilog console and daily rolling file sinks (`logs/log-YYYYMMDD.txt`).
- Product CRUD sample with paginated list reads and Docker Compose for API, PostgreSQL, and Redis.

## Prerequisites

- .NET 8 SDK
- Docker Desktop with Compose
- A JWT issuer that signs tokens with the configured issuer, audience, and signing key. The starter validates tokens; it does not issue or manage user credentials.

## Run with Docker Compose

```powershell
Copy-Item .env.example .env
# Edit .env and set unique local POSTGRES_PASSWORD and JWT_SIGNING_KEY values (at least 32 UTF-8 bytes).
docker compose up --build
./scripts/smoke-test.ps1
```

The API is available at <http://localhost:8080>; copying `.env.example` enables Swagger for local development. For deployments, set `ASPNETCORE_ENVIRONMENT=Production` in `.env` or the deployment environment. `GET /health/live` checks the process; `GET /health/ready` checks PostgreSQL and Redis. Compose keeps file logs in the `api_logs` volume.

The `.env.example` values are local placeholders only. Never use them outside an isolated development environment; provide production credentials through a secret manager.

## Run the API directly

Start PostgreSQL and Redis, then configure these user secrets or environment variables:

```powershell
dotnet build BaseProject.sln
dotnet user-secrets init --project src\BaseProject.Api
dotnet user-secrets set "ConnectionStrings:Postgres" "Host=localhost;Port=5432;Database=baseproject;Username=baseproject;Password=your-local-password" --project src\BaseProject.Api
dotnet user-secrets set "ConnectionStrings:Redis" "localhost:6379" --project src\BaseProject.Api
dotnet user-secrets set "Jwt:SigningKey" "replace-with-at-least-32-random-utf8-bytes" --project src\BaseProject.Api
dotnet run --project src\BaseProject.Api
```

Development and the single API container in Compose apply EF Core migrations at startup. Production defaults to no automatic migration; run migrations as a separate deployment step before starting or scaling API replicas. Add and commit a migration whenever the model changes:

```powershell
dotnet tool restore
dotnet ef migrations add DescriptiveChange --project src\BaseProject.Infrastructure --startup-project src\BaseProject.Api --output-dir Persistence\Migrations
```

## Product API

All product endpoints require a bearer token with the listed role:

| Method | Route | Role | Purpose |
| --- | --- | --- | --- |
| GET | `/api/products?search=...&page=1&pageSize=20` | `Admin`, `ProductReader`, `ProductManager` | Search and list; Dapper read path, cached in Redis; page size 1–100 |
| GET | `/api/products/{id}` | `Admin`, `ProductReader`, `ProductManager` | Get a product |
| POST | `/api/products` | `Admin`, `ProductManager` | Create a product |
| PUT | `/api/products/{id}` | `Admin`, `ProductManager` | Update; body must include the current `version` |
| DELETE | `/api/products/{id}?version={version}` | `Admin`, `ProductManager` | Soft-delete with optimistic concurrency |

Example create payload:

```json
{
  "name": "Sample product",
  "description": "A product for the starter",
  "price": 12.50
}
```

Successful and failed responses use the `success`, `message`, `data`, and optional `errors` envelope. Errors do not expose exception details in 500 responses.

## Architecture notes

- Commands that implement `ITransactionalRequest` run inside the `UnitOfWork` transaction behavior. Cache version invalidation runs after commit.
- Product `Version` is an EF concurrency token; clients send it back for update/delete. Concurrent database updates return 409.
- `RepositoryBase<TEntity>` is a small write-oriented abstraction. Prefer focused read interfaces and Dapper projections for query-heavy endpoints.
- `ICurrentUserService` obtains the actor from the authenticated claims; unauthenticated/system operations are attributed to `system`.
- Configure issuer/audience and a strong signing key for the JWT issuer. Add identity provisioning/token issuance separately to fit the deployment's identity provider.
- Keep secrets outside source control. `.env` is ignored; configure production secrets with the deployment platform's secret manager.

## Extend the starter

1. Add a domain entity in `Domain` and its EF mapping/migration in `Infrastructure`. For a soft-deletable entity, configure an EF query filter and make reads exclude deleted rows; the Product mapping is the example.
2. Add focused repository interfaces and CQRS requests in `Application`. Register validators; `ValidationBehavior` runs them before handlers.
3. Use `ITransactionalRequest` for commands that write data; call `IUnitOfWork.SaveChangesAsync` in the handler. Add `ICacheInvalidationRequest` only when the command changes cached reads.
4. Expose a thin controller action and map `Result<T>` through `ApiControllerBase`. Add a policy for protected endpoints.
5. Add tests for business rules, persistence behavior, authorization, and response codes. Run `dotnet test BaseProject.sln` before deployment.

For a new deployment, choose an identity provider and set its JWT issuer, audience, and signing key. The sample's symmetric key is for local integration only. Plan the database migration step, secret storage, log collection, and readiness probe for the target platform.
"# base-project" 
