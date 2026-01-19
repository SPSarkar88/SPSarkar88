# Building a Complete CRUD Todo Application with Angular and .NET Core

## Introduction

In this comprehensive tutorial, we'll build a full-stack Todo application using Angular for the frontend and .NET Core Web API for the backend. This application demonstrates Create, Read, Update, and Delete (CRUD) operations with a modern, professional architecture.

**What You'll Learn:**
- Creating a RESTful API with .NET Core
- Building a responsive UI with Angular
- Implementing CRUD operations
- Connecting Angular to .NET API
- Error handling and validation
- Best practices for full-stack development

## Prerequisites

Before starting, ensure you have:
- **.NET SDK 8.0** or later ([Download](https://dotnet.microsoft.com/download))
- **Node.js 18+** and npm ([Download](https://nodejs.org/))
- **Angular CLI**: Install with `npm install -g @angular/cli`
- **Visual Studio Code** or any preferred IDE
- **SQL Server** (LocalDB or Express)
- Basic knowledge of C# and TypeScript

## Project Architecture

```
TodoApp/
├── Backend (TodoAPI)/
│   ├── Controllers/
│   ├── Models/
│   ├── Data/
│   └── Program.cs
└── Frontend (TodoUI)/
    ├── src/
    │   ├── app/
    │   │   ├── models/
    │   │   ├── services/
    │   │   └── components/
    │   └── environments/
```

---

## Part 1: Building the .NET Core Web API Backend

### Step 1: Create the .NET Web API Project

Open your terminal and run:

```bash
# Create solution folder
mkdir TodoApp
cd TodoApp

# Create Web API project
dotnet new webapi -n TodoAPI
cd TodoAPI

# Install required packages
dotnet add package Microsoft.EntityFrameworkCore.SqlServer
dotnet add package Microsoft.EntityFrameworkCore.Tools
dotnet add package Microsoft.EntityFrameworkCore.Design
```

### Step 2: Create the Todo Model

Create `Models/TodoItem.cs`:

```csharp
namespace TodoAPI.Models
{
    public class TodoItem
    {
        public int Id { get; set; }
        
        public string Title { get; set; } = string.Empty;
        
        public string Description { get; set; } = string.Empty;
        
        public bool IsCompleted { get; set; }
        
        public DateTime CreatedDate { get; set; } = DateTime.UtcNow;
        
        public DateTime? DueDate { get; set; }
        
        public string Priority { get; set; } = "Medium"; // Low, Medium, High
    }
}
```

### Step 3: Create the Database Context

Create `Data/TodoContext.cs`:

```csharp
using Microsoft.EntityFrameworkCore;
using TodoAPI.Models;

namespace TodoAPI.Data
{
    public class TodoContext : DbContext
    {
        public TodoContext(DbContextOptions<TodoContext> options) : base(options)
        {
        }

        public DbSet<TodoItem> TodoItems { get; set; }

        protected override void OnModelCreating(ModelBuilder modelBuilder)
        {
            base.OnModelCreating(modelBuilder);

            // Seed data
            modelBuilder.Entity<TodoItem>().HasData(
                new TodoItem
                {
                    Id = 1,
                    Title = "Complete project documentation",
                    Description = "Write comprehensive documentation for the project",
                    IsCompleted = false,
                    CreatedDate = DateTime.UtcNow,
                    Priority = "High"
                },
                new TodoItem
                {
                    Id = 2,
                    Title = "Review pull requests",
                    Description = "Review pending pull requests on GitHub",
                    IsCompleted = true,
                    CreatedDate = DateTime.UtcNow.AddDays(-1),
                    Priority = "Medium"
                }
            );
        }
    }
}
```

### Step 4: Create the Todo Controller

Create `Controllers/TodoController.cs`:

```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;
using TodoAPI.Data;
using TodoAPI.Models;

namespace TodoAPI.Controllers
{
    [Route("api/[controller]")]
    [ApiController]
    public class TodoController : ControllerBase
    {
        private readonly TodoContext _context;
        private readonly ILogger<TodoController> _logger;

        public TodoController(TodoContext context, ILogger<TodoController> logger)
        {
            _context = context;
            _logger = logger;
        }

        // GET: api/Todo
        [HttpGet]
        public async Task<ActionResult<IEnumerable<TodoItem>>> GetTodoItems()
        {
            try
            {
                var items = await _context.TodoItems
                    .OrderByDescending(t => t.CreatedDate)
                    .ToListAsync();
                return Ok(items);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error retrieving todo items");
                return StatusCode(500, "Internal server error");
            }
        }

        // GET: api/Todo/5
        [HttpGet("{id}")]
        public async Task<ActionResult<TodoItem>> GetTodoItem(int id)
        {
            try
            {
                var todoItem = await _context.TodoItems.FindAsync(id);

                if (todoItem == null)
                {
                    return NotFound($"Todo item with ID {id} not found");
                }

                return Ok(todoItem);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error retrieving todo item {Id}", id);
                return StatusCode(500, "Internal server error");
            }
        }

        // POST: api/Todo
        [HttpPost]
        public async Task<ActionResult<TodoItem>> CreateTodoItem(TodoItem todoItem)
        {
            try
            {
                if (string.IsNullOrWhiteSpace(todoItem.Title))
                {
                    return BadRequest("Title is required");
                }

                todoItem.CreatedDate = DateTime.UtcNow;
                _context.TodoItems.Add(todoItem);
                await _context.SaveChangesAsync();

                return CreatedAtAction(
                    nameof(GetTodoItem),
                    new { id = todoItem.Id },
                    todoItem
                );
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error creating todo item");
                return StatusCode(500, "Internal server error");
            }
        }

        // PUT: api/Todo/5
        [HttpPut("{id}")]
        public async Task<IActionResult> UpdateTodoItem(int id, TodoItem todoItem)
        {
            if (id != todoItem.Id)
            {
                return BadRequest("ID mismatch");
            }

            try
            {
                var existingItem = await _context.TodoItems.FindAsync(id);
                if (existingItem == null)
                {
                    return NotFound($"Todo item with ID {id} not found");
                }

                existingItem.Title = todoItem.Title;
                existingItem.Description = todoItem.Description;
                existingItem.IsCompleted = todoItem.IsCompleted;
                existingItem.DueDate = todoItem.DueDate;
                existingItem.Priority = todoItem.Priority;

                await _context.SaveChangesAsync();
                return NoContent();
            }
            catch (DbUpdateConcurrencyException ex)
            {
                _logger.LogError(ex, "Concurrency error updating todo item {Id}", id);
                return StatusCode(409, "The item was modified by another user");
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error updating todo item {Id}", id);
                return StatusCode(500, "Internal server error");
            }
        }

        // DELETE: api/Todo/5
        [HttpDelete("{id}")]
        public async Task<IActionResult> DeleteTodoItem(int id)
        {
            try
            {
                var todoItem = await _context.TodoItems.FindAsync(id);
                if (todoItem == null)
                {
                    return NotFound($"Todo item with ID {id} not found");
                }

                _context.TodoItems.Remove(todoItem);
                await _context.SaveChangesAsync();

                return NoContent();
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error deleting todo item {Id}", id);
                return StatusCode(500, "Internal server error");
            }
        }

        // PATCH: api/Todo/5/toggle
        [HttpPatch("{id}/toggle")]
        public async Task<IActionResult> ToggleTodoItem(int id)
        {
            try
            {
                var todoItem = await _context.TodoItems.FindAsync(id);
                if (todoItem == null)
                {
                    return NotFound($"Todo item with ID {id} not found");
                }

                todoItem.IsCompleted = !todoItem.IsCompleted;
                await _context.SaveChangesAsync();

                return Ok(todoItem);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error toggling todo item {Id}", id);
                return StatusCode(500, "Internal server error");
            }
        }
    }
}
```

### Step 5: Configure Program.cs

Replace the content of `Program.cs`:

```csharp
using Microsoft.EntityFrameworkCore;
using TodoAPI.Data;

var builder = WebApplication.CreateBuilder(args);

// Add services to the container
builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

// Configure Database
builder.Services.AddDbContext<TodoContext>(options =>
    options.UseSqlServer(
        builder.Configuration.GetConnectionString("DefaultConnection")
    )
);

// Configure CORS
builder.Services.AddCors(options =>
{
    options.AddPolicy("AllowAngularApp",
        policy =>
        {
            policy.WithOrigins("http://localhost:4200")
                  .AllowAnyHeader()
                  .AllowAnyMethod();
        });
});

var app = builder.Build();

// Configure the HTTP request pipeline
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();

app.UseCors("AllowAngularApp");

app.UseAuthorization();

app.MapControllers();

app.Run();
```

### Step 6: Update appsettings.json

Add the connection string to `appsettings.json`:

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=(localdb)\\mssqllocaldb;Database=TodoDb;Trusted_Connection=true;TrustServerCertificate=true"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "AllowedHosts": "*"
}
```

### Step 7: Create and Apply Database Migration

```bash
# Create initial migration
dotnet ef migrations add InitialCreate

# Apply migration to database
dotnet ef database update
```

### Step 8: Run the API

```bash
dotnet run
```

The API will be available at `https://localhost:7XXX` (check console output). Swagger UI will be at `https://localhost:7XXX/swagger`.

---

## Part 2: Building the Angular Frontend

### Step 1: Create Angular Project

```bash
# Navigate to TodoApp folder (parent directory)
cd ..

# Create Angular project
ng new TodoUI --routing --style=css
cd TodoUI
```

Answer the prompts:
- Would you like to add Angular routing? **Yes**
- Which stylesheet format would you like to use? **CSS**

### Step 2: Install Bootstrap

```bash
npm install bootstrap
```

Add Bootstrap to `angular.json`:

```json
"styles": [
  "node_modules/bootstrap/dist/css/bootstrap.min.css",
  "src/styles.css"
]
```

### Step 3: Create Todo Model

Create `src/app/models/todo.model.ts`:

```typescript
export interface Todo {
  id: number;
  title: string;
  description: string;
  isCompleted: boolean;
  createdDate: Date;
  dueDate?: Date;
  priority: 'Low' | 'Medium' | 'High';
}

export interface CreateTodoDto {
  title: string;
  description: string;
  priority: 'Low' | 'Medium' | 'High';
  dueDate?: Date;
}

export interface UpdateTodoDto {
  id: number;
  title: string;
  description: string;
  isCompleted: boolean;
  priority: 'Low' | 'Medium' | 'High';
  dueDate?: Date;
}
```

### Step 4: Create Todo Service

Create `src/app/services/todo.service.ts`:

```typescript
import { Injectable } from '@angular/core';
import { HttpClient, HttpErrorResponse } from '@angular/common/http';
import { Observable, throwError } from 'rxjs';
import { catchError, tap } from 'rxjs/operators';
import { Todo, CreateTodoDto, UpdateTodoDto } from '../models/todo.model';

@Injectable({
  providedIn: 'root'
})
export class TodoService {
  private apiUrl = 'https://localhost:7XXX/api/todo'; // Update with your API port

  constructor(private http: HttpClient) { }

  // Get all todos
  getAllTodos(): Observable<Todo[]> {
    return this.http.get<Todo[]>(this.apiUrl).pipe(
      tap(data => console.log('Fetched todos:', data)),
      catchError(this.handleError)
    );
  }

  // Get single todo
  getTodoById(id: number): Observable<Todo> {
    return this.http.get<Todo>(`${this.apiUrl}/${id}`).pipe(
      catchError(this.handleError)
    );
  }

  // Create new todo
  createTodo(todo: CreateTodoDto): Observable<Todo> {
    return this.http.post<Todo>(this.apiUrl, todo).pipe(
      tap(data => console.log('Created todo:', data)),
      catchError(this.handleError)
    );
  }

  // Update todo
  updateTodo(id: number, todo: UpdateTodoDto): Observable<void> {
    return this.http.put<void>(`${this.apiUrl}/${id}`, todo).pipe(
      tap(() => console.log('Updated todo:', id)),
      catchError(this.handleError)
    );
  }

  // Delete todo
  deleteTodo(id: number): Observable<void> {
    return this.http.delete<void>(`${this.apiUrl}/${id}`).pipe(
      tap(() => console.log('Deleted todo:', id)),
      catchError(this.handleError)
    );
  }

  // Toggle todo completion
  toggleTodo(id: number): Observable<Todo> {
    return this.http.patch<Todo>(`${this.apiUrl}/${id}/toggle`, {}).pipe(
      tap(data => console.log('Toggled todo:', data)),
      catchError(this.handleError)
    );
  }

  // Error handling
  private handleError(error: HttpErrorResponse) {
    let errorMessage = 'An unknown error occurred';
    
    if (error.error instanceof ErrorEvent) {
      // Client-side error
      errorMessage = `Error: ${error.error.message}`;
    } else {
      // Server-side error
      errorMessage = `Error Code: ${error.status}\nMessage: ${error.message}`;
    }
    
    console.error(errorMessage);
    return throwError(() => new Error(errorMessage));
  }
}
```

### Step 5: Create Todo List Component

Generate the component:

```bash
ng generate component components/todo-list
```

Update `src/app/components/todo-list/todo-list.component.ts`:

```typescript
import { Component, OnInit } from '@angular/core';
import { TodoService } from '../../services/todo.service';
import { Todo, CreateTodoDto } from '../../models/todo.model';

@Component({
  selector: 'app-todo-list',
  templateUrl: './todo-list.component.html',
  styleUrls: ['./todo-list.component.css']
})
export class TodoListComponent implements OnInit {
  todos: Todo[] = [];
  filteredTodos: Todo[] = [];
  editingTodo: Todo | null = null;
  
  // New todo form
  newTodo: CreateTodoDto = {
    title: '',
    description: '',
    priority: 'Medium'
  };

  // Filter
  filterStatus: 'all' | 'active' | 'completed' = 'all';

  constructor(private todoService: TodoService) { }

  ngOnInit(): void {
    this.loadTodos();
  }

  loadTodos(): void {
    this.todoService.getAllTodos().subscribe({
      next: (data) => {
        this.todos = data;
        this.applyFilter();
      },
      error: (error) => {
        console.error('Error loading todos:', error);
        alert('Failed to load todos. Please try again.');
      }
    });
  }

  applyFilter(): void {
    switch (this.filterStatus) {
      case 'active':
        this.filteredTodos = this.todos.filter(t => !t.isCompleted);
        break;
      case 'completed':
        this.filteredTodos = this.todos.filter(t => t.isCompleted);
        break;
      default:
        this.filteredTodos = [...this.todos];
    }
  }

  setFilter(status: 'all' | 'active' | 'completed'): void {
    this.filterStatus = status;
    this.applyFilter();
  }

  createTodo(): void {
    if (!this.newTodo.title.trim()) {
      alert('Please enter a title');
      return;
    }

    this.todoService.createTodo(this.newTodo).subscribe({
      next: () => {
        this.loadTodos();
        this.resetForm();
      },
      error: (error) => {
        console.error('Error creating todo:', error);
        alert('Failed to create todo. Please try again.');
      }
    });
  }

  editTodo(todo: Todo): void {
    this.editingTodo = { ...todo };
  }

  cancelEdit(): void {
    this.editingTodo = null;
  }

  saveTodo(): void {
    if (!this.editingTodo) return;

    this.todoService.updateTodo(this.editingTodo.id, this.editingTodo).subscribe({
      next: () => {
        this.loadTodos();
        this.editingTodo = null;
      },
      error: (error) => {
        console.error('Error updating todo:', error);
        alert('Failed to update todo. Please try again.');
      }
    });
  }

  toggleTodo(id: number): void {
    this.todoService.toggleTodo(id).subscribe({
      next: () => {
        this.loadTodos();
      },
      error: (error) => {
        console.error('Error toggling todo:', error);
        alert('Failed to toggle todo. Please try again.');
      }
    });
  }

  deleteTodo(id: number): void {
    if (!confirm('Are you sure you want to delete this todo?')) {
      return;
    }

    this.todoService.deleteTodo(id).subscribe({
      next: () => {
        this.loadTodos();
      },
      error: (error) => {
        console.error('Error deleting todo:', error);
        alert('Failed to delete todo. Please try again.');
      }
    });
  }

  resetForm(): void {
    this.newTodo = {
      title: '',
      description: '',
      priority: 'Medium'
    };
  }

  getPriorityClass(priority: string): string {
    switch (priority) {
      case 'High': return 'badge bg-danger';
      case 'Medium': return 'badge bg-warning';
      case 'Low': return 'badge bg-info';
      default: return 'badge bg-secondary';
    }
  }
}
```

Update `src/app/components/todo-list/todo-list.component.html`:

```html
<div class="container mt-5">
  <div class="row">
    <div class="col-md-12">
      <h1 class="text-center mb-4">
        <i class="bi bi-check2-square"></i> Todo Application
      </h1>

      <!-- Create New Todo Card -->
      <div class="card mb-4">
        <div class="card-header bg-primary text-white">
          <h5 class="mb-0">Create New Todo</h5>
        </div>
        <div class="card-body">
          <form (ngSubmit)="createTodo()">
            <div class="mb-3">
              <label class="form-label">Title *</label>
              <input 
                type="text" 
                class="form-control" 
                [(ngModel)]="newTodo.title" 
                name="title"
                placeholder="Enter todo title"
                required>
            </div>
            <div class="mb-3">
              <label class="form-label">Description</label>
              <textarea 
                class="form-control" 
                [(ngModel)]="newTodo.description" 
                name="description"
                rows="3"
                placeholder="Enter todo description"></textarea>
            </div>
            <div class="row">
              <div class="col-md-6 mb-3">
                <label class="form-label">Priority</label>
                <select class="form-select" [(ngModel)]="newTodo.priority" name="priority">
                  <option value="Low">Low</option>
                  <option value="Medium">Medium</option>
                  <option value="High">High</option>
                </select>
              </div>
              <div class="col-md-6 mb-3">
                <label class="form-label">Due Date</label>
                <input 
                  type="date" 
                  class="form-control" 
                  [(ngModel)]="newTodo.dueDate" 
                  name="dueDate">
              </div>
            </div>
            <button type="submit" class="btn btn-primary">
              <i class="bi bi-plus-circle"></i> Add Todo
            </button>
          </form>
        </div>
      </div>

      <!-- Filter Buttons -->
      <div class="btn-group mb-3" role="group">
        <button 
          type="button" 
          class="btn btn-outline-primary"
          [class.active]="filterStatus === 'all'"
          (click)="setFilter('all')">
          All ({{ todos.length }})
        </button>
        <button 
          type="button" 
          class="btn btn-outline-success"
          [class.active]="filterStatus === 'active'"
          (click)="setFilter('active')">
          Active ({{ todos.filter(t => !t.isCompleted).length }})
        </button>
        <button 
          type="button" 
          class="btn btn-outline-secondary"
          [class.active]="filterStatus === 'completed'"
          (click)="setFilter('completed')">
          Completed ({{ todos.filter(t => t.isCompleted).length }})
        </button>
      </div>

      <!-- Todo List -->
      <div class="card">
        <div class="card-header bg-secondary text-white">
          <h5 class="mb-0">Todo List</h5>
        </div>
        <div class="card-body">
          <div *ngIf="filteredTodos.length === 0" class="text-center text-muted py-4">
            <i class="bi bi-inbox" style="font-size: 3rem;"></i>
            <p class="mt-2">No todos found</p>
          </div>

          <div class="list-group">
            <div 
              *ngFor="let todo of filteredTodos" 
              class="list-group-item mb-2"
              [class.list-group-item-success]="todo.isCompleted">
              
              <!-- View Mode -->
              <div *ngIf="editingTodo?.id !== todo.id">
                <div class="d-flex justify-content-between align-items-start">
                  <div class="flex-grow-1">
                    <div class="form-check d-inline-block">
                      <input 
                        class="form-check-input" 
                        type="checkbox" 
                        [checked]="todo.isCompleted"
                        (change)="toggleTodo(todo.id)">
                    </div>
                    <h5 class="d-inline-block mb-1" [class.text-decoration-line-through]="todo.isCompleted">
                      {{ todo.title }}
                    </h5>
                    <span class="ms-2" [ngClass]="getPriorityClass(todo.priority)">
                      {{ todo.priority }}
                    </span>
                    <p class="mb-1 text-muted">{{ todo.description }}</p>
                    <small class="text-muted">
                      Created: {{ todo.createdDate | date:'short' }}
                      <span *ngIf="todo.dueDate"> | Due: {{ todo.dueDate | date:'short' }}</span>
                    </small>
                  </div>
                  <div class="btn-group">
                    <button 
                      class="btn btn-sm btn-outline-primary" 
                      (click)="editTodo(todo)"
                      title="Edit">
                      <i class="bi bi-pencil"></i>
                    </button>
                    <button 
                      class="btn btn-sm btn-outline-danger" 
                      (click)="deleteTodo(todo.id)"
                      title="Delete">
                      <i class="bi bi-trash"></i>
                    </button>
                  </div>
                </div>
              </div>

              <!-- Edit Mode -->
              <div *ngIf="editingTodo?.id === todo.id">
                <div class="mb-2">
                  <label class="form-label">Title</label>
                  <input 
                    type="text" 
                    class="form-control" 
                    [(ngModel)]="editingTodo.title">
                </div>
                <div class="mb-2">
                  <label class="form-label">Description</label>
                  <textarea 
                    class="form-control" 
                    [(ngModel)]="editingTodo.description" 
                    rows="2"></textarea>
                </div>
                <div class="row">
                  <div class="col-md-6 mb-2">
                    <label class="form-label">Priority</label>
                    <select class="form-select" [(ngModel)]="editingTodo.priority">
                      <option value="Low">Low</option>
                      <option value="Medium">Medium</option>
                      <option value="High">High</option>
                    </select>
                  </div>
                  <div class="col-md-6 mb-2">
                    <label class="form-label">Status</label>
                    <select class="form-select" [(ngModel)]="editingTodo.isCompleted">
                      <option [ngValue]="false">Active</option>
                      <option [ngValue]="true">Completed</option>
                    </select>
                  </div>
                </div>
                <div class="btn-group">
                  <button class="btn btn-sm btn-success" (click)="saveTodo()">
                    <i class="bi bi-check"></i> Save
                  </button>
                  <button class="btn btn-sm btn-secondary" (click)="cancelEdit()">
                    <i class="bi bi-x"></i> Cancel
                  </button>
                </div>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</div>
```

Add styles to `src/app/components/todo-list/todo-list.component.css`:

```css
.list-group-item {
  border-left: 4px solid #dee2e6;
  transition: all 0.3s ease;
}

.list-group-item:hover {
  background-color: #f8f9fa;
  border-left-color: #0d6efd;
}

.list-group-item-success {
  border-left-color: #198754;
  background-color: #d1e7dd;
}

.text-decoration-line-through {
  text-decoration: line-through;
  opacity: 0.6;
}

.card {
  box-shadow: 0 0.125rem 0.25rem rgba(0, 0, 0, 0.075);
}

.btn-group .btn {
  margin-left: 2px;
}

.badge {
  font-size: 0.75rem;
  padding: 0.25em 0.5em;
}
```

### Step 6: Update App Module

Update `src/app/app.module.ts`:

```typescript
import { NgModule } from '@angular/core';
import { BrowserModule } from '@angular/platform-browser';
import { HttpClientModule } from '@angular/common/http';
import { FormsModule } from '@angular/forms';

import { AppRoutingModule } from './app-routing.module';
import { AppComponent } from './app.component';
import { TodoListComponent } from './components/todo-list/todo-list.component';

@NgModule({
  declarations: [
    AppComponent,
    TodoListComponent
  ],
  imports: [
    BrowserModule,
    AppRoutingModule,
    HttpClientModule,
    FormsModule
  ],
  providers: [],
  bootstrap: [AppComponent]
})
export class AppModule { }
```

### Step 7: Update App Component

Update `src/app/app.component.html`:

```html
<app-todo-list></app-todo-list>
```

### Step 8: Add Bootstrap Icons

Add to `src/index.html` in the `<head>` section:

```html
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/bootstrap-icons@1.11.0/font/bootstrap-icons.css">
```

### Step 9: Update Environment Configuration

Update `src/app/services/todo.service.ts` with your actual API URL. Check your .NET API console output for the correct port.

### Step 10: Run the Angular Application

```bash
ng serve
```

Navigate to `http://localhost:4200` in your browser.

---

## Testing the Application

### Backend Testing (Using Swagger)

1. Navigate to `https://localhost:7XXX/swagger`
2. Test each endpoint:
   - GET /api/todo - Retrieve all todos
   - GET /api/todo/{id} - Get specific todo
   - POST /api/todo - Create new todo
   - PUT /api/todo/{id} - Update todo
   - DELETE /api/todo/{id} - Delete todo
   - PATCH /api/todo/{id}/toggle - Toggle completion

### Frontend Testing

1. **Create Todo**: Fill in the form and click "Add Todo"
2. **View Todos**: See all todos in the list
3. **Filter Todos**: Use filter buttons (All, Active, Completed)
4. **Toggle Completion**: Click checkbox to mark complete/incomplete
5. **Edit Todo**: Click edit button, modify fields, save
6. **Delete Todo**: Click delete button and confirm

---

## Common Issues and Solutions

### Issue 1: CORS Error

**Problem**: Browser blocks API requests

**Solution**: Ensure CORS is properly configured in `Program.cs`:

```csharp
app.UseCors("AllowAngularApp");
```

### Issue 2: Database Connection Failed

**Problem**: Cannot connect to database

**Solution**: 
- Ensure SQL Server LocalDB is installed
- Verify connection string in `appsettings.json`
- Run migrations: `dotnet ef database update`

### Issue 3: API Not Found (404)

**Problem**: Angular can't reach the API

**Solution**: 
- Update API URL in `todo.service.ts` with correct port
- Ensure .NET API is running
- Check browser console for exact error

### Issue 4: Model Binding Failed

**Problem**: Data not saving correctly

**Solution**: 
- Ensure model properties match between Angular and .NET
- Check browser network tab for request payload
- Verify content-type is application/json

---

## Best Practices Implemented

### Backend (.NET)
- ✅ RESTful API design
- ✅ Dependency injection
- ✅ Entity Framework Core
- ✅ Error handling and logging
- ✅ Async/await pattern
- ✅ CORS configuration
- ✅ API documentation (Swagger)

### Frontend (Angular)
- ✅ Component-based architecture
- ✅ Service layer for API calls
- ✅ Type safety with TypeScript
- ✅ Reactive programming with RxJS
- ✅ Error handling
- ✅ Responsive UI with Bootstrap
- ✅ Form validation

---

## Enhancements and Next Steps

### Security
- Add JWT authentication
- Implement authorization
- Validate input on backend
- Sanitize user input

### Features
- Add categories/tags
- Implement search functionality
- Add sorting options
- User accounts and permissions
- Email notifications for due dates

### Performance
- Implement pagination
- Add caching
- Optimize database queries
- Lazy loading for components

### Testing
- Unit tests for services
- Integration tests for API
- E2E tests with Cypress
- Component testing with Jasmine

---

## Deployment

### Backend Deployment (Azure)

```bash
# Publish .NET app
dotnet publish -c Release -o ./publish

# Deploy to Azure App Service
az webapp up --name your-app-name --resource-group your-rg
```

### Frontend Deployment (Azure Static Web Apps)

```bash
# Build Angular app
ng build --configuration production

# Deploy to Azure
az staticwebapp create --name your-app --source ./dist
```

---

## Conclusion

You've successfully built a full-stack CRUD Todo application using Angular and .NET Core! This application demonstrates:

- Modern web development practices
- RESTful API design
- TypeScript and C# integration
- Responsive UI design
- Error handling and validation

### Key Takeaways

1. **.NET Core Web API** provides a robust backend framework
2. **Entity Framework Core** simplifies database operations
3. **Angular** offers a powerful frontend framework
4. **TypeScript** ensures type safety
5. **Bootstrap** accelerates UI development

### Resources

- [Angular Documentation](https://angular.io/docs)
- [.NET Documentation](https://docs.microsoft.com/en-us/dotnet/)
- [Entity Framework Core](https://docs.microsoft.com/en-us/ef/core/)
- [Bootstrap Documentation](https://getbootstrap.com/docs/)

Happy coding! 🚀
