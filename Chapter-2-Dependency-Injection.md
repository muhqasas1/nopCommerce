# Chapter 2: Dependency Injection and Service Layer Patterns

## Learning Objectives
By the end of this chapter, you will understand:
- How nopCommerce implements advanced dependency injection patterns
- The service layer pattern and its enterprise benefits
- How to design loosely coupled, testable services
- The role of interfaces in enterprise applications

---

## 2.1 Starting with Real Code: The Service Registration

Let's examine how nopCommerce configures its dependency injection container.

### Service Registration Pattern

Navigate to `/workspace/src/Presentation/Nop.Web.Framework/Infrastructure/Extensions/ServiceCollectionExtensions.cs`:

```csharp
public static void ConfigureApplicationServices(this IServiceCollection services,
    WebApplicationBuilder builder)
{
    // Register all services
    var typeFinder = Singleton<ITypeFinder>.Instance;
    
    // Register services
    services.RegisterServices(typeFinder);
    
    // Register data providers
    services.RegisterDataProviders(typeFinder);
    
    // Register plugins
    services.RegisterPlugins(typeFinder);
}
```

**Key Observations:**
1. **Type Discovery**: Uses reflection to find service implementations
2. **Automatic Registration**: Services are registered automatically
3. **Plugin Support**: Third-party plugins are registered dynamically
4. **Layered Registration**: Different types of services registered separately

---

## 2.2 The Service Registration Implementation

Let's dive deeper into how services are actually registered:

```csharp
private static void RegisterServices(this IServiceCollection services, ITypeFinder typeFinder)
{
    // Register services
    var servicesRegistrations = typeFinder
        .FindClassesOfType<IDependencyRegistrar>()
        .Select(type => (IDependencyRegistrar)Activator.CreateInstance(type))
        .OrderBy(registrar => registrar.Order)
        .ToList();

    foreach (var registrar in servicesRegistrations)
        registrar.Register(services, typeFinder);
}
```

**Pattern Analysis:**
1. **Convention over Configuration**: Services follow naming conventions
2. **Ordered Registration**: Services registered in specific order
3. **Extensibility**: New services can be added without modifying core code
4. **Type Safety**: Compile-time checking of service types

---

## 2.3 The Service Layer Pattern in Action

### Service Interface Design

Let's examine a complex service interface:

Navigate to `/workspace/src/Libraries/Nop.Services/Customers/ICustomerService.cs`:

```csharp
public partial interface ICustomerService
{
    // Basic CRUD operations
    Task<Customer> GetCustomerByIdAsync(int customerId);
    Task<Customer> GetCustomerByGuidAsync(Guid customerGuid);
    Task<Customer> GetCustomerByEmailAsync(string email);
    
    // Complex queries with pagination
    Task<IPagedList<Customer>> GetAllCustomersAsync(
        CustomerRoleIds? customerRoleIds = null,
        string email = null,
        string username = null,
        string firstName = null,
        string lastName = null,
        int dayOfBirth = 0,
        int monthOfBirth = 0,
        int company = 0,
        string phone = null,
        string zipPostalCode = null,
        int ipAddressId = 0,
        int pageIndex = 0,
        int pageSize = int.MaxValue,
        bool getOnlyTotalCount = false);
    
    // Business operations
    Task<CustomerLoginResults> ValidateCustomerAsync(string usernameOrEmail, string password);
    Task<CustomerRegistrationResult> RegisterCustomerAsync(CustomerRegistrationRequest request);
    Task<PasswordChangeResult> ChangePasswordAsync(ChangePasswordRequest request);
}
```

**Interface Design Principles:**
1. **Single Responsibility**: Each method has one clear purpose
2. **Async by Default**: All operations are asynchronous
3. **Flexible Parameters**: Optional parameters for different use cases
4. **Return Types**: Specific result types for business operations

---

## 2.4 Service Implementation Patterns

### Constructor Injection Pattern

Navigate to `/workspace/src/Libraries/Nop.Services/Customers/CustomerService.cs`:

```csharp
public partial class CustomerService : ICustomerService
{
    #region Fields

    protected readonly CustomerSettings _customerSettings;
    protected readonly IEventPublisher _eventPublisher;
    protected readonly IGenericAttributeService _genericAttributeService;
    protected readonly INopDataProvider _dataProvider;
    protected readonly IRepository<Customer> _customerRepository;
    protected readonly IRepository<CustomerRole> _customerRoleRepository;
    protected readonly IShortTermCacheManager _shortTermCacheManager;
    protected readonly IStaticCacheManager _staticCacheManager;
    protected readonly IWorkContext _workContext;

    #endregion

    #region Ctor

    public CustomerService(CustomerSettings customerSettings,
        IEventPublisher eventPublisher,
        IGenericAttributeService genericAttributeService,
        INopDataProvider dataProvider,
        IRepository<Customer> customerRepository,
        IRepository<CustomerRole> customerRoleRepository,
        IShortTermCacheManager shortTermCacheManager,
        IStaticCacheManager staticCacheManager,
        IWorkContext workContext)
    {
        _customerSettings = customerSettings;
        _eventPublisher = eventPublisher;
        _genericAttributeService = genericAttributeService;
        _dataProvider = dataProvider;
        _customerRepository = customerRepository;
        _customerRoleRepository = customerRoleRepository;
        _shortTermCacheManager = shortTermCacheManager;
        _staticCacheManager = staticCacheManager;
        _workContext = workContext;
    }

    #endregion
}
```

**Constructor Injection Benefits:**
1. **Immutability**: Dependencies are set once and cannot be changed
2. **Testability**: Easy to mock dependencies for unit testing
3. **Clear Dependencies**: All dependencies are visible in the constructor
4. **Compile-time Safety**: Missing dependencies cause compile errors

---

## 2.5 Advanced Service Patterns

### Service Composition Pattern

Let's examine how services compose with each other:

```csharp
public async Task<CustomerRegistrationResult> RegisterCustomerAsync(CustomerRegistrationRequest request)
{
    // Validate input
    if (request == null)
        throw new ArgumentNullException(nameof(request));

    // Check if customer already exists
    var existingCustomer = await GetCustomerByEmailAsync(request.Email);
    if (existingCustomer != null)
        return CustomerRegistrationResult.CustomerAlreadyExists;

    // Create new customer
    var customer = new Customer
    {
        CustomerGuid = Guid.NewGuid(),
        Email = request.Email,
        Username = request.Username,
        Active = true,
        CreatedOnUtc = DateTime.UtcNow
    };

    // Save customer
    await _customerRepository.InsertAsync(customer);

    // Set default customer role
    var defaultRole = await _customerRoleRepository.GetByIdAsync(_customerSettings.DefaultCustomerRoleId);
    if (defaultRole != null)
        await AddCustomerRoleMappingAsync(customer.Id, defaultRole.Id);

    // Publish event
    await _eventPublisher.PublishAsync(new CustomerRegisteredEvent(customer));

    return CustomerRegistrationResult.Successful;
}
```

**Service Composition Patterns:**
1. **Validation**: Input validation at the service boundary
2. **Business Logic**: Complex business rules encapsulated
3. **Data Persistence**: Repository pattern for data access
4. **Event Publishing**: Domain events for loose coupling
5. **Result Objects**: Specific return types for different outcomes

---

## 2.6 The Repository Pattern Implementation

### Generic Repository Interface

Navigate to `/workspace/src/Libraries/Nop.Data/IRepository.cs`:

```csharp
public partial interface IRepository<T> where T : BaseEntity
{
    Task<T> GetByIdAsync(int? id, Func<IQueryable<T>, IQueryable<T>> func = null);
    Task<IList<T>> GetByIdsAsync(IList<int> ids, Func<IQueryable<T>, IQueryable<T>> func = null);
    Task<IList<T>> GetAllAsync(Func<IQueryable<T>, IQueryable<T>> func = null);
    Task<IPagedList<T>> GetAllPagedAsync(Func<IQueryable<T>, IQueryable<T>> func = null,
        int pageIndex = 0, int pageSize = int.MaxValue, bool getOnlyTotalCount = false);
    Task<int> InsertAsync(T entity);
    Task InsertAsync(IList<T> entities);
    Task<int> UpdateAsync(T entity);
    Task UpdateAsync(IList<T> entities);
    Task<int> DeleteAsync(T entity);
    Task DeleteAsync(IList<T> entities);
}
```

