<!-- docs/module-06-e2e-testing-deployment/05-capstone-hands-on-solution.md -->
# 6.5 Module 6 Capstone Project: End-to-End Task Management System

## Capstone Scenario
You will build an end-to-end Task Management System. The solution features:
1. **ASP.NET Core Web API**: SQLite EF Core persistence, DTOs with FluentValidation, and a real-time SignalR Hub.
2. **React 18 + TypeScript Client**: Zustand state management, TanStack Query mutations, Tailwind CSS responsive layout, and SignalR live updates.
3. **Automated Tests**: xUnit backend unit tests and Vitest frontend component tests.

---

## 1. Project Directory Layout

```text
capstone-project/
├── backend/
│   ├── Program.cs
│   └── Backend.csproj
├── frontend/
│   ├── src/
│   │   ├── App.tsx
│   │   ├── main.tsx
│   │   └── index.css
│   ├── package.json
│   ├── vite.config.ts
│   └── index.html
└── tests/
    └── Backend.Tests/
        ├── TaskServiceTests.cs
        └── Backend.Tests.csproj
```

---

## 2. Complete Backend Implementation (`backend/Program.cs`)

Create the backend project:
```bash
dotnet new web -n Capstone.Backend
cd Capstone.Backend
dotnet add package Microsoft.EntityFrameworkCore.Sqlite
dotnet add package Microsoft.AspNetCore.SignalR
dotnet add package FluentValidation
dotnet add package FluentValidation.DependencyInjectionExtensions
```

Replace `Program.cs` with the complete implementation:

```csharp
using System.Text.Json;
using FluentValidation;
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.SignalR;
using Microsoft.EntityFrameworkCore;

var builder = WebApplication.CreateBuilder(args);

// 1. Service Registrations
builder.Services.AddDbContext<TaskDbContext>(options =>
    options.UseSqlite("Data Source=tasks.db"));

builder.Services.AddSignalR();
builder.Services.AddValidatorsFromAssemblyContaining<CreateTaskValidator>();

builder.Services.AddCors(options =>
{
    options.AddPolicy("CorsDev", policy =>
    {
        policy.WithOrigins("http://localhost:5173")
              .AllowAnyHeader()
              .AllowAnyMethod()
              .AllowCredentials();
    });
});

var app = builder.Build();

app.UseCors("CorsDev");

// 2. Database Initialization
using (var scope = app.Services.CreateScope())
{
    var db = scope.ServiceProvider.GetRequiredService<TaskDbContext>();
    db.Database.EnsureCreated();
}

// 3. API Endpoints
var taskGroup = app.MapGroup("/api/tasks");

// GET /api/tasks
taskGroup.MapGet("/", async (TaskDbContext db, CancellationToken ct) =>
{
    var tasks = await db.Tasks.AsNoTracking().OrderByDescending(t => t.CreatedAtUtc).ToListAsync(ct);
    var dtos = tasks.Select(t => new TaskItemDto(t.Id, t.Title, t.Description, t.IsCompleted, t.CreatedAtUtc));
    return Results.Ok(dtos);
});

// POST /api/tasks
taskGroup.MapPost("/", async (
    CreateTaskRequest request,
    IValidator<CreateTaskRequest> validator,
    TaskDbContext db,
    IHubContext<TaskNotificationHub, ITaskClient> hub,
    CancellationToken ct) =>
{
    var validationResult = await validator.ValidateAsync(request, ct);
    if (!validationResult.IsValid)
    {
        return Results.ValidationProblem(validationResult.ToDictionary());
    }

    var entity = new TaskEntity
    {
        Id = Guid.NewGuid(),
        Title = request.Title,
        Description = request.Description,
        IsCompleted = false,
        CreatedAtUtc = DateTime.UtcNow
    };

    db.Tasks.Add(entity);
    await db.SaveChangesAsync(ct);

    var dto = new TaskItemDto(entity.Id, entity.Title, entity.Description, entity.IsCompleted, entity.CreatedAtUtc);

    // Broadcast live event to all connected clients
    await hub.Clients.All.TaskCreated(dto);

    return Results.Created($"/api/tasks/{entity.Id}", dto);
});

// PATCH /api/tasks/{id:guid}/toggle
taskGroup.MapPatch("/{id:guid}/toggle", async (
    Guid id,
    TaskDbContext db,
    IHubContext<TaskNotificationHub, ITaskClient> hub,
    CancellationToken ct) =>
{
    var entity = await db.Tasks.FindAsync(new object[] { id }, ct);
    if (entity is null)
    {
        return Results.NotFound(new { message = $"Task with ID '{id}' was not found." });
    }

    entity.IsCompleted = !entity.IsCompleted;
    await db.SaveChangesAsync(ct);

    var dto = new TaskItemDto(entity.Id, entity.Title, entity.Description, entity.IsCompleted, entity.CreatedAtUtc);
    await hub.Clients.All.TaskStatusChanged(dto);

    return Results.Ok(dto);
});

// Map SignalR Hub
app.MapHub<TaskNotificationHub>("/hubs/tasks");

app.Run();

// ============================================================================
// DATA MODELS & SIGNALR CONTRACTS
// ============================================================================

public sealed class TaskEntity
{
    public Guid Id { get; set; }
    public string Title { get; set; } = string.Empty;
    public string Description { get; set; } = string.Empty;
    public bool IsCompleted { get; set; }
    public DateTime CreatedAtUtc { get; set; }
}

public sealed class TaskDbContext : DbContext
{
    public DbSet<TaskEntity> Tasks => Set<TaskEntity>();
    public TaskDbContext(DbContextOptions<TaskDbContext> options) : base(options) { }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);
        modelBuilder.Entity<TaskEntity>(entity =>
        {
            entity.HasKey(e => e.Id);
            entity.Property(e => e.Title).IsRequired().HasMaxLength(150);
            entity.Property(e => e.Description).HasMaxLength(500);
        });
    }
}

public sealed record CreateTaskRequest(string Title, string Description);
public sealed record TaskItemDto(Guid Id, string Title, string Description, bool IsCompleted, DateTime CreatedAtUtc);

public sealed class CreateTaskValidator : AbstractValidator<CreateTaskRequest>
{
    public CreateTaskValidator()
    {
        RuleFor(x => x.Title).NotEmpty().WithMessage("Title is required.").MaximumLength(150);
        RuleFor(x => x.Description).MaximumLength(500);
    }
}

public interface ITaskClient
{
    Task TaskCreated(TaskItemDto task);
    Task TaskStatusChanged(TaskItemDto task);
}

public sealed class TaskNotificationHub : Hub<ITaskClient> { }
```

