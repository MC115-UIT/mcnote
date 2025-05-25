**.NET 9** is just around the corner, bringing a wealth of changes and improvements. This article will walk you through the most impactful and broadly applicable features in .NET 9 and C# 13.

## 1. New Lock Object

C# 13 introduce a new `System.Threading.Lock` type for mutual exclustion. Historically, an `object` type was used for locking purposes, but now there's a dedicated `Lock` type, which is expected to become the future standard for most locking.

```c#
    // before
    public class LockExmaple
    {
        private readonly object _lock = new();
        
        public void DoStuff()
        {
            lock(_lock)
            {
                Console.WriteLine("We are inside the old lock");
            }
        }
    }

    // .NET 9
    public class LockExample
    {
        private readonly Lock _lock = new();

        public void DoStuff()
        {
            lock(_lock)
            {
                Console.WriteLine("We are in side .NET 9 lock");
            }
        }
    }
```

+ **Cleaner and Safer Code** : It make the code more readable and predictable. Additionally, the compiler wiil warn you if a `Lock` instances is incorrectly used as a regular `object`.
+ **Performance** : According to Microsoft, it can be more effecient then locking on an arbitrary `object` instance.
+ **New Locking Mechanism** : `EnterScope` replaces the `Monitor` class under the hood. Find the IL code of the example above [here](https://gist.github.com/dkorolov1/307588775724e2a920f995998785fb70) .It returns a `ref struct` that follows the `Dispose` pattern, allowing seamless use with the `using` statement.
+ **Asynchronous Limitations** : async calls are still not allowed within lock block due to the inherent limitations in how locks and asynchronous code interact. The traditional `SemaphoreSlim` approach remains the go-to alternative.

```c#
    public class LockExample
    {
        private readonly Lock _lock = new();
        private readonly SemaphoreSlim _semaphore = new(1,1);

        public async Task DoStuff(int val)
        {
            lock(_lock)
            {
                await Task.Delay(1000); // Compile error : can not `await` in the body of a `lock` statement
            }
            
            using(_lock.EnterScope())
            {
                await Task.Delay(1000); // Runtime error : Instance of type `System.Threading.Lock.Scope`
            }

            await _semaphore.WaitAsync();
            try
            {
                await Task.Delay(1000); // NO errors
            }
            finally
            {
                _semaphore.Release();
            }
        }
    }
```

## 2. Task.WhenEach()

Imagine you have a list of tasks that finish at different intervals. You want to process each as soon as it completes. `WaitAll()` will not work here since it waits for all tasks to finish. We can use the `WaitAny()` as wordaround to pick off the next one as it completes. C# 13 introduce a more elegant and efficient approach to handle that scenario : `Task.WhenEach`. Check out the example below.

```c#
    // List of 5 tasks that finish at random intervals

    var tasks = Enumerable.Range(1,5)
        .Select(asycn t => {
            await Task.Delay(new Random().Next(1000,5000));
            return $"Task {t} done";
        })
        .ToList();

    // before
    while(tasks.Count > 0)
    {
        var completedTask = await Task.WhenAny(tasks);
        tasks.Remove(completedTask);
        Console.WriteLine(await completedTask)
    }

    // in .NET 9
    await foreach(var completedTask in Task.WhenEach(tasks))
    {
        Console.WriteLine(await completedTask);
    }
```

`Task.WhenEach` return `IAsyncEnumerable<Task<TResult>>` and allow using `await foreach` to easily iterate through the tasks as they are completed.

## 3. params Collections

Starting with C# 13, `params` parameters can be of any types supported for collection expressions.


```c#
    // before
    static void WriteNumbersCount(params int[] numbers) => Console.WriteLine(numbers.Length);

    // NET 9
    static void WriteNumbersCount(params ReadOnlySpan<int> numbers) => Console.WriteLine(number.Length);

    static void WriteNumbersCount(params IEnumerable<int> numbers) => Console.WriteLine(number.Count);
    static void WriteNumbersCount(params HashSet<int> numbers) => Console.WriteLine(number.Count);

```

+ **Cleaner Codee** : Significantly reduces the number of calls to `.ToArray()`, `.ToList()`
+ **Performance** : Calls like `.ToArray()`, `.ToList()` already add extra resource overhead on their own. Also, it now supports passing `Span<>` and `IEnumerable<>`, leveraging more efficient memory usage and lazy execution. Overall, this provides greater flexibility and better performance in scenarios that demand it.

## 4. Semi-Auto Properties

When you declare an auto-implemented property in C# `public int Number {get; set;}`, the compiler generates a backing field (e.g.m `_number`) and internal getter/setter methods `void set_Number(int number)` and `int get_Number`. But if you need custom logic like validation, default values, computations, or lazy loading in a property's getter or setter, you usually have to manually define the backing field in your class. C# 13 simplifies this by introducing the `field` keyword, allowing you to access the backing field directly without declaring it manually.

```c#
    // before
    public class MagicNumber
    {
        private int _number;

        public int Number()
        {
            get => _number * 10;
            set {
                if( value < 0) throw new ArgumentOutOfRangeException(nameof(value), "Value must be greater than 0");
                _number = value
            }
        }
    }

    // in NET 9
    public class MagicNumber
    {
        public int Number()
        {
            get => field;
            set {
                if(value < 0 )
                    throw new ArgumentOutOfRangeException(nameof(value), "Value must be greater than 0");
                field = value;
            }
        }
    }
```

+ **Reduce Boilerplate Code** : Removes the need for manually defining private backing fields, resulting in cleaner and more concise code.
+ **Enhanced Readability** : There's no need to manage custom backing field names, making the use of the field keyword a standard that improves code clarity.
+ **Property-Scope Field** : The private backing field is confined to the property itself, preventing unintended use elsewhere in the class. This escapsulation improves safety and address a common source of errors.
+ **Potential Breaking Change** : If your class already has a property named `field` of the same type, it will take place precedence over the new `field` keyword, potentially leading to unexpected behavior. You can find [here](https://github.com/dotnet/csharplang/blob/main/proposals/field-keyword.md#breaking-changes) the C# team's proposal on how to handle this situation. This might be on of the reasons why this feature was delayed since 2016, when it was initially proposed.

## 5. Hybrid Cache

The new `HybridCache` API addresses certain gaps in the existing `IDistributedCache` and `IMemoryCache` APIs, such as the stampede problem, introduces new features and capabilities, and makes caching in .NET application more flexible and performant. `HybirdCache` is designed to be a drop-in replacement for most `IDistributedCache` and `IMemoryCache` scenarios.

```c#
    public record Post(int UserId, int Id, string Title, string Body);

    public class PostsService(
        IHttpClientFactory httpClientFactory,
        IMemoryCache memoryCache,
        IDistributedCache distributedCache,
        HybridCache hybridCache
    )
    {
        public async Task<List<Post>> GetUserPostAsync(string userId)
        {
            var cacheKey = $"post_{userId}";

            // before (Memory Cache)
            var posts = await memoryCache.GetOrCreateAsync(cacheKey, async _ => await GetPostAsync(userId));

            // before (IDistributed Cache)

            var postsJson = await distributedCache.GetStringAsync(cacheKey);
            if(postsJson is null)
            {
                posts = await GetPostsAsync(userId);
                await distributedCache.SetStringAsync(cacheKey, JsonSerializer.Serialize(posts));
            }
            else
            {
                posts = JsonSerializer.Deserialize<List<Post>>(postJson)
            }


            // in .NET 9 Hybrid Cache

            posts = await hybridCache.GetOrCreateAsync(cacheKey,
                async _ => await GetPostsAsync(userId), new HybridCacheEntryOptions(){
                    Flags = HybridCacheEntryFlags.DisableLocalCache | // Act as distributed cache
                            HybridCacheEntryFlags.DisableDistributedCache // Act as local cache
                });

            return posts;
        }
         private async Task<List<Post>?> GetPostsAsync(string userId)
        {
            Console.WriteLine("===========Fetching posts from API");
            var url = $"https://jsonplaceholder.typicode.com/posts?userId={userId}";
            var client = httpClientFactory.CreateClient();
            var response = await client.GetAsync(url);
            response.EnsureSuccessStatusCode();
            return await response.Content.ReadFromJsonAsync<List<Post>>();
        }
    }
```

+ **Best of both worlds** : The `HybridCache` gives you the flexibility to store data either in-memory (L1) or in a distributed (L2) cache, using a unified, concise API for both. This setup allows for fast, local access to frequently used data(L1) and a scalable, external cache (L2) that can handle larger, less-frequently-accessed data. `HybridCacheEntryFlags` are meant to control this behavior.
+ **Stampede Protection** :Both `IMemoryCache` and `IDistributedCache` suffer from the stampede issue. `HybridCache` solves this by ensuring only one caller regenerates the value for a give key, while others wait for the result. This helps avoid excessive cache repopulation.
+ **Extra features** : `HybridCache` offers extras like Tagging, [Configurable Serialization](https://github.com/dotnet/core/blob/main/release-notes/9.0/preview/preview4/aspnetcore.md#additional-hybridcache-features) through `.WithSerializer(...)` and `.WithSerializerFactory(...)` methods and [Reuse of Cache Instances](https://github.com/dotnet/core/blob/main/release-notes/9.0/preview/preview4/aspnetcore.md#note-on-object-reuse-with-hybridcache) using the `[ImmutableObject]` annotation.

- It might sound a bit tricky at first, but for `HybridCache` to store data out-of-process (L2), you still need to configure `IDistributedCache` (e.g, Redis), which `HybridCache` will then rely on. But even without an `IDistributedCache`, the `HybirdCache` service still provide in-process caching. For my use case above, I had this configuration.

```c#
    // Distributed Cache
    builder.Services.AddStackExchangeRedisCache(options => {
        options.Configuration = "localhost:6379";
        options.InstanceName = "SampleInstance"
    });

    builder.Services.AddMemoryCache();
    builder.Servics.AddHybridCache();

    builder.Services.AddSingleton<PostService>();
```

## 6. Built-in OpenAPI Document Generation
Starting with .NET 5, Web API templates have been shipped with built-in support for OpenApi via the [Swashbuckle.AspNetCore](https://github.com/domaindrivendev/Swashbuckle.AspNetCore) package. In .NET 9, Microsoft continues to support the OpenApi specification through its internally developed package, [Microsoft.AspNetCore.OpenApi](https://github.com/dotnet/aspnetcore/tree/main/src/OpenApi), which will replace [Swashbuckle.AspNetCore](https://github.com/domaindrivendev/Swashbuckle.AspNetCore).

```c#
    // Before
    builder.Services.AddEndpointsApiExplorer();
    builder.Services.AddSwaggerGen();

    var app = builder.Build();

    if (app.Environment.IsDevelopment())
    {
        app.UseSwagger();
        app.UseSwaggerUI();
    }

    // .NET 9
    builder.Services.AddOpenApi();
    var app = builder.Build();

    if (app.Environment.IsDevelopment())
    {
        app.MapOpenApi();
    }
```
After running your app, go to `/openapi/v1.json` to review the generated `OpenApi` document.

+ **Swagger UI** : The syntax has become shorter and appears more 'native' at first glance. However, by default,
you now only get an `OpenAPI` document without interactive API documentation. If interactive API documentation, like `SwagerUI` is essential, you'll need to integrate third-party tools like `Scalar` to achive this. Here is a detaild guide : [Scalar .NET API Reference Integration](https://github.com/scalar/scalar/blob/main/packages/scalar.aspnetcore/README.md#scalar-net-api-reference-integration)
+ **Build-Time Generation** : You can also generate OpenAPI documents at build-time using [Microsoft.Extensions.ApiDescription.Server](https://github.com/scalar/scalar/blob/main/packages/scalar.aspnetcore/README.md#scalar-net-api-reference-integration) package.

## 7. SearchValues Improvements

`SearchValues` was introduced in .NET 8, providing an immutable, read-only set of values optimized for significantly more efficient searching compared to traditional `ICollection.Contains`. Originally, it was limited to sets of characters of bytes. .NET 9 expands `SearchValues<T>` to also support strings.

```c#
    var text = "Exploring new capabilities of SearchValues".AsSpan();

    // Before

    var vowelSearch = SearchValues.Create(['n','e','w']);
    Console.WriteLine(text.ContainsAny(vowelSearch));

    // NET 9

    var keywordSearch = SearchValues.Create(["new","of"], StringComparison.OrdianlIgnoreCase);
    Console.WriteLine(text.ContainsAny(keywordSearch));
```

You can also specify the comparison type using `StringComparison` parameter.

In the future, this will definitely be the go-to option for applications that require extensive text processing, in scenarios like documents parsing, input filtration, spam detection, data redaction, search, etc.

## 8. New LINQ Methods

Three new methods were added : `CountBy`, `AggregateBy`, and `Index`. They are designed to enhance performance and conciseness in comon data manipulation task. See them in action in examples below.

### CountBy
```c#
    (string firstName, string lastName)[] people = 
    [
        ("John", "Doe"),
        ("Jane", "Peterson"),
        ("John", "Smith"),
        ("Mary", "Johnson"),
        ("Nick", "Carson"),
        ("Mary", "Morgan")  
    ];


    // before
    var firstNameCounts = people
        .GroupBy(p => p.firstName)
        .ToDictionary(group => group.Key, group => group.Count())
        .AsEnumerable()
    
    // NET 9

    var firstNameCounts = people
        .CountBy(p => p.firstName);

    foreach( var entry in firstNameCounts )
    {
        Console.WriteLine($"First name {entry.Key} appears {entry.Value} times");
    }

```

### AggregateBy

```c#
    (string name, string department, int vacationDaysLeft)[] employees =
    [
        ("John Doe", "IT", 12),
        ("Jane Peterson", "Marketing", 18),
        ("John Smith", "IT", 28),
        ("Mary Johnson", "HR", 17),
        ("Nick Carson", "Marketing", 5),
        ("Mary Morgan", "HR", 9)
    ];

    // before 

    var departmentVacationLeft = employees
        .GroupBy(e => e.department)
        .ToDictionary(group => group.Key, group => group.Sum(emp => emp.vacationDaysLeft))
        .AsEnumerable()

    // NET 9

    var departmentVacationLeft = employees
        .AggregateBy(emp => emp.department, 0 , (acc, emp) => acc + emp.vacationDaysLeft);
    foreach (var entry in departmentVacationDaysLeft)
        Console.WriteLine($"Department {entry.Key} has a total of {entry.Value} vacation days left.");
```
### Index

```c#
    var managers = new[]
    {
        "John Doe",
        "Jane Peterson",
        "John Smith"
    };

    // Before
    foreach (var (index, manager) in managers.Select((m, i) => (i, m)))
    Console.WriteLine($"Manager {index}: {manager}");

    // .NET 9
    foreach (var (index, manager) in managers.Index())
    Console.WriteLine($"Manager {index}: {manager}");
```
My personal favorite is `Index()` because the lack of an index in foreach has always been a pain in the neck and often led to clunkier workarounds.


## 9. Built-in UUID v7 Generation
Since the early days of .NET, we’ve used Guid.NewGuid() to generate UUIDs. This method produces UUID of version 4. Since then, the UUID specification has advanced significantly, and the current stable version is 7. One of the key features of version 7 is the timestamp that is part of the UUID. That's how it's structured:

```cmd
+------------------+---------------+----------------------+
| 48-bit timestamp | 12-bit random |    62-bit random     |
+------------------+---------------+----------------------+
```
The timestamp component facilitates a major benefit: you can sort UUIDs by their creation time. This makes them more suitable for use in databases and provides better guarantees of uniqueness in distributed contexts.

Now, UUID v7 generation is built-in in .NET, so you no longer need to use external libraries like UUIDNext for this purpose. A new method, Guid.CreateVersion7(), has been introduced for this. This method can also accept a timestamp, allowing you to create UUIDs that match specific times. This can be useful for testing purposes or inserting items into a specific position in sorted sequences.

```c#
    var guid = Guid.NewGuid(); // v4 UUID
    guid = Guid.CreateVersion7(); // v7 UUID
    guid = Guid.CreateVersion7(TimeProvider.System.GetUtcNow()); // v7 UUID with timestamp
```

`Guid.CreateVersion7()` uses NewGuid() under the hood. It integrates a 48-bit timestamp, and sets the correct version and variant bits to conform to UUIDv7 standards . This makes it slightly slower than NewGuid(). Keep this in mind, but unless you need to generate millions of UUIDs, the performance impact will be unnoticeable.

## 10. Other features

Below is a list of other interesting changes that are definitely worth your attention but are likely to be adopted less broadly and address more specific use cases.

+ [Implicit index access](https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-13#implicit-index-access)
+ [Partial properties](https://github.com/dotnet/core/blob/2f5ecee9ea988b4d85e288178c9d16131f3b0c43/release-notes/9.0/+ preview/preview6/csharp.md#partial-properties)
+ [Allows ref struct](https://github.com/dotnet/core/blob/main/release-notes/9.0/preview/preview6/libraries.md#allows-ref-struct-used-in-many-places-throughout-the-libraries)
+ [Base64Url](https://learn.microsoft.com/en-us/dotnet/core/whats-new/dotnet-9/libraries#base64url)
+ [Collection lookups with spans](https://github.com/dotnet/core/blob/2f5ecee9ea988b4d85e288178c9d16131f3b0c43/release-notes/9.0/preview/preview6/libraries.md#collection-lookups-with-spans)
+ [Feature switches with trimming support](https://github.com/dotnet/core/blob/2f5ecee9ea988b4d85e288178c9d16131f3b0c43/release-notes/9.0/preview/preview4/runtime.md#feature-switches-with-trimming-support)
+ [Regex.EnumerateSplits](https://github.com/dotnet/core/blob/2f5ecee9ea988b4d85e288178c9d16131f3b0c43/release-notes/9.0/preview/preview6/libraries.md#regexenumeratesplits)
+ [New Capabilities of System.Text.Json](https://github.com/dotnet/core/blob/2f5ecee9ea988b4d85e288178c9d16131f3b0c43/release-notes/9.0/preview/preview6/libraries.md#regexenumeratesplits)
+ [OrderedDictionary<TKey, TValue>](https://github.com/dotnet/core/blob/2f5ecee9ea988b4d85e288178c9d16131f3b0c43/release-notes/9.0/preview/preview6/libraries.md#ordereddictionarytkey-tvalue)
+ [ReadOnlySet<T>](https://github.com/dotnet/core/blob/d6826c747b07bb4f050e2692bc309b8acd6ec1ec/release-notes/9.0/preview/preview6/libraries.md#readonlysett)
+ [Debug.Assert now reports assert condition](https://github.com/dotnet/core/blob/main/release-notes/9.0/preview/preview6/libraries.md#allows-ref-struct-used-in-many-places-throughout-the-libraries)
+ [New Tensor<T> Type](https://github.com/dotnet/core/blob/main/release-notes/9.0/preview/preview4/libraries.md#new-tensort-type)

SRC : [here](https://medium.com/@dkorolov1/must-know-new-c-13-and-net-9-features-a42e5592e65b)
