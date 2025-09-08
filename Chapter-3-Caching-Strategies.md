# Chapter 3: Caching Strategies and Performance

## Learning Objectives
By the end of this chapter, you will understand:
- How nopCommerce implements multi-level caching strategies
- The difference between static and short-term caching
- How to design cache-friendly applications
- Performance optimization techniques for enterprise applications

---

## 3.1 Starting with Real Code: The Caching Infrastructure

Let's examine nopCommerce's sophisticated caching system.

### Cache Manager Interfaces

Navigate to `/workspace/src/Libraries/Nop.Core/Caching/IStaticCacheManager.cs`:

```csharp
public partial interface IStaticCacheManager : IDisposable, ICacheKeyService
{
    Task<T> GetAsync<T>(CacheKey key, Func<Task<T>> acquire);
    Task<T> GetAsync<T>(CacheKey key, Func<T> acquire);
    Task<T> GetAsync<T>(CacheKey key, T defaultValue = default);
    Task<object> GetAsync(CacheKey key);
    Task RemoveAsync(CacheKey cacheKey, params object[] cacheKeyParameters);
    Task SetAsync<T>(CacheKey key, T data);
    Task RemoveByPrefixAsync(string prefix, params object[] prefixParameters);
    Task ClearAsync();
}
```

**Key Observations:**
1. **Generic Methods**: Type-safe caching operations
2. **Async Support**: All operations are asynchronous
3. **Cache Key Management**: Centralized cache key handling
4. **Bulk Operations**: Remove by prefix, clear all
5. **Lazy Loading**: Functions to load data if not cached

---

## 3.2 Multi-Level Caching Architecture

nopCommerce implements a sophisticated multi-level caching strategy:

### Cache Levels

1. **Static Cache** (`IStaticCacheManager`): Long-term caching between HTTP requests
2. **Short-term Cache** (`IShortTermCacheManager`): Per-request caching
3. **Distributed Cache**: Redis or SQL Server for multi-instance scenarios

### Cache Implementation Hierarchy

```
┌─────────────────────────────────────┐
│        Application Layer            │
│     (Controllers, Services)         │
└─────────────────┬───────────────────┘
                  │
┌─────────────────▼───────────────────┐
│         Short-term Cache            │
│      (Per-request caching)          │
└─────────────────┬───────────────────┘
                  │
┌─────────────────▼───────────────────┐
│         Static Cache                │
│    (Long-term caching)              │
└─────────────────┬───────────────────┘
                  │
┌─────────────────▼───────────────────┐
│       Distributed Cache             │
│    (Redis/SQL Server)               │
└─────────────────────────────────────┘
```

---

## 3.3 Static Cache Implementation

### Redis Cache Manager

Navigate to `/workspace/src/Libraries/Nop.Services/Caching/RedisCacheManager.cs`:

```csharp
public partial class RedisCacheManager : DistributedCacheManager
{
    protected readonly IRedisConnectionWrapper _connectionWrapper;

    public RedisCacheManager(AppSettings appSettings,
        IDistributedCache distributedCache,
        IRedisConnectionWrapper connectionWrapper,
        ICacheKeyManager cacheKeyManager,
        IConcurrentCollection<object> concurrentCollection)
        : base(appSettings, distributedCache, cacheKeyManager, concurrentCollection)
    {
        _connectionWrapper = connectionWrapper;
    }

    protected virtual async Task<IEnumerable<RedisKey>> GetKeysAsync(EndPoint endPoint, string prefix = null)
    {
        return await (await _connectionWrapper.GetServerAsync(endPoint))
            .KeysAsync((await _connectionWrapper.GetDatabaseAsync()).Database, 
                string.IsNullOrEmpty(prefix) ? null : $"{prefix}*")
            .ToListAsync();
    }
}
```

**Redis Implementation Benefits:**
1. **Distributed**: Works across multiple application instances
2. **Performance**: In-memory data store with persistence
3. **Scalability**: Can handle large amounts of data
4. **Persistence**: Data survives application restarts

---

## 3.4 Cache Key Management

### Cache Key Structure

