# Practical Exercises and Mini-Tasks

## Learning Philosophy
These exercises follow the **"Learn by Doing"** approach, where you'll implement real-world patterns based on nopCommerce's architecture.

---

## Chapter 1: Project Structure and Architecture

### Exercise 1.1: Code Navigation Challenge
**Objective**: Learn to navigate a large enterprise codebase

**Tasks**:
1. Find the `Customer` entity in the domain layer
2. Trace how a customer is created from controller to database
3. Identify all the layers involved in the process
4. Document the data flow with a diagram

**Expected Outcome**: Understanding of layered architecture and data flow

### Exercise 1.2: Create Your Own Entity
**Objective**: Practice domain modeling

**Tasks**:
1. Create a `Product` entity that inherits from `BaseEntity`
2. Add properties: Name, Description, Price, CategoryId
3. Create a `ProductCategory` entity
4. Implement the relationship between Product and ProductCategory

**Code Template**:
```csharp
// In Nop.Core/Domain/Catalog/Product.cs
public partial class Product : BaseEntity
{
    public string Name { get; set; }
    public string Description { get; set; }
    public decimal Price { get; set; }
    public int CategoryId { get; set; }
    public virtual ProductCategory Category { get; set; }
}
```

### Exercise 1.3: Architecture Analysis
**Objective**: Understand architectural decisions

**Tasks**:
1. Analyze why nopCommerce uses interfaces for all services
2. Explain the benefits of the repository pattern
3. Identify potential issues with the current architecture
4. Suggest improvements for scalability

---

## Chapter 2: Dependency Injection and Service Layer

### Exercise 2.1: Create a Service Interface
**Objective**: Practice interface design

**Tasks**:
1. Create `IProductService` interface
2. Define methods for CRUD operations
3. Add methods for complex queries
4. Include business operation methods

**Code Template**:
```csharp
public partial interface IProductService
{
    Task<Product> GetProductByIdAsync(int productId);
    Task<IPagedList<Product>> GetProductsByCategoryAsync(int categoryId, int pageIndex, int pageSize);
    Task<Product> CreateProductAsync(Product product);
    Task<Product> UpdateProductAsync(Product product);
    Task DeleteProductAsync(int productId);
    Task<IPagedList<Product>> SearchProductsAsync(string searchTerm, int pageIndex, int pageSize);
}
```

### Exercise 2.2: Implement the Service
**Objective**: Practice service implementation

**Tasks**:
1. Implement `ProductService` class
2. Use constructor injection for dependencies
3. Implement caching for read operations
4. Add event publishing for write operations

**Code Template**:
```csharp
public partial class ProductService : IProductService
{
    private readonly IRepository<Product> _productRepository;
    private readonly IStaticCacheManager _staticCacheManager;
    private readonly IEventPublisher _eventPublisher;

    public ProductService(
        IRepository<Product> productRepository,
        IStaticCacheManager staticCacheManager,
        IEventPublisher eventPublisher)
    {
        _productRepository = productRepository;
        _staticCacheManager = staticCacheManager;
        _eventPublisher = eventPublisher;
    }

    public async Task<Product> GetProductByIdAsync(int productId)
    {
        if (productId == 0)
            return null;

        var cacheKey = _staticCacheManager.PrepareKeyForDefaultCache(
            NopProductDefaults.ProductByIdCacheKey, productId);

        return await _staticCacheManager.GetAsync(cacheKey, async () =>
        {
            return await _productRepository.GetByIdAsync(productId);
        });
    }
}
```

### Exercise 2.3: Service Registration
**Objective**: Practice DI container configuration

**Tasks**:
1. Register your service in the DI container
2. Choose appropriate service lifetime
3. Test the service registration
4. Verify dependency resolution

---

## Chapter 3: Caching Strategies

### Exercise 3.1: Design Cache Keys
**Objective**: Practice cache key design

**Tasks**:
1. Create cache key constants for Product service
2. Design hierarchical key structure
3. Include parameters in cache keys
4. Set appropriate cache times

**Code Template**:
```csharp
public static partial class NopProductDefaults
{
    public static CacheKey ProductByIdCacheKey => new("Nop.product.id-{0}", ProductByIdPrefix);
    public static CacheKey ProductsByCategoryCacheKey => new("Nop.product.category-{0}-{1}-{2}", ProductsByCategoryPrefix);
    public static CacheKey ProductSearchCacheKey => new("Nop.product.search-{0}-{1}-{2}", ProductSearchPrefix);
    
    public static string ProductByIdPrefix => "Nop.product.id-";
    public static string ProductsByCategoryPrefix => "Nop.product.category-";
    public static string ProductSearchPrefix => "Nop.product.search-";
}
```

### Exercise 3.2: Implement Caching
**Objective**: Practice caching implementation

**Tasks**:
1. Add caching to all read operations
2. Implement cache invalidation for write operations
3. Use different cache times for different data types
4. Add cache statistics

