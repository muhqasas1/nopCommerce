# Glossary and Cheat Sheet

## Architecture Patterns

### Layered Architecture
**Definition**: A software architecture pattern that organizes code into horizontal layers, each with specific responsibilities.

**nopCommerce Implementation**:
- **Presentation Layer**: Controllers, views, web middleware
- **Business Logic Layer**: Services, business rules
- **Data Access Layer**: Repositories, database context
- **Domain Layer**: Entities, interfaces, core abstractions

**Benefits**:
- Separation of concerns
- Maintainability
- Testability
- Team development

### Repository Pattern
**Definition**: A design pattern that abstracts data access logic and provides a uniform interface for accessing data.

**nopCommerce Implementation**:
```csharp
public interface IRepository<T> where T : BaseEntity
{
    Task<T> GetByIdAsync(int? id);
    Task<IList<T>> GetAllAsync();
    Task<int> InsertAsync(T entity);
    Task<int> UpdateAsync(T entity);
    Task<int> DeleteAsync(T entity);
}
```

**Benefits**:
- Type safety
- Consistency
- Testability
- Flexibility

### Service Layer Pattern
**Definition**: A design pattern that encapsulates business logic in service classes, providing a clear interface for business operations.

**nopCommerce Implementation**:
```csharp
public interface ICustomerService
{
    Task<Customer> GetCustomerByIdAsync(int customerId);
    Task<Customer> CreateCustomerAsync(Customer customer);
    Task<Customer> UpdateCustomerAsync(Customer customer);
}
```

**Benefits**:
- Business logic encapsulation
- Reusability
- Testability
- Maintainability

---

## Dependency Injection

### Service Lifetimes

#### Singleton
**Definition**: One instance for the entire application lifetime.

**Use Cases**:
- Configuration services
- Cache managers
- Loggers
- Stateless services

**Registration**:
```csharp
services.AddSingleton<ICacheManager, CacheManager>();
```

#### Scoped
**Definition**: One instance per HTTP request.

**Use Cases**:
- Repositories
- Business services
- Database contexts
- Per-request services

**Registration**:
```csharp
services.AddScoped<ICustomerService, CustomerService>();
```

#### Transient
**Definition**: New instance every time it's requested.

**Use Cases**:
- Lightweight services
- Utilities
- Factories
- Stateless helpers

**Registration**:
```csharp
services.AddTransient<IEmailService, EmailService>();
```

### Constructor Injection
**Definition**: Dependencies are provided through the constructor.

**Benefits**:
- Immutability
- Testability
- Clear dependencies
- Compile-time safety

**Example**:
```csharp
public class CustomerService : ICustomerService
{
    private readonly IRepository<Customer> _repository;
    private readonly ICacheManager _cacheManager;

    public CustomerService(IRepository<Customer> repository, ICacheManager cacheManager)
    {
        _repository = repository;
        _cacheManager = cacheManager;
    }
}
```

---

## Caching Strategies

### Cache Levels

#### Static Cache
**Definition**: Long-term caching between HTTP requests.

**Use Cases**:
- Configuration data
- Reference data
- Frequently accessed data
- Expensive computations

**Implementation**:
```csharp
public async Task<Customer> GetCustomerByIdAsync(int customerId)
{
    var cacheKey = _staticCacheManager.PrepareKeyForDefaultCache(
        NopCustomerDefaults.CustomerByIdCacheKey, customerId);

    return await _staticCacheManager.GetAsync(cacheKey, async () =>
    {
        return await _customerRepository.GetByIdAsync(customerId);
    });
}
```

#### Short-term Cache
**Definition**: Per-request caching.

**Use Cases**:
- Request-specific data
- Temporary calculations
- Per-user data
- Session data

**Implementation**:
```csharp
public async Task<Customer> GetCurrentCustomerAsync()
{
    var customerId = _shortTermCacheManager.Get<int?>(NopCustomerDefaults.CustomerIdAttributeName);
    if (customerId.HasValue)
    {
        return await GetCustomerByIdAsync(customerId.Value);
    }
    // Load from authentication...
}
```

### Cache Key Design

