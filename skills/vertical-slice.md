---
name: vertical-slice
description: Use this skill when the user asks to create, generate, build, or scaffold a new API endpoint, feature, or vertical slice in an ASP.NET Core project using Minimal APIs.
---

# Vertical Slice Generator

## Output Rule (No Yapping)
Output strictly what is requested in each step. Do NOT output conversational filler, polite introductions, explanations of the code, or post-generation summaries.

## Architecture & File Structure Rules
- **Access Modifiers:** All generated types (the main slice class, Request/Response DTOs, Validator, Endpoint, and Handler) MUST be strictly `internal`. Do NOT use `public` for types. The ONLY exception is the endpoint mapping method (e.g., `public void MapEndpoint` or `public static void MapEndpoint`), which must be `public`.
- **Single File:** The entire feature (Request, Response, Validator, Endpoint, and optionally Handler) MUST be enclosed within a single `internal static class FeatureName`.
- **MediatR is FORBIDDEN:** Do NOT use `IMediator` or `IRequestHandler`.
- **No Repositories:** Do NOT use the Repository pattern. Inject the specific application DbContext (e.g., `AppDbContext`), NEVER the base `Microsoft.EntityFrameworkCore.DbContext`. Obtain the exact DbContext class name from the project's `AGENTS.md` context or the user's prompt.
- **Dependency Injection & Auto-Registration:** You MUST explicitly use the `[FromServices]` attribute for ALL application services injected into the Minimal API `Handle` method parameters (e.g., `[FromServices] IValidator<Request> validator`, `[FromServices] AppDbContext db`, `[FromServices] Handler handler`). **CRITICAL EXCEPTION:** Do NOT use `[FromServices]` on framework-provided parameters such as `CancellationToken`, `HttpContext`, or `ClaimsPrincipal` — they are resolved automatically. Do NOT rely on implicit DI resolution for your custom services. Check AGENTS.md for the project's specific DI registration rules. If the project uses custom marker interfaces for auto-registration (e.g., IScopedType), ensure the Handler (if used) explicitly implements them. Do NOT implement these marker interfaces on the Validator. If there are no auto-registration rules in AGENTS.md, do not implement any extra interfaces on the Handler.
- **Async & Cancellation:** All endpoint and Handler methods MUST be asynchronous. The Endpoint's `Handle` method MUST return `Task<IResult>`. The `Handler` (if used) MUST NOT return `IResult` directly; it should return a domain outcome (e.g., a DTO or a Result pattern) as defined by the project's `AGENTS.md`. You MUST inject a `CancellationToken` into the entry point and pass it down to ALL asynchronous operations (especially EF Core database calls).
- **Fail-first (Early Return) Pattern:** You MUST use guard clauses and handle all error cases, null checks, and failures first across ALL methods (Endpoints, Handlers, etc.), including **every intermediate operation inside a Handler** — not only the entry-point validation. The rule is: **check for the negative/failure condition, return early, then continue**. NEVER invert this: do NOT write `if (operation succeeded) { return success; }`. Return the corresponding failure immediately and let the success path fall through to the last line. The successful outcome MUST be the very last statement of the method.

## Context Fallbacks & Dependencies
- **DbContext Fallback:** If `AGENTS.md` is missing or you cannot find there exact `DbContext` implementation, stop and ask the user.
- **Mandatory Usings:** You MUST include `using FluentValidation;` and `using FluentValidation.Results;` (the latter is strictly required for the `.ToDictionary()` extension method).
- **Custom Interfaces:** Read interface namespaces from `AGENTS.md`. If absent, ask the user.
- **Result Pattern & Error Handling:** The specific implementation of the Result pattern (e.g., `ErrorOr`, `FluentResults`, or custom wrappers) and how to map them to HTTP status codes MUST be obtained from the project's `AGENTS.md`. If missing, do NOT use a Result pattern. Return DTOs directly from the Handler. In the Endpoint, handle failure by checking for null (e.g., `if (response is null) return Results.NotFound();`). Do NOT blindly return HTTP 200 OK if a Handler returns a Result object containing an error. **CRITICAL: Only use the Result pattern when interacting with a `Handler`. For Simple Logic (Skeleton 1) where you query the DbContext directly, do NOT wrap the outcome in a Result pattern; just return the DTO directly (e.g., `Results.Ok(dto)`).**
- **Result Propagation (Failure pass-through):** *(Apply only if the project uses a Result pattern as defined in `AGENTS.md`.)* When propagating a failure from a nested call, return the failed result as-is. Do NOT re-wrap it into a new failure object — that double-wraps the error. Construct a new failure object only when originating a new error at that call site, not when propagating one from below.