**Generic Repository Benefits:**
1. **Type Safety**: Compile-time type checking
2. **Consistency**: Same interface for all entities
3. **Flexibility**: Func parameters allow custom querying
4. **Async Support**: All operations are asynchronous

---

## 2.7 Service Lifetime Management

### Service Registration with Lifetimes

```csharp
// Singleton services
services.AddSingleton<ICacheKeyManager, CacheKeyManager>();
services.AddSingleton<IEventPublisher, EventPublisher>();

// Scoped services
services.AddScoped<ICustomerService, CustomerService>();
services.AddScoped<IWorkContext, WorkContext>();

// Transient services
services.AddTransient<ILogger, Logger>();
```

**Lifetime Considerations:**
1. **Singleton**: Stateless services, configuration, cache managers
2. **Scoped**: Per-request services, repositories, business services
3. **Transient**: Lightweight services, utilities, factories

---

## 2.8 Business Reasoning: Why These Patterns?

### Scalability Considerations

**Why service layer pattern?**
- **Separation of Concerns**: Business logic separated from presentation
- **Reusability**: Services can be used by multiple controllers
- **Testability**: Business logic can be tested independently
- **Maintainability**: Changes to business logic don't affect presentation

### Enterprise Requirements

**Why dependency injection?**
- **Loose Coupling**: Services depend on interfaces, not implementations
- **Testability**: Easy to mock dependencies for testing
- **Flexibility**: Can swap implementations without changing code
- **Configuration**: Services can be configured differently per environment

---

## 2.9 Practical Exercise: Service Design

### Exercise 1: Create a Service Interface
Create an interface for a product service with these methods:
- `GetProductByIdAsync(int id)`
- `GetProductsByCategoryAsync(int categoryId, int pageIndex, int pageSize)`
- `CreateProductAsync(Product product)`
- `UpdateProductAsync(Product product)`
- `DeleteProductAsync(int id)`

### Exercise 2: Implement the Service
Implement the service with:
- Constructor injection
- Repository pattern
- Caching
- Event publishing
- Error handling

### Exercise 3: Register the Service
Register your service in the DI container with appropriate lifetime.

---

## 2.10 Common Pitfalls and Best Practices

### ❌ **Common Mistakes**

1. **Service Locator Anti-pattern**: Using `EngineContext.Current.Resolve<T>()`
2. **Circular Dependencies**: Services depending on each other
3. **Wrong Lifetime**: Using wrong service lifetime
4. **Missing Interfaces**: Depending on concrete classes instead of interfaces

### ✅ **Best Practices**

1. **Interface Segregation**: Small, focused interfaces
2. **Constructor Injection**: Inject all dependencies via constructor
3. **Async/Await**: Use async methods consistently
4. **Error Handling**: Implement proper exception handling
5. **Event Publishing**: Use events for loose coupling

---

## 2.11 Summary and Next Steps

### Key Takeaways

1. **Dependency Injection**: Loose coupling through interfaces
2. **Service Layer**: Business logic encapsulated in services
3. **Repository Pattern**: Consistent data access
4. **Service Lifetimes**: Appropriate lifetime for each service type
5. **Event Publishing**: Loose coupling through domain events

### What's Next?

In the next chapter, we'll explore **Caching Strategies and Performance**, learning how nopCommerce implements multi-level caching for enterprise performance.

---

## Further Reading

- [Dependency Injection in .NET](https://docs.microsoft.com/en-us/dotnet/core/extensions/dependency-injection)
- [Service Layer Pattern](https://martinfowler.com/eaaCatalog/serviceLayer.html)
- [Repository Pattern](https://martinfowler.com/eaaCatalog/repository.html)

---

*Ready to optimize performance? Let's explore caching strategies in Chapter 3!*