#### Naming Conventions
- **Namespace Prefix**: `Nop.customer.`
- **Operation Suffix**: `.id-`, `.email-`
- **Parameter Placeholders**: `{0}`, `{1}`
- **Prefix Constants**: Reusable prefixes

#### Example
```csharp
public static partial class NopCustomerDefaults
{
    public static CacheKey CustomerByIdCacheKey => new("Nop.customer.id-{0}", CustomerByIdPrefix);
    public static string CustomerByIdPrefix => "Nop.customer.id-";
}
```

### Cache Invalidation
**Definition**: The process of removing cached data when it becomes stale.

**Strategies**:
- **Time-based**: TTL (Time To Live)
- **Event-based**: Invalidate on data changes
- **Manual**: Explicit cache clearing
- **Pattern-based**: Remove by key pattern

**Implementation**:
```csharp
public async Task UpdateCustomerAsync(Customer customer)
{
    await _customerRepository.UpdateAsync(customer);
    
    // Clear cache
    await _staticCacheManager.RemoveByPrefixAsync(NopCustomerDefaults.CustomerByIdPrefix);
    
    // Publish event
    await _eventPublisher.PublishAsync(new CustomerUpdatedEvent(customer));
}
```

---

## Middleware and Pipeline

### Middleware Pipeline
**Definition**: A sequence of middleware components that process HTTP requests and responses.

**Order Matters**:
1. Exception handling
2. HTTPS redirect
3. Authentication
4. Authorization
5. Static files
6. Routing
7. Endpoints

### Custom Middleware
**Definition**: Application-specific middleware that handles cross-cutting concerns.

**Implementation**:
```csharp
public class CustomMiddleware
{
    private readonly RequestDelegate _next;

    public CustomMiddleware(RequestDelegate next)
    {
        _next = next;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        // Before next middleware
        await _next(context);
        // After next middleware
    }
}
```

### Action Filters
**Definition**: Attributes that can be applied to controllers or actions to handle cross-cutting concerns.

**Types**:
- **Authorization Filters**: Handle authentication and authorization
- **Action Filters**: Handle action execution
- **Result Filters**: Handle action results
- **Exception Filters**: Handle exceptions

**Implementation**:
```csharp
public class ValidateModelAttribute : ActionFilterAttribute
{
    public override void OnActionExecuting(ActionExecutingContext context)
    {
        if (!context.ModelState.IsValid)
        {
            context.Result = new BadRequestObjectResult(context.ModelState);
        }
    }
}
```

---

## Event-Driven Architecture

### Domain Events
**Definition**: Events that represent something important that happened in the domain.

**Benefits**:
- Loose coupling
- Extensibility
- Audit trail
- Integration

**Implementation**:
```csharp
public class CustomerRegisteredEvent : INotification
{
    public Customer Customer { get; }

    public CustomerRegisteredEvent(Customer customer)
    {
        Customer = customer;
    }
}
```

### Event Publishing
**Definition**: The process of publishing domain events when something important happens.

**Implementation**:
```csharp
public async Task<Customer> RegisterCustomerAsync(CustomerRegistrationRequest request)
{
    var customer = new Customer { /* ... */ };
    await _customerRepository.InsertAsync(customer);
    
    // Publish event
    await _eventPublisher.PublishAsync(new CustomerRegisteredEvent(customer));
    
    return customer;
}
```

### Event Handling
**Definition**: The process of handling domain events when they are published.

**Implementation**:
```csharp
public class CustomerRegisteredEventHandler : INotificationHandler<CustomerRegisteredEvent>
{
    public async Task Handle(CustomerRegisteredEvent notification, CancellationToken cancellationToken)
    {
        // Send welcome email
        // Create user profile
        // Log registration
    }
}
```

---

## Performance Optimization

### Async/Await
**Definition**: A pattern for writing asynchronous code that doesn't block threads.

**Benefits**:
- Better scalability
- Responsive UI
- Resource efficiency
- Better user experience

**Best Practices**:
- Use async/await consistently
- Avoid blocking async calls
- Use ConfigureAwait(false) in libraries
- Handle exceptions properly

### Database Optimization
**Definition**: Techniques to improve database performance.