## Logic Placement Rule (Endpoint vs. Handler)
You must decide where to put the business logic based on its complexity:
- **Simple Logic (Keep in Endpoint):** If the feature consists of basic CRUD operations, 1-2 straightforward database queries, and simple mapping. If the flow is just "read request -> query DB -> return response", write the logic DIRECTLY inside the `private static async Task<IResult> Handle(...)` method of the nested `Endpoint` class.
- **Complex Logic (Extract to Handler):** Extract the logic into a nested `Handler` class ONLY IF the feature requires complex business rules, deeply nested conditions, explicit database transactions, or needs to inject multiple external dependencies (e.g., Email services, external APIs) alongside the `DbContext`. Inject this `Handler` into the `Endpoint.Handle` method.

## Component Specifications
- **Endpoint & Routing:** Create a nested `internal sealed class Endpoint`. Check AGENTS.md or the Reference Search to see if endpoints MUST implement a specific interface (e.g., `IEndpoint`, `ICarterModule`). If so, implement it and map the route accordingly. If not, use standard static method extensions for mapping. You MUST use standard RESTful HTTP methods (GET, POST, PUT, DELETE). Route paths MUST be in `kebab-case` and logically structured (e.g., `/api/user-profiles`). NEVER use PascalCase or camelCase in the URL path. Append `.WithTags("FeatureGroup")` to the endpoint mapping to organize OpenAPI documentation. Infer the tag name from the domain context.
- **Responses & DTOs:** NEVER return raw Domain Entities directly to the client. Always **manually** map complex types to DTOs (e.g., a nested `Response` record) before returning them from the endpoint or handler. Do NOT use AutoMapper, Mapster, or any other mapping libraries. **CRITICAL EXCEPTION:** If the endpoint returns only a single value (e.g., a single string token, an int ID, List<T> items, or a bool), do NOT wrap it in a Response DTO record. Return the type directly (e.g., Results.Ok(token)).
- **Request Binding:** You MUST strictly follow Minimal API binding rules based on the HTTP method. Use `[FromBody]` for POST, PUT, and PATCH requests. You MUST use `[AsParameters]` for GET and DELETE requests when using a complex `Request` record, because Minimal APIs cannot bind complex types from the request body for these methods by default.
- **HTTP Status Codes:** You MUST return appropriate HTTP status codes based on the RESTful operation and the execution outcome:
  - Return `Results.Ok(...)` (200) for successful GET requests or when returning modified data.
  - Return `Results.Created(...)` (201) for successful POST requests that create a new resource.
  - Return `Results.NoContent()` (204) for successful DELETE or PUT/PATCH requests that do not return a body.
  - Return `Results.NotFound()` (404) if a requested entity does not exist in the database.
  - Return `Results.Conflict(...)` (409) or `Results.BadRequest(...)` (400) for domain rule violations.
  - In .Produces() methods use StatusCodes static class, **DO NOT USE HARDCODED STATUS CODE NUMBERS.**
- **Validation:** If the endpoint accepts user input, create a nested `internal sealed class Validator : AbstractValidator<Request>` using FluentValidation. You MUST inject the Validator as its interface `IValidator<Request>` (NEVER the concrete `Validator` class) and call `.ValidateAsync(request, ct)`. If validation fails, map the errors using: `return Results.ValidationProblem(validationResult.ToDictionary());`. When validating nested required objects: first assert `.NotNull()` on the parent property, then chain nested rules directly with `.ChildRules(...)` or `.SetValidator(...)`. Do NOT wrap nested rules in `.When(x => x.NestedObject is not null)` — the `NotNull()` check already communicates the requirement; the `When` guard is redundant noise and obscures intent.
- **Exception handling:** `catch { }` block must have logging. If logging method not specified in `AGENTS.md` use basic `Console.WriteLine()`.

## Required File Structure (Skeleton)
You MUST structure the file exactly like one of the following templates. Choose the appropriate skeleton based on the Logic Placement Rule.

### Skeleton 1: Simple Logic (No Handler)
Use this when the logic is simple enough to reside directly in the Endpoint. Do NOT generate a Handler class.