---

## 3. Complete Frontend Implementation (`frontend/src/App.tsx`)

Initialize Vite React client:
```bash
npm create vite@latest frontend -- --template react-ts
cd frontend
npm install @microsoft/signalr @tanstack/react-query axios
npm install -D tailwindcss postcss autoprefixer
npx tailwindcss init -p
```

Replace `src/App.tsx`:

```tsx
import React, { useState, useEffect } from 'react';
import * as signalR from '@microsoft/signalr';
import axios from 'axios';
import {
  QueryClient,
  QueryClientProvider,
  useQuery,
  useMutation,
  useQueryClient,
} from '@tanstack/react-query';

// ---------------- 1. Interfaces ----------------

export interface TaskItem {
  id: string;
  title: string;
  description: string;
  isCompleted: boolean;
  createdAtUtc: string;
}

const API_BASE = 'http://localhost:5000/api/tasks';
const HUB_URL = 'http://localhost:5000/hubs/tasks';

// ---------------- 2. Main Dashboard Component ----------------

function TaskApp() {
  const queryClient = useQueryClient();
  const [title, setTitle] = useState('');
  const [description, setDescription] = useState('');
  const [hubConnected, setHubConnected] = useState(false);

  // Fetch tasks
  const { data: tasks = [], isLoading } = useQuery<TaskItem[]>({
    queryKey: ['tasks'],
    queryFn: async () => {
      const res = await axios.get<TaskItem[]>(API_BASE);
      return res.data;
    },
  });

  // SignalR WebSocket Subscription
  useEffect(() => {
    const connection = new signalR.HubConnectionBuilder()
      .withUrl(HUB_URL)
      .withAutomaticReconnect()
      .build();

    connection.on('TaskCreated', (newTask: TaskItem) => {
      queryClient.setQueryData<TaskItem[]>(['tasks'], (old = []) => [newTask, ...old]);
    });

    connection.on('TaskStatusChanged', (updatedTask: TaskItem) => {
      queryClient.setQueryData<TaskItem[]>(['tasks'], (old = []) =>
        old.map((t) => (t.id === updatedTask.id ? updatedTask : t))
      );
    });

    connection
      .start()
      .then(() => setHubConnected(true))
      .catch((err) => console.error('SignalR Connection Error:', err));

    return () => {
      connection.stop();
    };
  }, [queryClient]);

  // Create Task Mutation
  const createMutation = useMutation({
    mutationFn: async (payload: { title: string; description: string }) => {
      await axios.post(API_BASE, payload);
    },
    onSuccess: () => {
      setTitle('');
      setDescription('');
    },
  });

  // Toggle Task Mutation
  const toggleMutation = useMutation({
    mutationFn: async (id: string) => {
      await axios.patch(`${API_BASE}/${id}/toggle`);
    },
  });

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    if (!title.trim()) return;
    createMutation.mutate({ title, description });
  };

  return (
    <div className="min-h-screen bg-slate-100 p-6 sm:p-12 font-sans">
      <div className="max-w-2xl mx-auto space-y-6">
        {/* Header */}
        <div className="flex justify-between items-center bg-white p-6 rounded-xl shadow-xs border">
          <div>
            <h1 className="text-2xl font-bold text-slate-800">Team Task Manager</h1>
            <p className="text-xs text-slate-500 mt-1">Full-stack C# Web API + React + SignalR</p>
          </div>
          <div className="flex items-center gap-2">
            <span
              className={`w-3 h-3 rounded-full ${hubConnected ? 'bg-emerald-500' : 'bg-rose-500'} animate-pulse`}
            />
            <span className="text-xs font-semibold text-slate-600">
              {hubConnected ? 'Live' : 'Connecting'}
            </span>
          </div>
        </div>

        {/* Task Creation Form */}
        <form onSubmit={handleSubmit} className="bg-white p-6 rounded-xl shadow-xs border space-y-3">
          <input
            type="text"
            value={title}
            onChange={(e) => setTitle(e.target.value)}
            placeholder="What needs to be done?"
            className="w-full px-4 py-2 border rounded-lg text-sm focus:ring-2 focus:ring-indigo-500 focus:outline-none"
          />
          <input
            type="text"
            value={description}
            onChange={(e) => setDescription(e.target.value)}
            placeholder="Optional details or context..."
            className="w-full px-4 py-2 border rounded-lg text-sm focus:ring-2 focus:ring-indigo-500 focus:outline-none"
          />
          <div className="flex justify-end">
            <button
              type="submit"
              disabled={createMutation.isPending || !title.trim()}
              className="px-5 py-2 bg-indigo-600 hover:bg-indigo-700 disabled:opacity-50 text-white font-medium text-sm rounded-lg transition"
            >
              {createMutation.isPending ? 'Saving...' : 'Add Task'}
            </button>
          </div>
        </form>

        {/* Task List */}
        <div className="bg-white rounded-xl shadow-xs border divide-y overflow-hidden">
          {isLoading ? (
            <div className="p-8 text-center text-slate-400">Loading tasks...</div>
          ) : tasks.length === 0 ? (
            <div className="p-8 text-center text-slate-400">No tasks registered yet. Create one!</div>
          ) : (
            tasks.map((task) => (
              <div
                key={task.id}
                className="p-4 flex items-center justify-between hover:bg-slate-50 transition"
              >
                <div className="flex items-center gap-3">
                  <input
                    type="checkbox"
                    checked={task.isCompleted}
                    onChange={() => toggleMutation.mutate(task.id)}
                    className="w-5 h-5 rounded border-slate-300 text-indigo-600 focus:ring-indigo-500 cursor-pointer"
                  />
                  <div>
                    <p
                      className={`text-sm font-semibold ${
                        task.isCompleted ? 'line-through text-slate-400' : 'text-slate-800'
                      }`}
                    >
                      {task.title}
                    </p>
                    {task.description && (
                      <p className="text-xs text-slate-500 mt-0.5">{task.description}</p>
                    )}
                  </div>
                </div>
                <span className="text-[10px] text-slate-400 font-mono">
                  {new Date(task.createdAtUtc).toLocaleTimeString()}
                </span>
              </div>
            ))
          )}
        </div>
      </div>
    </div>
  );
}

const queryClient = new QueryClient();

export default function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <TaskApp />
    </QueryClientProvider>
  );
}
```