**Strategies**:
- **Indexing**: Proper database indexes
- **Query Optimization**: Efficient queries
- **Connection Pooling**: Reuse connections
- **Caching**: Reduce database calls

### Memory Management
**Definition**: Techniques to optimize memory usage.

**Strategies**:
- **Object Pooling**: Reuse objects
- **Lazy Loading**: Load data when needed
- **Disposal**: Proper resource cleanup
- **Garbage Collection**: Optimize GC pressure

---

## Security Patterns

### Authentication
**Definition**: The process of verifying who a user is.

**nopCommerce Implementation**:
- Cookie-based authentication
- External authentication providers
- Multi-factor authentication
- Impersonation support

### Authorization
**Definition**: The process of determining what a user can do.

**nopCommerce Implementation**:
- Role-based authorization
- Permission-based authorization
- Resource-based authorization
- Policy-based authorization

### Input Validation
**Definition**: The process of validating user input.

**Strategies**:
- **Model Validation**: Data annotations
- **Custom Validators**: Business rule validation
- **Sanitization**: Clean input data
- **Encoding**: Prevent injection attacks

---

## Testing Patterns

### Unit Testing
**Definition**: Testing individual components in isolation.

**Benefits**:
- Fast execution
- Easy debugging
- Reliable results
- Early bug detection

**Implementation**:
```csharp
[Test]
public async Task GetCustomerByIdAsync_ValidId_ReturnsCustomer()
{
    // Arrange
    var customerId = 1;
    var expectedCustomer = new Customer { Id = customerId };
    _mockRepository.Setup(r => r.GetByIdAsync(customerId))
        .ReturnsAsync(expectedCustomer);

    // Act
    var result = await _customerService.GetCustomerByIdAsync(customerId);

    // Assert
    Assert.AreEqual(expectedCustomer, result);
}
```

### Integration Testing
**Definition**: Testing how components work together.

**Benefits**:
- Real-world scenarios
- End-to-end testing
- System validation
- Performance testing

### Mocking
**Definition**: Creating fake objects that simulate real dependencies.

**Benefits**:
- Isolated testing
- Controlled behavior
- Fast execution
- Easy setup

---

## Common Anti-Patterns

### Service Locator
**Definition**: Using a service locator to resolve dependencies.

**Problem**:
- Hidden dependencies
- Hard to test
- Tight coupling
- Runtime errors

**Solution**:
- Use constructor injection
- Depend on interfaces
- Use DI container

### God Object
**Definition**: A class that has too many responsibilities.

**Problem**:
- Hard to maintain
- Violates SRP
- Difficult to test
- Poor performance

**Solution**:
- Split into smaller classes
- Single responsibility
- Clear interfaces
- Proper abstraction

### Anemic Domain Model
**Definition**: Domain objects with only data and no behavior.

**Problem**:
- Business logic scattered
- No encapsulation
- Hard to maintain
- Poor design

**Solution**:
- Rich domain models
- Encapsulate behavior
- Business logic in entities
- Proper abstraction

---

## Quick Reference

### Service Registration
```csharp
// Singleton
services.AddSingleton<IService, Service>();

// Scoped
services.AddScoped<IService, Service>();

// Transient
services.AddTransient<IService, Service>();
```

### Caching
```csharp
// Get with fallback
var result = await _cache.GetAsync(key, async () => await LoadDataAsync());

// Set
await _cache.SetAsync(key, data);

// Remove
await _cache.RemoveAsync(key);

// Remove by prefix
await _cache.RemoveByPrefixAsync(prefix);
```

### Middleware
```csharp
// Register middleware
app.UseMiddleware<CustomMiddleware>();

// Use built-in middleware
app.UseAuthentication();
app.UseAuthorization();
app.UseStaticFiles();
```

### Action Filters
```csharp
// Apply to controller
[ValidateModel]
public class CustomerController : Controller
{
    // Actions
}

// Apply to action
[HttpGet]
[ValidateModel]
public async Task<IActionResult> GetCustomer(int id)
{
    // Action implementation
}
```

---

*This cheat sheet provides quick reference for the most important patterns and concepts covered in the nopCommerce learning guide.*