**Code Template**:
```csharp
public async Task<Product> GetProductByIdAsync(int productId)
{
    if (productId == 0)
        return null;

    var cacheKey = _staticCacheManager.PrepareKeyForDefaultCache(
        NopProductDefaults.ProductByIdCacheKey, productId);

    return await _staticCacheManager.GetAsync(cacheKey, async () =>
    {
        return await _productRepository.GetByIdAsync(productId);
    });
}

public async Task<Product> UpdateProductAsync(Product product)
{
    if (product == null)
        throw new ArgumentNullException(nameof(product));

    await _productRepository.UpdateAsync(product);

    // Clear cache
    await _staticCacheManager.RemoveByPrefixAsync(NopProductDefaults.ProductByIdPrefix);
    await _staticCacheManager.RemoveByPrefixAsync(NopProductDefaults.ProductsByCategoryPrefix);

    // Publish event
    await _eventPublisher.PublishAsync(new ProductUpdatedEvent(product));

    return product;
}
```

### Exercise 3.3: Cache Performance Testing
**Objective**: Practice performance optimization

**Tasks**:
1. Create a performance test for cached vs non-cached operations
2. Measure cache hit rates
3. Test cache invalidation performance
4. Optimize cache configuration

---

## Chapter 4: Middleware and Request Pipeline

### Exercise 4.1: Create Custom Middleware
**Objective**: Practice middleware development

**Tasks**:
1. Create request logging middleware
2. Implement response time tracking
3. Add user activity logging
4. Handle errors gracefully

**Code Template**:
```csharp
public class RequestLoggingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<RequestLoggingMiddleware> _logger;

    public RequestLoggingMiddleware(RequestDelegate next, ILogger<RequestLoggingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        var stopwatch = Stopwatch.StartNew();
        
        try
        {
            await _next(context);
        }
        finally
        {
            stopwatch.Stop();
            
            _logger.LogInformation(
                "Request {Method} {Path} completed in {ElapsedMs}ms with status {StatusCode}",
                context.Request.Method,
                context.Request.Path,
                stopwatch.ElapsedMilliseconds,
                context.Response.StatusCode);
        }
    }
}
```

### Exercise 4.2: Create Action Filter
**Objective**: Practice action filter development

**Tasks**:
1. Create a validation action filter
2. Implement custom authorization logic
3. Add business operation logging
4. Handle validation errors

**Code Template**:
```csharp
public class ValidateProductFilter : IAsyncActionFilter
{
    private readonly IProductService _productService;

    public ValidateProductFilter(IProductService productService)
    {
        _productService = productService;
    }

    public async Task OnActionExecutionAsync(ActionExecutingContext context, ActionExecutionDelegate next)
    {
        if (context.ActionArguments.ContainsKey("productId"))
        {
            var productId = (int)context.ActionArguments["productId"];
            var product = await _productService.GetProductByIdAsync(productId);
            
            if (product == null)
            {
                context.Result = new NotFoundResult();
                return;
            }
        }

        await next();
    }
}
```

### Exercise 4.3: Pipeline Configuration
**Objective**: Practice pipeline setup

**Tasks**:
1. Configure your middleware in the pipeline
2. Set correct execution order
3. Add error handling
4. Test the pipeline

---

## Advanced Exercises

### Exercise A.1: Event-Driven Architecture
**Objective**: Implement domain events

**Tasks**:
1. Create domain events for Product operations
2. Implement event handlers
3. Add event publishing to services
4. Test event flow

### Exercise A.2: Plugin System
**Objective**: Understand extensibility

**Tasks**:
1. Create a simple plugin interface
2. Implement a plugin
3. Register the plugin dynamically
4. Test plugin functionality

### Exercise A.3: Multi-tenancy
**Objective**: Implement store context

**Tasks**:
1. Create store context service
2. Implement store-specific data filtering
3. Add store switching functionality
4. Test multi-store scenarios

---

## Testing Your Implementation

### Unit Testing
1. Test service methods independently
2. Mock dependencies properly
3. Test error scenarios
4. Verify business logic

### Integration Testing
1. Test service interactions
2. Verify database operations
3. Test caching behavior
4. Validate event publishing

### Performance Testing
1. Load test your services
2. Measure cache performance
3. Test under high concurrency
4. Optimize based on results

---

## Code Review Checklist

### Architecture
- [ ] Follows layered architecture
- [ ] Uses dependency injection
- [ ] Implements proper separation of concerns
- [ ] Follows naming conventions

### Performance
- [ ] Implements caching where appropriate
- [ ] Uses async/await consistently
- [ ] Optimizes database queries
- [ ] Handles errors gracefully

### Security
- [ ] Validates input parameters
- [ ] Implements proper authorization
- [ ] Handles sensitive data correctly
- [ ] Logs security events

### Maintainability
- [ ] Code is well-documented
- [ ] Follows consistent patterns
- [ ] Is easily testable
- [ ] Handles edge cases

---

## Next Steps

After completing these exercises:

1. **Review Your Code**: Compare with nopCommerce implementations
2. **Identify Patterns**: Look for common patterns in your code
3. **Optimize**: Improve performance and maintainability
4. **Extend**: Add more complex features
5. **Share**: Discuss your implementation with others

---

*Remember: The goal is not just to complete the exercises, but to understand the underlying principles and apply them to your own projects.*