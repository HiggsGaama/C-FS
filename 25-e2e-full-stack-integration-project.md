<!-- 25-e2e-full-stack-integration-project.md -->
# 6.1 End-to-End Full-Stack Integration Architecture

## Learning Objectives
- Architect a multi-project full-stack monorepo or polyrepo pairing ASP.NET Core Web API and React 18.
- Establish robust Cross-Origin Resource Sharing (CORS) rules for local development and production domains.
- Bridge typed DTO contracts between C# backend and TypeScript client using type generation or contract synchronization.
- Configure development proxies in Vite to eliminate cross-origin cookies and CORS preflight latency.

---

## 1. Unified Integration Topology

```mermaid
graph TB
    subgraph ClientDev ["Vite Development Server (Port 5173)"]
        ReactApp["React TypeScript SPA"]
        ViteProxy["Vite Dev Server Proxy (/api, /hubs)"]
        ReactApp --> ViteProxy
    end

    subgraph ProductionGateway ["Production Ingress / Reverse Proxy (Port 443)"]
        Nginx["Nginx / Azure App Service / Cloudflare"]
        StaticAssets["Static Frontend (Vite Dist)"]
        Nginx -->|GET /* (HTML/JS/CSS)| StaticAssets
        Nginx -->|PROXY /api/* & /hubs/*| Kestrel
    end

    subgraph BackendAPI ["ASP.NET Core Web API (Port 5000 / 5001)"]
        Kestrel["Kestrel HTTP Server"]
        Controllers["Endpoints & Minimal APIs"]
        Hubs["SignalR Hubs (/hubs/orders)"]
        Kestrel --> Controllers
        Kestrel --> Hubs
    end

    ViteProxy -.->|HTTP & WS Proxy| Kestrel
```

---

## 2. Vite Development Proxy Configuration

Configuring a proxy in `vite.config.ts` allows the frontend code to send requests to relative paths (e.g. `/api/products` or `/hubs/telemetry`), avoiding browser CORS preflight requests during development.

```typescript
// vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  server: {
    port: 5173,
    strictPort: true,
    proxy: {
      // Proxy REST API endpoints to ASP.NET Core
      '/api': {
        target: 'http://localhost:5000',
        changeOrigin: true,
        secure: false,
      },
      // Proxy SignalR WebSockets
      '/hubs': {
        target: 'http://localhost:5000',
        changeOrigin: true,
        ws: true, // Enable WebSocket proxying
        secure: false,
      },
    },
  },
});
```

---

## 3. Production ASP.NET Core CORS Policy

In `Program.cs`, define an environment-aware CORS policy:

```csharp
// In ASP.NET Core Program.cs
var builder = WebApplication.CreateBuilder(args);

var allowedOrigins = builder.Configuration.GetSection("Cors:AllowedOrigins").Get<string[]>() 
    ?? new[] { "https://myapp.production.com", "http://localhost:5173" };

builder.Services.AddCors(options =>
{
    options.AddPolicy("ProductionCorsPolicy", policy =>
    {
        policy.WithOrigins(allowedOrigins)
              .AllowAnyHeader()
              .AllowAnyMethod()
              .AllowCredentials() // Mandatory for SignalR and authentication cookies
              .SetPreflightMaxAge(TimeSpan.FromHours(1)); // Cache preflight OPTIONS responses
    });
});

var app = builder.Build();

app.UseCors("ProductionCorsPolicy");
```

---

## 4. Shared Contract Typing

To prevent drift between backend C# records and frontend TypeScript interfaces, maintain synchronized definitions:

```csharp
// Backend: C# Record
namespace Shared.Contracts;

public sealed record CustomerSummaryDto(
    Guid Id, 
    string FullName, 
    string Email, 
    decimal TotalSpent, 
    bool IsActive);
```

```typescript
// Frontend: TypeScript Interface
export interface CustomerSummaryDto {
  id: string;
  fullName: string;
  email: string;
  totalSpent: number;
  isActive: boolean;
}
```

---

## 5. Key Takeaways
1. Use **Vite's built-in proxy** during development to keep request URLs relative and avoid CORS handshake overhead.
2. Production environments should terminate TLS at the reverse proxy (Nginx or cloud ingress) and forward requests to Kestrel over internal networking.
3. Keep C# DTOs and TypeScript models strictly aligned to avoid serialization and parsing errors.
