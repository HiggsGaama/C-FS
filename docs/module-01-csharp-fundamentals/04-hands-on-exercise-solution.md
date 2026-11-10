<!-- docs/module-01-csharp-fundamentals/04-hands-on-exercise-solution.md -->
# 1.4 Module 1 Hands-On Lab: Asynchronous In-Memory Order Processing Engine

## Lab Scenario
You are building an in-memory high-throughput order valuation and risk mitigation pipeline for an e-commerce platform. The pipeline must ingest raw incoming orders, validate them using pattern matching, calculate discounts based on customer tier, verify inventory asynchronously across warehouses, and emit an audit trail via async streams.

---

## Lab Requirements
1. **Model Entities**: Implement immutable records for `OrderItem`, `OrderRequest`, `OrderValidationResult`, and `ProcessedOrder`.
2. **Business Rules via Pattern Matching**:
   - VIP customers with orders over $500 get a 20% discount.
   - Wholesale orders (any item quantity > 50) get a 15% discount.
   - Orders with total amount <= 0 or empty item list fail validation.
3. **Asynchronous Verification**:
   - Simulate checking multi-warehouse availability asynchronously using `Task.WhenAll`.
   - Support `CancellationToken` cancellation.
4. **Execution Test**:
   - Provide a complete, executable `Program.cs` that can be run with `dotnet run`.

---

## Complete Solution Code

Create a new console application:
```bash
dotnet new console -n Module01.Lab
cd Module01.Lab
```

Replace `Program.cs` with the complete, production-ready implementation below:

```csharp
// Program.cs
using System.Collections.Concurrent;
using System.Runtime.CompilerServices;

namespace Module01.Lab;

public enum CustomerTier
{
    Standard,
    VIP,
    Wholesale
}

public sealed record OrderItem(string Sku, string ProductName, int Quantity, decimal UnitPrice)
{
    public decimal LineTotal => Quantity * UnitPrice;
}

public sealed record OrderRequest(
    Guid OrderId,
    string CustomerId,
    CustomerTier Tier,
    IReadOnlyList<OrderItem> Items,
    DateTime PlacedAtUtc);

public sealed record ProcessedOrder(
    Guid OrderId,
    string CustomerId,
    decimal OriginalTotal,
    decimal DiscountAmount,
    decimal FinalPayable,
    bool InventoryConfirmed,
    DateTime CompletedAtUtc);

public sealed class OrderProcessingException : Exception
{
    public Guid OrderId { get; }

    public OrderProcessingException(Guid orderId, string message) : base(message)
    {
        OrderId = orderId;
    }
}

public sealed class WarehouseInventoryService
{
    private static readonly ConcurrentDictionary<string, int> StockLevels = new()
    {
        ["SKU-LAPTOP-01"] = 15,
        ["SKU-MOUSE-02"] = 120,
        ["SKU-KEYBOARD-03"] = 45,
        ["SKU-MONITOR-04"] = 8
    };

    public async Task<bool> VerifyStockAsync(string sku, int requestedQty, CancellationToken ct = default)
    {
        // Simulate remote warehouse RPC call
        await Task.Delay(100, ct);

        if (StockLevels.TryGetValue(sku, out int currentStock))
        {
            return currentStock >= requestedQty;
        }

        return false;
    }
}

public sealed class OrderProcessor
{
    private readonly WarehouseInventoryService _inventoryService;

    public OrderProcessor(WarehouseInventoryService inventoryService)
    {
        _inventoryService = inventoryService;
    }

    public decimal CalculateDiscount(OrderRequest order)
    {
        decimal rawTotal = order.Items.Sum(i => i.LineTotal);
        bool hasBulkLineItem = order.Items.Any(i => i.Quantity >= 50);

        // Pattern matching switch expression
        return (order.Tier, rawTotal, hasBulkLineItem) switch
        {
            (CustomerTier.VIP, >= 500m, _) => rawTotal * 0.20m,
            (_, _, true) => rawTotal * 0.15m,
            (CustomerTier.VIP, _, _) => rawTotal * 0.10m,
            (CustomerTier.Standard, >= 200m, _) => rawTotal * 0.05m,
            _ => 0.00m
        };
    }

    public async Task<ProcessedOrder> ProcessOrderAsync(OrderRequest order, CancellationToken ct = default)
    {
        if (order.Items.Count == 0)
        {
            throw new OrderProcessingException(order.OrderId, "Cannot process order with no items.");
        }

        decimal originalTotal = order.Items.Sum(i => i.LineTotal);
        if (originalTotal <= 0)
        {
            throw new OrderProcessingException(order.OrderId, "Order value must be greater than zero.");
        }

        // Fan-out concurrent warehouse inventory verification
        var stockChecks = order.Items.Select(item => _inventoryService.VerifyStockAsync(item.Sku, item.Quantity, ct));
        bool[] stockResults = await Task.WhenAll(stockChecks);

        bool allInStock = stockResults.All(result => result);
        if (!allInStock)
        {
            throw new OrderProcessingException(order.OrderId, "One or more items in the order are out of stock.");
        }

        decimal discount = CalculateDiscount(order);
        decimal finalAmount = originalTotal - discount;

        return new ProcessedOrder(
            OrderId: order.OrderId,
            CustomerId: order.CustomerId,
            OriginalTotal: originalTotal,
            DiscountAmount: discount,
            FinalPayable: finalAmount,
            InventoryConfirmed: true,
            CompletedAtUtc: DateTime.UtcNow
        );
    }

    public async IAsyncEnumerable<ProcessedOrder> ProcessBatchStreamAsync(
        IEnumerable<OrderRequest> orders,
        [EnumeratorCancellation] CancellationToken ct = default)
    {
        foreach (var order in orders)
        {
            ct.ThrowIfCancellationRequested();

            ProcessedOrder processed;
            try
            {
                processed = await ProcessOrderAsync(order, ct);
            }
            catch (OrderProcessingException ex)
            {
                Console.ForegroundColor = ConsoleColor.Yellow;
                Console.WriteLine($"[SKIPPED] Order {ex.OrderId}: {ex.Message}");
                Console.ResetColor();
                continue;
            }

            yield return processed;
        }
    }
}

public static class Program
{
    public static async Task Main()
    {
        Console.WriteLine("=================================================");
        Console.WriteLine("  Module 1 Lab: Asynchronous Order Processor    ");
        Console.WriteLine("=================================================\n");

        var warehouse = new WarehouseInventoryService();
        var processor = new OrderProcessor(warehouse);

        var sampleOrders = new List<OrderRequest>
        {
            new(
                OrderId: Guid.NewGuid(),
                CustomerId: "CUST-001",
                Tier: CustomerTier.VIP,
                Items: new List<OrderItem>
                {
                    new("SKU-LAPTOP-01", "Developer Ultrabook", 1, 1200.00m),
                    new("SKU-MOUSE-02", "Ergonomic Wireless Mouse", 1, 65.00m)
                },
                PlacedAtUtc: DateTime.UtcNow
            ),
            new(
                OrderId: Guid.NewGuid(),
                CustomerId: "CUST-002",
                Tier: CustomerTier.Standard,
                Items: new List<OrderItem>
                {
                    new("SKU-KEYBOARD-03", "Mechanical RGB Keyboard", 2, 90.00m)
                },
                PlacedAtUtc: DateTime.UtcNow
            ),
            new(
                OrderId: Guid.NewGuid(),
                CustomerId: "CUST-003",
                Tier: CustomerTier.Wholesale,
                Items: new List<OrderItem>
                {
                    new("SKU-MOUSE-02", "Ergonomic Wireless Mouse", 60, 40.00m)
                },
                PlacedAtUtc: DateTime.UtcNow
            ),
            new(
                OrderId: Guid.NewGuid(),
                CustomerId: "CUST-004",
                Tier: CustomerTier.Standard,
                Items: new List<OrderItem>
                {
                    new("SKU-LAPTOP-01", "Developer Ultrabook", 50, 1200.00m) // Exceeds stock (only 15 available)
                },
                PlacedAtUtc: DateTime.UtcNow
            )
        };

        using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(10));

        Console.WriteLine("Streaming order execution results:\n");
        await foreach (var completed in processor.ProcessBatchStreamAsync(sampleOrders, cts.Token))
        {
            Console.ForegroundColor = ConsoleColor.Green;
            Console.WriteLine($"[CONFIRMED] Order ID : {completed.OrderId}");
            Console.ResetColor();
            Console.WriteLine($"   Customer ID    : {completed.CustomerId}");
            Console.WriteLine($"   Original Total : {completed.OriginalTotal:C}");
            Console.WriteLine($"   Discount       : -{completed.DiscountAmount:C}");
            Console.WriteLine($"   Final Payable  : {completed.FinalPayable:C}");
            Console.WriteLine($"   Timestamp      : {completed.CompletedAtUtc:O}\n");
        }

        Console.WriteLine("All orders successfully evaluated.");
    }
}
```

---

## Verification & Execution Steps

1. Verify and run the program:
   ```bash
   dotnet run
   ```
2. **Expected Output**:
   - Order `CUST-001` (VIP, > $500) applies 20% discount on $1,265.00 ($253.00 discount).
   - Order `CUST-002` (Standard, < $200) applies no discount on $180.00.
   - Order `CUST-003` (Wholesale, 60 items) applies 15% discount on $2,400.00.
   - Order `CUST-004` (Laptop 50 items) is caught by `OrderProcessingException` due to stock exhaustion and gracefully skipped without terminating the stream.
