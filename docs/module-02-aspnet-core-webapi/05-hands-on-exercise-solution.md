<!-- docs/module-02-aspnet-core-webapi/05-hands-on-exercise-solution.md -->
# 2.5 Module 2 Hands-On Lab: Production Inventory REST API with EF Core & Serilog

## Lab Scenario
You will build a production-grade Inventory and Order Management Web API using ASP.NET Core Minimal APIs, EF Core with an SQLite database, FluentValidation for incoming payloads, custom correlation middleware, and RFC 7807 global exception handling.

---

## Lab Requirements
1. **Database Entity & DbContext**:
   - `InventoryItem`: `Id (Guid)`, `Sku (string)`, `Name (string)`, `Quantity (int)`, `UnitPrice (decimal)`, `LastUpdatedUtc (DateTime)`.
   - SQLite Code-First configuration with Fluent API.
2. **DTO & FluentValidation**:
   - `CreateInventoryItemRequest`: Sku (alphanumeric pattern), Name (non-empty), Quantity (>= 0), UnitPrice (> 0).
   - `InventoryItemResponse`: Output DTO.
3. **Endpoints**:
   - `GET /api/inventory`: List all items with pagination (`page`, `pageSize`).
   - `GET /api/inventory/{id:guid}`: Retrieve item by ID.
   - `POST /api/inventory`: Create item (validated with FluentValidation).
   - `POST /api/inventory/{id:guid}/adjust-stock`: Adjust item stock level safely with concurrency handling.
4. **Middleware**:
   - Global RFC 7807 Exception Handler.
   - Correlation ID injector.

---

## Complete Solution Code

### 1. Project Creation & Package Setup
```bash
dotnet new web -n Module02.InventoryApi
cd Module02.InventoryApi

dotnet add package Microsoft.EntityFrameworkCore.Sqlite
dotnet add package Microsoft.EntityFrameworkCore.Design
dotnet add package FluentValidation
dotnet add package FluentValidation.DependencyInjectionExtensions
dotnet add package Serilog.AspNetCore
dotnet add package Serilog.Sinks.Console
```

### 2. Complete Program.cs File

Replace the contents of `Program.cs` with the complete, fully functioning implementation below:

```csharp
using System.Diagnostics;
using System.Net;
using System.Text.Json;
using FluentValidation;
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;
using Serilog;
using Serilog.Context;

// 1. Serilog Setup
Log.Logger = new LoggerConfiguration()
    .WriteTo.Console(outputTemplate: "[{Timestamp:HH:mm:ss} {Level:u3}] [{CorrelationId}] {Message:lj}{NewLine}{Exception}")
    .CreateBootstrapLogger();

var builder = WebApplication.CreateBuilder(args);

builder.Host.UseSerilog((ctx, lc) => lc
    .WriteTo.Console()
    .Enrich.FromLogContext());

// 2. EF Core SQLite Configuration
builder.Services.AddDbContext<InventoryDbContext>(options =>
    options.UseSqlite("Data Source=inventory.db"));

// 3. Register Validators
builder.Services.AddValidatorsFromAssemblyContaining<CreateInventoryItemValidator>();

var app = builder.Build();

// 4. Ensure SQLite database is created automatically
using (var scope = app.Services.CreateScope())
{
    var db = scope.ServiceProvider.GetRequiredService<InventoryDbContext>();
    db.Database.EnsureCreated();
}

// 5. Middleware Pipeline
app.UseMiddleware<GlobalExceptionMiddleware>();
app.UseMiddleware<CorrelationMiddleware>();

// 6. Minimal API Endpoints
var inventoryGroup = app.MapGroup("/api/inventory");

// GET /api/inventory (Paginated)
inventoryGroup.MapGet("/", async (
    [FromQuery] int page, 
    [FromQuery] int pageSize, 
    InventoryDbContext db, 
    CancellationToken ct) =>
{
    int safePage = page <= 0 ? 1 : page;
    int safeSize = pageSize <= 0 ? 10 : Math.Min(pageSize, 50);

    var query = db.InventoryItems.AsNoTracking().OrderBy(i => i.Sku);
    int totalCount = await query.CountAsync(ct);

    var items = await query
        .Skip((safePage - 1) * safeSize)
        .Take(safeSize)
        .Select(i => new InventoryItemResponse(i.Id, i.Sku, i.Name, i.Quantity, i.UnitPrice, i.LastUpdatedUtc))
        .ToListAsync(ct);

    return Results.Ok(new
    {
        Page = safePage,
        PageSize = safeSize,
        TotalCount = totalCount,
        TotalPages = (int)Math.Ceiling(totalCount / (double)safeSize),
        Data = items
    });
});

// GET /api/inventory/{id:guid}
inventoryGroup.MapGet("/{id:guid}", async (Guid id, InventoryDbContext db, CancellationToken ct) =>
{
    var item = await db.InventoryItems.AsNoTracking().FirstOrDefaultAsync(i => i.Id == id, ct);
    if (item is null)
    {
        throw new KeyNotFoundException($"Inventory item with ID '{id}' was not found.");
    }

    return Results.Ok(new InventoryItemResponse(item.Id, item.Sku, item.Name, item.Quantity, item.UnitPrice, item.LastUpdatedUtc));
});

// POST /api/inventory
inventoryGroup.MapPost("/", async (
    CreateInventoryItemRequest request,
    IValidator<CreateInventoryItemRequest> validator,
    InventoryDbContext db,
    CancellationToken ct) =>
{
    var valResult = await validator.ValidateAsync(request, ct);
    if (!valResult.IsValid)
    {
        return Results.ValidationProblem(valResult.ToDictionary());
    }

    bool skuExists = await db.InventoryItems.AnyAsync(i => i.Sku == request.Sku.ToUpperInvariant(), ct);
    if (skuExists)
    {
        throw new InvalidOperationException($"SKU '{request.Sku}' is already registered.");
    }

    var entity = new InventoryItem
    {
        Id = Guid.NewGuid(),
        Sku = request.Sku.ToUpperInvariant(),
        Name = request.Name,
        Quantity = request.Quantity,
        UnitPrice = request.UnitPrice,
        LastUpdatedUtc = DateTime.UtcNow
    };

    db.InventoryItems.Add(entity);
    await db.SaveChangesAsync(ct);

    var response = new InventoryItemResponse(entity.Id, entity.Sku, entity.Name, entity.Quantity, entity.UnitPrice, entity.LastUpdatedUtc);
    return Results.Created($"/api/inventory/{entity.Id}", response);
});

// POST /api/inventory/{id:guid}/adjust-stock
inventoryGroup.MapPost("/{id:guid}/adjust-stock", async (
    Guid id, 
    AdjustStockRequest request, 
    InventoryDbContext db, 
    CancellationToken ct) =>
{
    var item = await db.InventoryItems.FirstOrDefaultAsync(i => i.Id == id, ct);
    if (item is null)
    {
        throw new KeyNotFoundException($"Inventory item with ID '{id}' was not found.");
    }

    int newQuantity = item.Quantity + request.QuantityDelta;
    if (newQuantity < 0)
    {
        throw new InvalidOperationException($"Stock cannot be reduced below zero. Current: {item.Quantity}, Adjustment: {request.QuantityDelta}");
    }

    item.Quantity = newQuantity;
    item.LastUpdatedUtc = DateTime.UtcNow;

    await db.SaveChangesAsync(ct);

    return Results.Ok(new InventoryItemResponse(item.Id, item.Sku, item.Name, item.Quantity, item.UnitPrice, item.LastUpdatedUtc));
});

app.Run();

// ============================================================================
// DOMAIN ENTITIES & EF CORE DB CONTEXT
// ============================================================================

public sealed class InventoryItem
{
    public Guid Id { get; set; }
    public string Sku { get; set; } = string.Empty;
    public string Name { get; set; } = string.Empty;
    public int Quantity { get; set; }
    public decimal UnitPrice { get; set; }
    public DateTime LastUpdatedUtc { get; set; }
}

public sealed class InventoryDbContext : DbContext
{
    public DbSet<InventoryItem> InventoryItems => Set<InventoryItem>();

    public InventoryDbContext(DbContextOptions<InventoryDbContext> options) : base(options) { }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);

        modelBuilder.Entity<InventoryItem>(entity =>
        {
            entity.HasKey(e => e.Id);
            entity.HasIndex(e => e.Sku).IsUnique();
            entity.Property(e => e.Sku).IsRequired().HasMaxLength(32);
            entity.Property(e => e.Name).IsRequired().HasMaxLength(150);
            entity.Property(e => e.UnitPrice).HasPrecision(18, 2);
        });
    }
}

// ============================================================================
// CONTRACTS & FLUENT VALIDATION
// ============================================================================

public sealed record CreateInventoryItemRequest(string Sku, string Name, int Quantity, decimal UnitPrice);
public sealed record AdjustStockRequest(int QuantityDelta);
public sealed record InventoryItemResponse(Guid Id, string Sku, string Name, int Quantity, decimal UnitPrice, DateTime LastUpdatedUtc);

public sealed class CreateInventoryItemValidator : AbstractValidator<CreateInventoryItemRequest>
{
    public CreateInventoryItemValidator()
    {
        RuleFor(x => x.Sku)
            .NotEmpty().WithMessage("SKU is required.")
            .Matches(@"^[A-Z0-9\-]{4,16}$").WithMessage("SKU must be 4-16 uppercase alphanumeric characters or dashes.");

        RuleFor(x => x.Name)
            .NotEmpty().WithMessage("Name is required.")
            .MaximumLength(150).WithMessage("Name cannot exceed 150 characters.");

        RuleFor(x => x.Quantity)
            .GreaterThanOrEqualTo(0).WithMessage("Initial quantity cannot be negative.");

        RuleFor(x => x.UnitPrice)
            .GreaterThan(0m).WithMessage("Unit price must be greater than zero.");
    }
}

// ============================================================================
// CUSTOM MIDDLEWARE
// ============================================================================

public sealed class CorrelationMiddleware
{
    private readonly RequestDelegate _next;
    public CorrelationMiddleware(RequestDelegate next) => _next = next;

    public async Task InvokeAsync(HttpContext context)
    {
        string correlationId = context.Request.Headers.TryGetValue("X-Correlation-ID", out var cid) && !string.IsNullOrWhiteSpace(cid)
            ? cid.ToString()
            : Guid.NewGuid().ToString("N");

        context.Items["X-Correlation-ID"] = correlationId;
        context.Response.Headers["X-Correlation-ID"] = correlationId;

        using (LogContext.PushProperty("CorrelationId", correlationId))
        {
            await _next(context);
        }
    }
}

public sealed class GlobalExceptionMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<GlobalExceptionMiddleware> _logger;

    public GlobalExceptionMiddleware(RequestDelegate next, ILogger<GlobalExceptionMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await _next(context);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "HTTP failure on {Path}", context.Request.Path);

            var (status, title) = ex switch
            {
                KeyNotFoundException => (HttpStatusCode.NotFound, "Resource Not Found"),
                InvalidOperationException => (HttpStatusCode.Conflict, "Business Conflict"),
                ArgumentException => (HttpStatusCode.BadRequest, "Bad Request"),
                _ => (HttpStatusCode.InternalServerError, "Internal Server Error")
            };

            context.Response.ContentType = "application/problem+json";
            context.Response.StatusCode = (int)status;

            var problem = new ProblemDetails
            {
                Status = (int)status,
                Title = title,
                Detail = ex.Message,
                Instance = context.Request.Path
            };

            if (context.Items.TryGetValue("X-Correlation-ID", out var correlationId))
            {
                problem.Extensions["correlationId"] = correlationId;
            }

            var json = JsonSerializer.Serialize(problem, new JsonSerializerOptions { PropertyNamingPolicy = JsonNamingPolicy.CamelCase });
            await context.Response.WriteAsync(json);
        }
    }
}
```

---

## Verification & Execution Steps

1. Start the API:
   ```bash
   dotnet run
   ```
2. In a separate terminal or HTTP client (cURL / Postman), create a new inventory item:
   ```bash
   curl -X POST http://localhost:5000/api/inventory \
     -H "Content-Type: application/json" \
     -d '{"sku": "MON-4K-001", "name": "32-inch 4K Gaming Monitor", "quantity": 10, "unitPrice": 499.99}'
   ```
3. Test validation rejection:
   ```bash
   curl -X POST http://localhost:5000/api/inventory \
     -H "Content-Type: application/json" \
     -d '{"sku": "bad", "name": "", "quantity": -5, "unitPrice": 0}'
   ```
   **Expected**: HTTP 400 ValidationProblem with detailed validation errors.
4. Test Stock Adjustment and conflict prevention (adjusting below zero):
   ```bash
   curl -X POST http://localhost:5000/api/inventory/<ITEM_GUID>/adjust-stock \
     -H "Content-Type: application/json" \
     -d '{"quantityDelta": -50}'
   ```
   **Expected**: HTTP 409 Conflict with RFC 7807 ProblemDetails payload.
