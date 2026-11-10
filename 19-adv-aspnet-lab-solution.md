<!-- 19-adv-aspnet-lab-solution.md -->
# 4.5 Module 4 Hands-On Lab: Real-Time Order Stream with JWT Auth & SignalR

## Lab Scenario
You are developing an event-driven Order Management API. The API requires JWT authentication, allows clients to place orders, and immediately broadcasts real-time order lifecycle updates ("Placed" -> "Processing" -> "Shipped") using a SignalR Hub driven by an asynchronous Background Service.

---

## Lab Requirements
1. **Security & Tokens**:
   - Issue JWT Access Tokens for authenticated users.
   - Enforce authorization on the order creation endpoint.
2. **SignalR Hub**:
   - Strongly-typed `OrderHub` emitting `OnOrderCreated` and `OnOrderStatusChanged`.
3. **Background Worker**:
   - Background worker monitoring an in-memory order queue and stepping through status transitions with real-time push broadcasts.
4. **Execution**:
   - Fully functioning, self-contained `Program.cs` that can be run with `dotnet run`.

---

## Complete Solution Code

Create a new web project:
```bash
dotnet new web -n Module04.RealTimeOrders
cd Module04.RealTimeOrders

dotnet add package Microsoft.AspNetCore.Authentication.JwtBearer
```

Replace `Program.cs` with the complete implementation below:

