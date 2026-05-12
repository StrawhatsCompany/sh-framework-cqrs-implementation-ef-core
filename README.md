# SH.Framework.Library.Cqrs.Implementation.EntityFrameworkCore

[![NuGet Version](https://img.shields.io/nuget/v/SH.Framework.Library.Cqrs.Implementation.EntityFrameworkCore.svg)](https://www.nuget.org/packages/SH.Framework.Library.Cqrs.Implementation.EntityFrameworkCore/)
[![NuGet Downloads](https://img.shields.io/nuget/dt/SH.Framework.Library.Cqrs.Implementation.EntityFrameworkCore.svg)](https://www.nuget.org/packages/SH.Framework.Library.Cqrs.Implementation.EntityFrameworkCore/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

Entity Framework Core bindings for [SH.Framework.Library.Cqrs.Implementation](https://www.nuget.org/packages/SH.Framework.Library.Cqrs.Implementation/). Adds `IQueryable<T>` extension methods that translate a `ListRequest<TResponse>` — filters, ordering, paging — into a single EF Core query and return a paged `ListResult<T>`.

If you're already using the CQRS Result pattern from the parent package and want list/query endpoints that "just work" against a `DbSet<T>`, this is the missing piece.

## 🚀 Features

- **📋 `ListRequest<T>` → EF Core query** — one call to apply filters, sort orders, and paging to any `IQueryable<TEntity>`.
- **🔍 Rich filter operators** — `Equals`, `NotEquals`, `GreaterThan`/`Less...`, `Contains` / `StartsWith` / `EndsWith` (+ their `Not` variants), `Between`, `IsNull` / `IsNotNull`, `IsEmpty` / `IsNotEmpty`, `In` / `NotIn`.
- **🧭 Multi-key sorting** — `ListRequestOrder` entries chain as `OrderBy → ThenBy` / `OrderByDescending → ThenByDescending`.
- **📦 Built-in paging** — page size clamped to `[1, 1000]` (default `10`), 1-based page numbers.
- **🎯 Projection-aware** — `ToListResultAsync(query, request, selector)` applies the projection after filtering/sorting so SQL stays minimal.
- **🛡️ Safe property resolution** — reflection-based property lookup is case-insensitive and silently skips fields that don't exist on the entity (no runtime crash from a stale UI request).
- **🧩 Multi-target** — net8.0, net9.0, net10.0; EF Core 8 / 9 / 10 respectively.

## 📦 Installation

```bash
dotnet add package SH.Framework.Library.Cqrs.Implementation.EntityFrameworkCore
```

> This package transitively brings in `SH.Framework.Library.Cqrs.Implementation` (for `ListRequest`, `ListResult`, `Result<T>`) and the matching `Microsoft.EntityFrameworkCore` major.

## 🛠️ Quick start

### 1. Define a list request slice

```csharp
using SH.Framework.Library.Cqrs.Implementation;

public sealed class ListOrdersQuery : ListRequest<OrderResponse> { }

public sealed record OrderResponse(Guid Id, string Customer, decimal Total, DateTime CreatedAt);
```

`ListRequest<TResponse>` already carries `Page`, `PageSize`, `Filters`, and `Orders` — your slice just inherits.

### 2. Wire it up in the handler

```csharp
using Microsoft.EntityFrameworkCore;
using SH.Framework.Library.Cqrs.Implementation;
using SH.Framework.Library.Cqrs.Implementation.EntityFrameworkCore;

public sealed class ListOrdersHandler(AppDbContext db)
    : RequestHandler<ListOrdersQuery, ListResult<OrderResponse>>
{
    public override async Task<Result<ListResult<OrderResponse>>> HandleAsync(
        ListOrdersQuery req, CancellationToken ct = default)
    {
        var result = await db.Orders
            .AsNoTracking()
            .ToListResultAsync(
                req,
                o => new OrderResponse(o.Id, o.Customer, o.Total, o.CreatedAt),
                ct);

        return Result.Success(result);
    }
}
```

That single `ToListResultAsync(req, selector, ct)` does:

1. `ApplyFilters` — translates each `ListRequestFilter` into a `Where` clause.
2. `ApplyOrders` — chains `OrderBy` / `ThenBy` for each `ListRequestOrder`.
3. `CountAsync` against the filtered (but un-paged, un-projected) query.
4. `Skip / Take` for paging, then `Select(selector)`, then `ToListAsync`.
5. Returns `ListResult<OrderResponse>` with `Items`, `TotalCount`, `Page`, `PageSize`.

### 3. Send a query

```jsonc
POST /api/v1/orders/list
{
  "page": 1,
  "pageSize": 25,
  "orders": [
    { "field": "CreatedAt", "direction": "Desc" }
  ],
  "filters": [
    { "field": "Customer",  "operator": "Contains",    "value": "acme" },
    { "field": "Total",     "operator": "Between",     "value": "100,5000" },
    { "field": "CreatedAt", "operator": "GreaterThan", "value": "2025-01-01" }
  ]
}
```

## 🧱 API surface

All methods are extensions on `IQueryable<TEntity>`:

| Method | Purpose |
|---|---|
| `.Apply(request)` | Filters + orders, no paging, no materialization. Useful when you need to chain extra LINQ before counting. |
| `.ApplyWithPaging(request)` | Filters + orders + `Skip/Take`. Still deferred. |
| `.ApplyFilters(filters)` | Filters only. |
| `.ApplyOrders(orders)` | Orders only. |
| `.ApplyPaging(page, pageSize)` | Paging only (1-based; page size clamped to `[1, 1000]`). |
| `.ToListResultAsync(request, ct)` | Returns `ListResult<TEntity>` — entities, no projection. |
| `.ToListResultAsync(request, selector, ct)` | Returns `ListResult<TResponse>` — projection applied after filter + sort. |

## 🔎 Filter operator reference

| Operator | Applies to | Notes |
|---|---|---|
| `Equals` / `NotEquals` | any | Value parsed against the property's type. |
| `GreaterThan` / `GreaterThanOrEqual` / `LessThan` / `LessThanOrEqual` | comparable types | |
| `Contains` / `NotContains` / `StartsWith` / `NotStartsWith` / `EndsWith` / `NotEndsWith` | `string` | Null-checked before the string method runs. |
| `Between` / `NotBetween` | comparable types | Value is `"min,max"`. |
| `IsNull` / `IsNotNull` | nullable types | |
| `IsEmpty` / `IsNotEmpty` | `string` | Treats `null` and `""` as empty. |
| `In` / `NotIn` | any | Value is a comma-separated list. |

Type conversion handles `Guid`, `DateTime`, `DateTimeOffset`, `DateOnly`, `TimeOnly`, enums, and anything with a `TypeConverter` that accepts strings.

## 🔗 Related packages

- [SH.Framework.Library.Cqrs](https://www.nuget.org/packages/SH.Framework.Library.Cqrs/) — the core CQRS abstractions (Request / Handler / Notification / Pipeline).
- [SH.Framework.Library.Cqrs.Implementation](https://www.nuget.org/packages/SH.Framework.Library.Cqrs.Implementation/) — base classes and the `Result<T>` pattern this lib builds on.

## 📦 Target frameworks

| TFM | EF Core |
|---|---|
| net8.0 | 8.0.x |
| net9.0 | 9.0.x |
| net10.0 | 10.0.x |

## 📄 License

MIT — see [LICENSE](./LICENSE).
