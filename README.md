# WebAPI Performance Fix: Cache Dictionary Thundering Herd

**Date:** 2025-12-16
**Issue:** WebAPI Lookups endpoint failing under 100 concurrent requests
**Root Cause:** Missing thread synchronization in cache dictionary classes

---

## Problem Description

The WebAPI `/breezeapi/AgentApp/Lookups` endpoint was failing under concurrent load:
- 100 concurrent requests caused timeouts and OutOfMemoryExceptions
- Direct SQL queries to the same database handled 100 concurrent requests fine
- After 1 manual request "warmed up" the cache, subsequent 100 concurrent requests worked fine

This pointed to an initialization/caching issue rather than a database capacity problem.

---

## Root Cause Analysis

The caching layer in `Merlin.DAL.Caching.CacheDictionary` had a **thundering herd bug**.

All cache dictionary classes followed this pattern (example from `CacheDictionary.cs`):

```csharp
public IEnumerable<object> GetAll()
{
    // NO LOCK - BUG!
    if (!_cache.Any())
    {
        RepopulateCache();  // Hits database
    }
    return _cache.Select(x => x.Value);
}
```

### What Happened with 100 Concurrent Requests on Cold Start:

1. All 100 threads call `GetAll()` simultaneously
2. All 100 threads check `!_cache.Any()` → all see `true` (cache is empty)
3. **All 100 threads call `RepopulateCache()` simultaneously**
4. Each `RepopulateCache()` call:
   - Creates a NEW `MemoryCache` instance
   - Executes database queries
   - Populates the cache
5. Result:
   - 100 database round trips instead of 1
   - 100 MemoryCache instances created (massive memory allocation)
   - Cache instances overwriting each other while threads are reading/writing
   - OutOfMemoryException and thread starvation

### Why Direct SQL Worked Fine:

Direct SQL bypasses the caching layer entirely - each request just runs a query and returns. No shared state, no race conditions.

### Why 1 Request "Warmed Up" Everything:

After one request completes, the cache is populated. Subsequent requests see `!_cache.Any()` as `false` and skip the database call, returning cached data immediately.

---

## The Fix

Added `lock` statements around the check-and-populate pattern in all cache dictionary classes to ensure only ONE thread populates the cache while others wait.

### Fixed Pattern:

```csharp
public IEnumerable<object> GetAll()
{
    lock (lockingObject)  // FIX: Only one thread populates cache
    {
        if (!_cache.Any())
        {
            RepopulateCache();
        }
        return _cache.Select(x => x.Value).ToList();
    }
}
```

### Files Modified:

| File | Method Fixed |
|------|--------------|
| `Merlin.DAL\Caching\CacheDictionary\CacheDictionary.cs` | `GetAll()` |
| `Merlin.DAL\Caching\CacheDictionary\LoVCacheDictionary.cs` | `GetAll()` |
| `Merlin.DAL\Caching\CacheDictionary\SimpleLoVCacheDictionary.cs` | `Get()` |
| `Merlin.DAL\Caching\CacheDictionary\SetupLoVCacheDictionary.cs` | `Get()` |
| `Merlin.DAL\Caching\CacheDictionary\MemberLoVCacheDictionary.cs` | `Get()` |
| `Merlin.DAL\Caching\CacheDictionary\UserCacheDictionary.cs` | `Get()` |
| `Merlin.DAL\Caching\CacheDictionary\UserLoVCacheDictionary.cs` | `Get()` |
| `Merlin.DAL\Caching\CacheDictionary\ObjectCacheDictionary.cs` | `Get()` |

### Changes Made to Each File:

1. Added static locking object:
   ```csharp
   private static readonly object lockingObject = new object();
   ```

2. Wrapped cache check-and-populate logic in `lock` block

---

## Result After Fix

With the locking fix in place:
- First request: Thread 1 acquires lock, populates cache, releases lock
- Other 99 threads: Wait for lock, then use already-populated cache
- Only 1 database round trip instead of 100
- No memory pressure from multiple cache instances
- All 100 requests complete successfully

---

## Technical Notes

### Why `lock` is Appropriate Here:

1. Cache population is a relatively fast operation (milliseconds)
2. Contention only occurs on cold start - once cache is populated, lock is acquired/released quickly
3. The alternative (100 simultaneous DB calls + memory allocation) is far worse than brief lock contention

### Thread Safety Considerations:

- The `lockingObject` is `static` so all instances of the same dictionary type share one lock
- This is intentional - we want ONE population across ALL requests, not one per instance
- `MemoryCache` itself is thread-safe for reads, so once populated, concurrent reads are fine

### What Was NOT the Problem:

- Database capacity (direct SQL worked fine)
- Thread pool starvation (though this was initially suspected)
- Entity Framework query compilation (this was a secondary issue)
- IIS configuration

---

## Lessons Learned

1. **Always protect shared mutable state** - the "check then act" pattern is a classic race condition
2. **Direct comparison testing is valuable** - testing direct SQL vs WebAPI isolated the problem to the application layer
3. **Symptoms can mislead** - OutOfMemoryException suggested memory issues, but root cause was a threading bug
4. **Warmup behavior is a clue** - when "one request fixes everything," look for initialization race conditions
