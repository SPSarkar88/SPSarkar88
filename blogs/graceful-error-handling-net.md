# Graceful Error Handling in .NET 8 Web APIs

## Introduction

In modern web development, how you handle errors is just as important as how you handle success. A robust error handling strategy ensures your API remains reliable, secure, and easy to consume. In .NET 8 (and 9), Microsoft has provided powerful tools to implement graceful error handling with minimal boilerplate.

In this guide, we'll walk through building a production-ready error handling mechanism using Global Exception Handling, ProblemDetails, and Custom Exceptions.

## Why "Graceful" Error Handling?

Graceful error handling means:
1.  **Consistency**: All errors return a standardized response structure.
2.  **Security**: Sensitive internal details (stack traces, database info) are hidden from clients in production.
3.  **Observability**: Errors are logged with sufficient context for developers to diagnose issues.
4.  **Usability**: Clients receive meaningful error messages they can act upon.

## The Old Way vs. The Modern Way

Historically, developers used middleware with try-catch blocks or filters. While valid, .NET 8 standardizes this with the `IExceptionHandler` interface, making it cleaner and more testable.

### Step 1: Define Custom Exceptions

First, let's create a base exception class and a few specific exceptions. This allows us to handle known domain errors differently from unexpected crashes.

```csharp
// Exceptions/DomainException.cs
public abstract class DomainException : Exception
{
    protected DomainException(string message) : base(message) { }
}

// Exceptions/NotFoundException.cs
public class NotFoundException : DomainException
{
    public NotFoundException(string message) : base(message) { }
}

// Exceptions/ValidationException.cs
public class ValidationException : DomainException
{
    public ValidationException(string message) : base(message) { }
}
```

### Step 2: Implement IExceptionHandler

This is the heart of our error handling. We'll implement `IExceptionHandler` to intercept exceptions and convert them into standardized `ProblemDetails` responses.

```csharp
using Microsoft.AspNetCore.Diagnostics;
using Microsoft.AspNetCore.Mvc;

public class GlobalExceptionHandler : IExceptionHandler
{
    private readonly ILogger<GlobalExceptionHandler> _logger;

    public GlobalExceptionHandler(ILogger<GlobalExceptionHandler> logger)
    {
        _logger = logger;
    }

    // `TryHandleAsync` is the contract method. If we return true, the pipeline stops here.
    // If we return false, it continues to the next handler (if any).

    public async ValueTask<bool> TryHandleAsync(
        HttpContext httpContext,
        Exception exception,
        CancellationToken cancellationToken)
    {
        _logger.LogError(
            exception, "Exception occurred: {Message}", exception.Message);

        // We use ProblemDetails (RFC 7807) to return a standard error format.
        // This includes standard fields like Title, Status, and Type.
        var problemDetails = new ProblemDetails
        {
            Status = StatusCodes.Status500InternalServerError,
            Title = "Server Error",
            Type = "https://datatracker.ietf.org/doc/html/rfc7231#section-6.6.1"
        };

        switch (exception)
        {
            case NotFoundException notFound:
                problemDetails.Status = StatusCodes.Status404NotFound;
                problemDetails.Title = "Not Found";
                problemDetails.Detail = notFound.Message;
                problemDetails.Type = "https://datatracker.ietf.org/doc/html/rfc7231#section-6.5.4";
                break;
                
            case ValidationException validation:
                problemDetails.Status = StatusCodes.Status400BadRequest;
                problemDetails.Title = "Bad Request";
                problemDetails.Detail = validation.Message;
                problemDetails.Type = "https://datatracker.ietf.org/doc/html/rfc7231#section-6.5.1";
                break;

            default:
                problemDetails.Detail = "An unexpected error occurred.";
                break;
        }

        httpContext.Response.StatusCode = problemDetails.Status ?? StatusCodes.Status500InternalServerError;

        await httpContext.Response
            .WriteAsJsonAsync(problemDetails, cancellationToken);

        return true;
    }
}
```

### Step 3: Register Services in Program.cs

Now, we need to register our handler and configure the middleware pipeline.

```csharp
var builder = WebApplication.CreateBuilder(args);

// Add services
builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

// Register the exception handler
builder.Services.AddExceptionHandler<GlobalExceptionHandler>();
builder.Services.AddProblemDetails();

var app = builder.Build();

// Configure Middleware
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

// Ensure this is called before other middleware
app.UseExceptionHandler(); 

app.UseHttpsRedirection();
app.MapControllers();

app.Run();
```

## Enhancing with Serilog

Built-in logging is good, but Serilog provides structured logging that is essential for analyzing errors in production.

1.  **Install Packages**:
    ```bash
    dotnet add package Serilog.AspNetCore
    ```

2.  **Configure in Program.cs**:

```csharp
using Serilog;

// ... at the start of Program.cs
Log.Logger = new LoggerConfiguration()
    .WriteTo.Console()
    .CreateBootstrapLogger();

builder.Host.UseSerilog((context, services, configuration) => configuration
    .ReadFrom.Configuration(context.Configuration)
    .ReadFrom.Services(services)
    .Enrich.FromLogContext()
    .WriteTo.Console());
```

## Testing the Error Handling

Let's see it in action with a sample controller.

```csharp
[ApiController]
[Route("[controller]")]
public class ProductsController : ControllerBase
{
    [HttpGet("{id}")]
    public IActionResult Get(int id)
    {
        if (id < 0)
        {
            throw new ValidationException("ID cannot be negative");
        }

        if (id == 99)
        {
            throw new NotFoundException($"Product with ID {id} not found");
        }

        // Simulating unexpected error
        if (id == 0)
        {
            throw new Exception("Database connection failed");
        }

        return Ok(new { Id = id, Name = "Sample Product" });
    }
}
```