```csharp
using FluentValidation;
using FluentValidation.Results;
// Include other necessary usings (EF Core, Domain entities, Custom Interfaces)

namespace Inferred.Namespace.Here;

internal static class FeatureName
{
    internal sealed record Request(string Data);

    // Create a Response record ONLY if returning multiple fields. 
    // If returning a single (e.g., string token, int Id, List<T> Items), delete this record and return the type directly.
    internal sealed record Response(string Result);

    internal sealed class Validator : AbstractValidator<Request>
    {
        public Validator()
        {
            // RuleFor(x => x.Data)...
        }
    }

    // Implement custom interface here IF required by the project (e.g., : IEndpoint). In than case Endpoint class must be 'internal sealed class Endpoint : IEndpoint'.
    internal static class Endpoint
    {
        // IF using an interface, change the map method to 'public void MapEndpoint(IEndpointRouteBuilder app)'.
        public static void MapEndpoint(this IEndpointRouteBuilder app)
        {
            app.MapPost("/api/feature-name", Handle)
                // adjust type if no Response record
                .Produces<Response>()
                .ProducesValidationProblem()
                .ProducesProblem(StatusCodes.Status404NotFound)
                .ProducesProblem(StatusCodes.Status409Conflict)
                .WithTags("FeatureGroup");
        }

        private static async Task<IResult> Handle(
            [FromBody] Request request, // Use [AsParameters] instead if this is a GET/DELETE request
            [FromServices] IValidator<Request> validator,
            [FromServices] AppDbContext db,
            CancellationToken cancellationToken)
        {
            ValidationResult validationResult = await validator.ValidateAsync(request, cancellationToken);
            if (!validationResult.IsValid)
            {
                return Results.ValidationProblem(validationResult.ToDictionary());
            }

            // Write simple DbContext query and logic here
            // var entity = await db.Entities.FirstOrDefaultAsync(...);

            // FAIL-FIRST check
            // if (entity is null)
            // {
            //     return Results.NotFound();
            // }

            // SUCCESS: Match the HTTP status code to the REST operation.
            // Choose ONE return based on the HTTP method:
            // POST (Create):  return Results.Created();
            // GET/PUT:        return Results.Ok(response);
            // DELETE:         return Results.NoContent();
            return Results.Created(); // ← активный, замени если нужно
        }
    }
}
```

### Skeleton 2: Complex Logic (With Handler)
Use this when the logic is complex and requires a separate Handler.

```csharp
using FluentValidation;
using FluentValidation.Results;
// Include other necessary usings (EF Core, Domain entities, Custom Interfaces)

namespace Inferred.Namespace.Here;

internal static class FeatureName
{
    internal sealed record Request(string Data);

    // Create a Response record ONLY if returning multiple fields. 
    // If returning a single (e.g., string token, int Id, List<T> Items), delete this record and return the type directly.
    internal sealed record Response(string Result);

    internal sealed class Validator : AbstractValidator<Request>
    {
        public Validator()
        {
            // RuleFor(x => x.Data)...
        }
    }

    // Implement custom interface here IF required by the project (e.g., : IEndpoint). In than case Endpoint class must be 'internal sealed class Endpoint : IEndpoint'.
    internal static class Endpoint
    {
        // IF using an interface, change the map method to 'public void MapEndpoint(IEndpointRouteBuilder app)'.
        public static void MapEndpoint(this IEndpointRouteBuilder app)
        {
            app.MapPost("/api/feature-name", Handle)
                // adjust type if no Response record
                .Produces<Response>()
                .ProducesValidationProblem()
                .ProducesProblem(StatusCodes.Status404NotFound)
                .ProducesProblem(StatusCodes.Status409Conflict)
                .WithTags("FeatureGroup");
        }

        private static async Task<IResult> Handle(
            [FromBody] Request request, // Use [AsParameters] instead if this is a GET/DELETE request
            [FromServices] IValidator<Request> validator,
            [FromServices] Handler handler, 
            CancellationToken cancellationToken)
        {
            ValidationResult validationResult = await validator.ValidateAsync(request, cancellationToken);
            if (!validationResult.IsValid)
            {
                return Results.ValidationProblem(validationResult.ToDictionary());
            }

            Response response = await handler.HandleAsync(request, cancellationToken);
            
            // FAIL-FIRST: Handle errors before success
            
            // If using a Result pattern (per AGENTS.md), check for failure here using
            // the project's Result API and return the appropriate HTTP error response
            // before reaching the success path.
            
            // If returning plain DTOs, handle edge cases (e.g., nulls):
            // if (response is null)
            // {
            //     return Results.NotFound();
            // }

            // SUCCESS: Match the HTTP status code to the REST operation.
            // If using a Result pattern, unwrap the value per the project's Result API (per AGENTS.md).
            // Otherwise use response directly.
            
            /// Choose ONE return based on the HTTP method:
            // POST (Create):  return Results.Created();
            // GET/PUT:        return Results.Ok(response);
            // DELETE:         return Results.NoContent();
            return Results.Created(); // ← активный, замени если нужно
        }
    }
    // Implement DI marker interface here IF required by AGENTS.md (e.g., : IScopedType)
    internal sealed class Handler(AppDbContext db)
    {
        // If the project uses a Result pattern, update the return type accordingly (per AGENTS.md).
        public async Task<Response> HandleAsync(Request request, CancellationToken cancellationToken)
        {
            // Complex domain logic and transaction handling here
            
            return new Response("Processed");
        }
    }
}
```