Navigate to `/workspace/src/Libraries/Nop.Core/Caching/CacheKey.cs`:

```csharp
public partial class CacheKey
{
    public CacheKey(string key, params object[] keyObjects)
    {
        Key = key;
        KeyObjects = keyObjects ?? Array.Empty<object>();
        CacheTime = TimeSpan.FromMinutes(60); // Default cache time
    }

    public string Key { get; }
    public object[] KeyObjects { get; }
    public TimeSpan CacheTime { get; set; }
    public CacheItemPriority Priority { get; set; } = CacheItemPriority.Normal;
}
```

**Cache Key Design:**
1. **Hierarchical Keys**: Structured key naming
2. **Parameter Support**: Keys can include parameters
3. **Configurable TTL**: Different cache times per key
4. **Priority Support**: Cache eviction priority

### Cache Key Generation

```csharp
public static partial class NopCustomerDefaults
{
    public static CacheKey CustomerByIdCacheKey => new("Nop.customer.id-{0}", CustomerByIdPrefix);
    public static CacheKey CustomerByGuidCacheKey => new("Nop.customer.guid-{0}", CustomerByGuidPrefix);
    public static CacheKey CustomerByEmailCacheKey => new("Nop.customer.email-{0}", CustomerByEmailPrefix);
    public static CacheKey CustomerByUsernameCacheKey => new("Nop.customer.username-{0}", CustomerByUsernamePrefix);
    
    public static string CustomerByIdPrefix => "Nop.customer.id-";
    public static string CustomerByGuidPrefix => "Nop.customer.guid-";
    public static string CustomerByEmailPrefix => "Nop.customer.email-";
    public static string CustomerByUsernamePrefix => "Nop.customer.username-";
}
```

**Key Naming Conventions:**
1. **Namespace Prefix**: `Nop.customer.` for customer-related keys
2. **Operation Suffix**: `.id-`, `.email-` for different operations
3. **Parameter Placeholders**: `{0}` for parameter substitution
4. **Prefix Constants**: Reusable prefixes for bulk operations

---

## 3.5 Caching in Service Layer

### Service-Level Caching Implementation

Let's examine how caching is implemented in the CustomerService:

```csharp
public async Task<Customer> GetCustomerByIdAsync(int customerId)
{
    if (customerId == 0)
        return null;

    var cacheKey = _staticCacheManager.PrepareKeyForDefaultCache(NopCustomerDefaults.CustomerByIdCacheKey, customerId);
    
    return await _staticCacheManager.GetAsync(cacheKey, async () =>
    {
        return await _customerRepository.GetByIdAsync(customerId);
    });
}

public async Task<Customer> GetCustomerByEmailAsync(string email)
{
    if (string.IsNullOrWhiteSpace(email))
        return null;

    var cacheKey = _staticCacheManager.PrepareKeyForDefaultCache(NopCustomerDefaults.CustomerByEmailCacheKey, email);
    
    return await _staticCacheManager.GetAsync(cacheKey, async () =>
    {
        return await _customerRepository.Table
            .FirstOrDefaultAsync(c => c.Email == email);
    });
}
```

**Caching Pattern:**
1. **Cache Key Preparation**: Generate cache key with parameters
2. **Lazy Loading**: Load data only if not in cache
3. **Repository Access**: Use repository for data access
4. **Async Operations**: All operations are asynchronous

---

## 3.6 Cache Invalidation Strategies

### Event-Based Cache Invalidation

```csharp
public async Task<int> UpdateCustomerAsync(Customer customer)
{
    if (customer == null)
        throw new ArgumentNullException(nameof(customer));

    await _customerRepository.UpdateAsync(customer);

    // Clear cache
    await _staticCacheManager.RemoveByPrefixAsync(NopCustomerDefaults.CustomerByIdPrefix);
    await _staticCacheManager.RemoveByPrefixAsync(NopCustomerDefaults.CustomerByEmailPrefix);
    await _staticCacheManager.RemoveByPrefixAsync(NopCustomerDefaults.CustomerByUsernamePrefix);

    // Publish event
    await _eventPublisher.PublishAsync(new CustomerUpdatedEvent(customer));

    return customer.Id;
}
```