```csharp
using System.Collections.Concurrent;
using System.IdentityModel.Tokens.Jwt;
using System.Security.Claims;
using System.Text;
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.SignalR;
using Microsoft.IdentityModel.Tokens;

// ============================================================================
// 1. SETUP BUILDER & CONFIGURATION
// ============================================================================

var builder = WebApplication.CreateBuilder(args);

const string JwtSecurityKey = "PRODUCTION_GRADE_SECRET_KEY_WITH_SUFFICIENT_LENGTH_123456789!";
const string JwtIssuer = "OrderService";
const string JwtAudience = "OrderServiceClients";

// Authentication & Authorization Setup
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidIssuer = JwtIssuer,
            ValidateAudience = true,
            ValidAudience = JwtAudience,
            ValidateLifetime = true,
            ClockSkew = TimeSpan.Zero,
            ValidateIssuerSigningKey = true,
            IssuerSigningKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(JwtSecurityKey))
        };

        // Allow SignalR to extract JWT token from query string during WebSocket handshake
        options.Events = new JwtBearerEvents
        {
            OnMessageReceived = context =>
            {
                var accessToken = context.Request.Query["access_token"];
                var path = context.HttpContext.Request.Path;
                if (!string.IsNullOrEmpty(accessToken) && path.StartsWithSegments("/hubs/orders"))
                {
                    context.Token = accessToken;
                }
                return Task.CompletedTask;
            }
        };
    });

builder.Services.AddAuthorization();
builder.Services.AddSignalR();
builder.Services.AddSingleton<IOrderQueue, InMemoryOrderQueue>();
builder.Services.AddHostedService<OrderProcessingWorker>();

// CORS for Frontend
builder.Services.AddCors(options =>
{
    options.AddPolicy("CorsPolicy", policy =>
    {
        policy.WithOrigins("http://localhost:5173", "http://localhost:3000")
              .AllowAnyHeader()
              .AllowAnyMethod()
              .AllowCredentials();
    });
});

var app = builder.Build();

app.UseCors("CorsPolicy");
app.UseAuthentication();
app.UseAuthorization();

// ============================================================================
// 2. ENDPOINTS
// ============================================================================

// POST /api/auth/token: Generates a JWT for testing
app.MapPost("/api/auth/token", (LoginRequest request) =>
{
    if (string.IsNullOrWhiteSpace(request.Username))
    {
        return Results.BadRequest(new { error = "Username is required." });
    }

    var claims = new[]
    {
        new Claim(ClaimTypes.NameIdentifier, Guid.NewGuid().ToString()),
        new Claim(ClaimTypes.Name, request.Username),
        new Claim(ClaimTypes.Role, "Customer")
    };

    var tokenDescriptor = new SecurityTokenDescriptor
    {
        Subject = new ClaimsIdentity(claims),
        Expires = DateTime.UtcNow.AddHours(2),
        Issuer = JwtIssuer,
        Audience = JwtAudience,
        SigningCredentials = new SigningCredentials(
            new SymmetricSecurityKey(Encoding.UTF8.GetBytes(JwtSecurityKey)),
            SecurityAlgorithms.HmacSha256Signature)
    };

    var handler = new JwtSecurityTokenHandler();
    var token = handler.WriteToken(handler.CreateToken(tokenDescriptor));

    return Results.Ok(new { AccessToken = token, ExpiresIn = 7200 });
});

// POST /api/orders: Protected endpoint for placing an order
app.MapPost("/api/orders", async (
    CreateOrderDto dto,
    ClaimsPrincipal user,
    IOrderQueue queue,
    IHubContext<OrderHub, IOrderClientContract> hubContext) =>
{
    var username = user.Identity?.Name ?? "Anonymous";

    var order = new OrderRecord(
        Id: Guid.NewGuid(),
        Customer: username,
        ItemDescription: dto.ItemDescription,
        Amount: dto.Amount,
        Status: "Placed",
        CreatedAtUtc: DateTime.UtcNow
    );

    // Broadcast new order placed to all connected clients
    await hubContext.Clients.All.OnOrderCreated(order);

    // Push into background worker queue for step-by-step processing
    queue.Enqueue(order);

    return Results.Accepted($"/api/orders/{order.Id}", order);
}).RequireAuthorization();

// Map SignalR Hub
app.MapHub<OrderHub>("/hubs/orders");

app.Run();

// ============================================================================
// 3. MODELS & CONTRACTS
// ============================================================================

public sealed record LoginRequest(string Username);
public sealed record CreateOrderDto(string ItemDescription, decimal Amount);

public sealed record OrderRecord(
    Guid Id,
    string Customer,
    string ItemDescription,
    decimal Amount,
    string Status,
    DateTime CreatedAtUtc);

public interface IOrderClientContract
{
    Task OnOrderCreated(OrderRecord order);
    Task OnOrderStatusChanged(Guid orderId, string newStatus, DateTime updatedAtUtc);
}

public sealed class OrderHub : Hub<IOrderClientContract>
{
    public override Task OnConnectedAsync()
    {
        Console.WriteLine($"[SignalR] Client connected: {Context.ConnectionId}");
        return base.OnConnectedAsync();
    }
}

// ============================================================================
// 4. QUEUE & BACKGROUND WORKER
// ============================================================================

public interface IOrderQueue
{
    void Enqueue(OrderRecord order);
    bool TryDequeue(out OrderRecord? order);
}

public sealed class InMemoryOrderQueue : IOrderQueue
{
    private readonly ConcurrentQueue<OrderRecord> _queue = new();

    public void Enqueue(OrderRecord order) => _queue.Enqueue(order);

    public bool TryDequeue(out OrderRecord? order) => _queue.TryDequeue(out order);
}

public sealed class OrderProcessingWorker : BackgroundService
{
    private readonly IOrderQueue _queue;
    private readonly IHubContext<OrderHub, IOrderClientContract> _hubContext;
    private readonly ILogger<OrderProcessingWorker> _logger;

    public OrderProcessingWorker(
        IOrderQueue queue,
        IHubContext<OrderHub, IOrderClientContract> hubContext,
        ILogger<OrderProcessingWorker> logger)
    {
        _queue = queue;
        _hubContext = hubContext;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _logger.LogInformation("OrderProcessingWorker started.");

        while (!stoppingToken.IsCancellationRequested)
        {
            if (_queue.TryDequeue(out var order) && order is not null)
            {
                _logger.LogInformation("Processing order {Id} for {Customer}...", order.Id, order.Customer);

                // Transition 1: Placed -> Processing (after 2 seconds)
                await Task.Delay(2000, stoppingToken);
                await _hubContext.Clients.All.OnOrderStatusChanged(order.Id, "Processing", DateTime.UtcNow);

                // Transition 2: Processing -> Shipped (after 3 seconds)
                await Task.Delay(3000, stoppingToken);
                await _hubContext.Clients.All.OnOrderStatusChanged(order.Id, "Shipped", DateTime.UtcNow);

                _logger.LogInformation("Order {Id} processed and shipped.", order.Id);
            }
            else
            {
                await Task.Delay(500, stoppingToken);
            }
        }
    }
}
```

---

## Verification & Execution Steps

1. Run the application:
   ```bash
   dotnet run
   ```
2. Generate an access token:
   ```bash
   curl -X POST http://localhost:5000/api/auth/token \
     -H "Content-Type: application/json" \
     -d '{"username": "DeveloperAlice"}'
   ```
3. Copy the returned `AccessToken` and place a new order:
   ```bash
   curl -X POST http://localhost:5000/api/orders \
     -H "Content-Type: application/json" \
     -H "Authorization: Bearer <PASTE_YOUR_ACCESS_TOKEN_HERE>" \
     -d '{"itemDescription": "Dual UltraWide Monitor Stand", "amount": 189.50}'
   ```
4. Observe the console output:
   - Order accepted (`202 Accepted`).
   - SignalR broadcasts `OnOrderCreated`.
   - Background worker dequeues the order, emits `OnOrderStatusChanged("Processing")` after 2s, and `OnOrderStatusChanged("Shipped")` after 3s!