## Workflow & Execution

### Step 1 — Context Gathering
Before writing any code, you MUST read the `AGENTS.md` file (if it exists in the project). Explicitly extract the exact `DbContext` name, the `Result` pattern wrapper (e.g., ErrorOr, FluentResults, custom), custom interfaces, and namespace conventions. Do NOT guess these values.

### Step 2 — Reference Search
Before searching, identify exactly what you do NOT yet know after reading `AGENTS.md`: specific naming conventions, Result pattern usage, or mapping style. Then read the **minimum number of existing feature files** needed to answer only those specific open questions — targeting files of similar complexity to the requested feature. Do NOT read files to confirm what you already know. Do NOT explore the Features directory broadly. Each file read requires a stated justification: *"I am reading this file because I still don't know X."* Stop as soon as all open questions are answered.

### Step 3 — Plan
Analyze the request and decide if a separate Handler class is needed based on complexity. Output the plan, including:
- The target file path for the new slice.
- The chosen skeleton (Simple or Complex).
- The derived branch name (see Step 4 for naming rules).

### Step 4 — Create Feature Branch
The FIRST action before writing any code is to create and switch to a new Git branch. If GitHub MCP is unavailable, ask the user before proceeding — do NOT create the branch via CLI alone, as the full procedure requires MCP.

**Branch naming rules:**
- Format: `feature/<action>-<domain>` in kebab-case.
- Derive `<action>` from the slice file name (e.g., `Create.cs` → `create`, `GetById.cs` → `get-by-id`).
- Derive `<domain>` from the feature folder name (e.g., `Admin` → `admin`, `UserProfiles` → `user-profiles`).
- Example: slice `Create.cs` inside folder `Admin` → branch `feature/create-admin`.

**Branch creation procedure:**
1. Read the base branch name from `AGENTS.md` (required fields: Develop branch name, GitHub Owner, GitHub Repo). If absent, ask the user before proceeding. Do NOT fall back to `main` or `master` silently.
2. Use the GitHub MCP tool to retrieve the current SHA of the base branch HEAD.
3. Use the GitHub MCP tool to create the new branch from that SHA.
4. Using the **git CLI**: run `git fetch origin && git checkout feature/<branch-name>` to switch to the newly created remote branch locally. Do NOT use `git checkout -b` — the branch already exists on the remote and `-b` will fail. If the checkout fails, STOP and report the error to the user.

### Step 5 — Generate
Write the complete feature code in a single file following all rules above.

### Step 6 — Save & Build (STRICT ORDER)
1. Save the generated file to the appropriate feature directory.
2. Verify the code compiles by running the build command on the specific project file (NOT the whole solution unless necessary). You MUST use this exact command to avoid context overflow:
`dotnet build <PathToTargetProject.csproj> /clp:ErrorsOnly -v q`
If there are compiler errors (missing using directives, type name mismatches, etc.), read the concise error output, fix the code, save the file, and recompile. 
**Maximum 3 attempts.** If the build still fails after the 3rd attempt, you MUST STOP immediately. Do NOT make further code changes, do NOT commit, do NOT push. Output the final compilation error to the user for manual intervention.

### Step 7 — Commit & Push
Proceed to this step ONLY if Step 6 succeeded with zero build errors.
1. Using the **git CLI** (not GitHub MCP): stage and commit the file directly to the new feature branch with a clear, conventional commit message. Use the format: `feat(<domain>): add <action> slice`.
2. Using the **git CLI**: push the branch to the remote origin. If the push fails, STOP and report the error.

### Step 8 — Create Pull Request
After a successful push, use the GitHub MCP tool to create a Pull Request:
- **Base branch:** the base branch read from `AGENTS.md` in Step 4.
- **Head branch:** the feature branch created in Step 4.
- **Title:** `feat(<domain>): <action> slice` (e.g., `feat(admin): create slice`).
- **Body:** a short description of what the slice does, which HTTP method and route it exposes, and whether a Handler was used.

Output only the PR URL. Do NOT add conversational filler.