---

## 4. Backend Unit Tests (`tests/Backend.Tests/TaskServiceTests.cs`)

Create test project:
```bash
dotnet new xunit -n Capstone.Backend.Tests
cd Capstone.Backend.Tests
dotnet add package FluentAssertions
```

```csharp
namespace Capstone.Backend.Tests;

using Xunit;
using FluentAssertions;

public sealed class CreateTaskValidatorTests
{
    private readonly CreateTaskValidator _validator = new();

    [Fact]
    public void Validate_WithValidTitle_ShouldPassValidation()
    {
        // Arrange
        var request = new CreateTaskRequest("Finish Capstone Project", "Ensure CI/CD passes");

        // Act
        var result = _validator.Validate(request);

        // Assert
        result.IsValid.Should().BeTrue();
    }

    [Fact]
    public void Validate_WithEmptyTitle_ShouldFailValidation()
    {
        // Arrange
        var request = new CreateTaskRequest("", "Some description");

        // Act
        var result = _validator.Validate(request);

        // Assert
        result.IsValid.Should().BeFalse();
        result.Errors.Should().Contain(e => e.PropertyName == "Title");
    }
}
```

---

## 5. Verification & Running the Capstone

1. **Start the C# API**:
   ```bash
   cd backend
   dotnet run
   ```
   *Runs at `http://localhost:5000`*.
2. **Start the React Frontend**:
   ```bash
   cd frontend
   npm run dev
   ```
   *Runs at `http://localhost:5173`*.
3. Open two browser windows at `http://localhost:5173`:
   - Add a task in Window 1.
   - Observe the task immediately appear in Window 2 in real-time via SignalR!
   - Toggle the task checkbox in Window 2 and see it strike through synchronously in Window 1!