**Cache Invalidation Strategy:**
1. **Prefix-based Removal**: Remove all keys with specific prefix
2. **Event Publishing**: Notify other services of changes
3. **Immediate Invalidation**: Clear cache immediately after update
4. **Multiple Key Types**: Clear different types of customer keys

---

## 3.7 Short-term Caching

### Per-Request Caching

```csharp
public async Task<Customer> GetCurrentCustomerAsync()
{
    // Check short-term cache first
    var customerId = _shortTermCacheManager.Get<int?>(NopCustomerDefaults.CustomerIdAttributeName);
    if (customerId.HasValue)
    {
        return await GetCustomerByIdAsync(customerId.Value);
    }

    // Load from authentication
    var customer = await _authenticationService.GetAuthenticatedCustomerAsync();
    if (customer != null)
    {
        _shortTermCacheManager.Set(NopCustomerDefaults.CustomerIdAttributeName, customer.Id);
    }

    return customer;
}
```

**Short-term Cache Benefits:**
1. **Per-Request Scope**: Data cached for single request
2. **Performance**: Avoid repeated database calls
3. **Memory Efficient**: Automatically cleared after request
4. **Fallback Strategy**: Falls back to static cache

---

## 3.8 Business Reasoning: Why This Caching Strategy?

### Performance Requirements

**Why multi-level caching?**
- **Response Time**: Sub-millisecond response times for cached data
- **Database Load**: Reduce database queries by 80-90%
- **Scalability**: Handle thousands of concurrent users
- **Cost Efficiency**: Reduce database costs

### Enterprise Considerations

**Why Redis for distributed cache?**
- **Multi-Instance**: Multiple application instances share cache
- **High Availability**: Redis clustering for fault tolerance
- **Memory Efficiency**: Optimized memory usage
- **Persistence**: Data survives application restarts

---

## 3.9 Practical Exercise: Implement Caching

### Exercise 1: Create a Cache Key
Create cache keys for a product service:
- Product by ID
- Products by category
- Product search results
- Product recommendations

### Exercise 2: Implement Service Caching
Add caching to your product service:
- Cache product details
- Cache product lists
- Implement cache invalidation
- Add cache statistics

### Exercise 3: Cache Configuration
Configure different cache times for:
- Product details (1 hour)
- Product lists (30 minutes)
- Search results (15 minutes)
- Recommendations (2 hours)

---

## 3.10 Common Pitfalls and Best Practices

### ❌ **Common Mistakes**

1. **Cache Stampede**: Multiple requests loading same data
2. **Stale Data**: Not invalidating cache after updates
3. **Memory Leaks**: Not disposing cache managers
4. **Wrong Cache Level**: Using wrong cache for data type

### ✅ **Best Practices**

1. **Cache Key Design**: Consistent, hierarchical key naming
2. **TTL Strategy**: Appropriate cache times for different data
3. **Cache Invalidation**: Immediate invalidation after updates
4. **Monitoring**: Track cache hit rates and performance
5. **Fallback Strategy**: Graceful degradation when cache fails

---

## 3.11 Summary and Next Steps

### Key Takeaways

1. **Multi-Level Caching**: Different cache levels for different needs
2. **Cache Key Management**: Structured, hierarchical key naming
3. **Lazy Loading**: Load data only when needed
4. **Cache Invalidation**: Immediate invalidation after updates
5. **Performance Monitoring**: Track cache effectiveness

### What's Next?

In the next chapter, we'll explore **Middleware and Request Pipeline**, learning how nopCommerce handles HTTP requests and implements cross-cutting concerns.

---

## Further Reading

- [Caching in .NET](https://docs.microsoft.com/en-us/aspnet/core/performance/caching/)
- [Redis Caching](https://redis.io/docs/manual/caching/)
- [Cache-Aside Pattern](https://docs.microsoft.com/en-us/azure/architecture/patterns/cache-aside)

---

*Ready to handle HTTP requests? Let's explore middleware patterns in Chapter 4!*