### Results

**Request**: `GET /products/-1`
**Response**:
```json
{
  "type": "https://datatracker.ietf.org/doc/html/rfc7231#section-6.5.1",
  "title": "Bad Request",
  "status": 400,
  "detail": "ID cannot be negative"
}
```

**Request**: `GET /products/0` (Unexpected Error)
**Response**:
```json
{
  "type": "https://datatracker.ietf.org/doc/html/rfc7231#section-6.6.1",
  "title": "Server Error",
  "status": 500,
  "detail": "An unexpected error occurred."
}
```

## Taking it to Production 

The implementation above is functional, but in the production environment that "functional" isn't enough for distributed systems. To make this code production-hardened, we need to address **Observability**, **Traceability**, and **Security context**.

Here is how we evolve the basic handler into a robust infrastructure component.

### 1. Correlation IDs & Traceability
When a user reports an error, they shouldn't just say "it failed." They should provide a **Trace ID** that allows you to find that specific request across all your logs and microservices.

We inject the `Activity.Current?.Id` or `HttpContext.TraceIdentifier` into the `ProblemDetails` extensions.

### 2. OpenTelemetry Integration
Modern .NET stacks rely on OpenTelemetry (OTel). If you catch and swallow an exception in middleware, your APM (Azure Monitor, Datadog, Jaeger) might see the request as "Successful" (HTTP 200/500 handled) unless you explicitly mark the **Activity** as failed.

**Why is this critical?**
By default, if you handle an exception in middleware (using try/catch or `IExceptionHandler`) and return a nice JSON response, OpenTelemetry often sees the request as "Successful" because the server didn't crash and returned a valid response.

We explicitly grab the current **Activity** (which represents the OTel span) and force its status to **Error**. This ensures your monitoring dashboards turn red for this request, even though the user got a "graceful" green response.

### 3. Environment-Aware Details
**Never** leak stack traces in Production. It's a security vulnerability. However, in Development, you need them for speed. We can inject `IHostEnvironment` to conditionally show details.

### The "Senior" Implementation

Here is the significantly upgraded `GlobalExceptionHandler`:

```csharp
public class ProductionExceptionHandler : IExceptionHandler
{
    private readonly ILogger<ProductionExceptionHandler> _logger;
    private readonly IHostEnvironment _env;

    public ProductionExceptionHandler(
        ILogger<ProductionExceptionHandler> logger, 
        IHostEnvironment env)
    {
        _logger = logger;
        _env = env; // Injected to check if we are in Dev or Prod
    }

    public async ValueTask<bool> TryHandleAsync(
        HttpContext httpContext,
        Exception exception,
        CancellationToken cancellationToken)
    {
        // 1. Mark the OpenTelemetry Activity as Error
        // This ensures your APM (like Azure App Insights) marks the request as "Failed"
        // even though we return a "success" (handled) response to the pipeline.
        var activity = System.Diagnostics.Activity.Current;
        activity?.SetStatus(System.Diagnostics.ActivityStatusCode.Error, exception.Message);
        
        // 2. Log with structured context (Serilog will pick up these properties)
        _logger.LogError(
            exception, 
            "Exception occurred: {Message} TraceId: {TraceId}", 
            exception.Message, 
            activity?.Id); // Structured logging for easier querying in Kibana/Splunk

        var problemDetails = new ProblemDetails
        {
            Status = StatusCodes.Status500InternalServerError,
            Title = "An error occurred while processing your request",
            Type = "https://datatracker.ietf.org/doc/html/rfc7231#section-6.6.1"
            // Instance = httpContext.Request.Path // Optional: helps identify the route
        };

        // 3. Map Domain Exceptions to specific HTTP Statuses
        if (exception is DomainException domainEx)
        {
            problemDetails.Title = domainEx.Message; // Safe for client
            
            problemDetails.Status = exception switch {
                NotFoundException => StatusCodes.Status404NotFound,
                ValidationException => StatusCodes.Status400BadRequest,
                _ => StatusCodes.Status400BadRequest
            };
        }

        // 4. Add TraceId for Frontend/Support correlation
        var traceId = activity?.Id ?? httpContext.TraceIdentifier;
        problemDetails.Extensions["traceId"] = traceId;

        // 5. Environment-Specific Logic (Security)
        if (_env.IsDevelopment())
        {
            problemDetails.Extensions["stackTrace"] = exception.StackTrace;
            problemDetails.Detail = exception.ToString(); // Full details in Dev
        }
        else
        {
            // In Prod, keep it vague unless it's a domain exception
            problemDetails.Detail = exception is DomainException 
                ? exception.Message 
                : "Please contact support and provide the Trace ID.";
        }

        httpContext.Response.StatusCode = problemDetails.Status ?? StatusCodes.Status500InternalServerError;

        await httpContext.Response
            .WriteAsJsonAsync(problemDetails, cancellationToken);

        return true;
    }
}
```

## Summary of Improvements

| Feature | Junior/Mid Implementation | Senior/Production Implementation |
| :--- | :--- | :--- |
| **Traceability** | Logs the error message. | Returns a `traceId` to the client and links it to backend logs. |
| **Observability** | Exception might be "swallowed" by middleware. | Explicitly sets `Activity.Current.Status` to Error for APM tools. |
| **Security** | Hardcoded details or risky exposure. | Strict separation of Dev (verbose) vs Prod (opaque) details. |
| **Standards** | Custom JSON format. | Strictly follows RFC 7807 `ProblemDetails`. |

## Conclusion

By using `IExceptionHandler` and `ProblemDetails`, we've created a centralized, compliant, and maintainable error handling strategy. This implementation keeps your controllers clean and ensures your API clients always receive consistent feedback.
