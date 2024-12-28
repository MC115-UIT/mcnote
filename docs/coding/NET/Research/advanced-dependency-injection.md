---
tags:
    - Research
    - .NET
---

## Advanced Dependency Injection Patterns and Service Lifetime Management in .NET
REF : [From medium](https://freedium.cfd/https://medium.com/@riturajpokhriyal/advanced-dependency-injection-patterns-and-service-lifetime-management-in-net-d1ae3e45ac1d)


### Understanding Service Lifetime Scoping

### The Hidden Dangers of Singleton Services

One of the most common pitfalls is injecting scoped or transitent services into singletons. Let's look at why this is problematic


```C#
    public class SingletonService
    {
        private readonly ScopedService _scopedService;


        /// Anti-pattern : Injecting scoped service into singleton
        public SingletonService(ScopedService scopedService)
        {
            _scopedService = scopedService;
        }
    }
```

To detect these issues early, we can create a validation helper function:

```C#
    public static class ServiceCollectionValidation
    {
        public static void ValidateScopes(this IServiceCollection services)
        {
            var singletonServices = services
                .Where (s => s.Lifetime == ServiceLifeTime.Singleton)
                .ToList();

            var scopedServices = services
                .Where(s => s.Lifetime == ServiceLifeTime.Scoped)
                .Select(s => s.ServiceType)
                .ToList();

            singletonServiecs.Select(singleton => ValidateConstructorInjection(singleton.ImplementationType, scopedServices)).ToList();
        }
        
        private static void ValidateConstructorInjection(Type type, List<Type> scopedServices)
        {
            type.GetConstructors()
                .SelectMany(constructor => constructor.GetParameters())
                .Where(parameter => parameters.Contains(parameter.ParameterType))
                .Select(parameter => throw new InvalidationException (
                    $"Type {type.Name} is registered as singleton but depends on scoped service {parameter.ParameterType.Name}"
                ))
                .ToList();
        }
    }
```

### Implementing Factory Pattern for Complex Lifetime Management

When you need more control over object creation and lifetime:

```C#
    public interface IServiceFactory<T>
    {
        T Create();
    }

    public class ServiceFactory<T> : IServiceFactory<T>
    {
        private readonly IServiceProvider _serviceProvider;
        
        public ServiceFactory(IServiceProvider serviceProvider)
        {
            _serviceProvider = serviceProvider;
        }

        public T Create()
        {
            return ActivatorUtilities.CreateInstance<T>(_serviceProvider);
        }
    }

    // Registration
    services.AddSingleton(typeof(IServiceFactory<>), typeof(ServiceFactory<>));


    // Usage
    public class ComplexService
    {
        private readonly IServiceFactory<ScopedDependency> _factory;

        public ComplexService( IServiceFactory<ScopedDependency> factory)
        {
            _factory = factory;
        }


        public void DoWork()
        {
            using var dependency = _factory.Create();
            // Work with the dependency
        }
    }
```

### Advanced Registration Patterns

### Decorator Pattern Implementation


Implementing with decorators with DI for cross-cutting concerns;

```C#
    public static class ServiceCollectionExtentions
    {
        public static IServiceCollection Decorate<TService, TDecorator>(
            this IServiceCollection services
        ) where TDecorator : TService
        {
            var wrappedDescriptor = services.FirstOrDefault( s => s.ServiceType == typeof(TService));

            if(wrappedDescriptor == null)
            {
                throw new InvalidOperationException($"{typeof(TService).Name} is not registered");
            }

            var objectFactory = ActivatorUtilities.CreateFactory(
                typeof(TDecorator),
                new[] {typeof(TService)}
            );

            services.Replace(ServiceDescriptor.Describe(
                typeof(TService),
                sp => (TService)objectFactory(sp, new[] {sp.CreateInstance(wrappedDescriptor)}),
                wrappedDescriptor.Lifetime
            ));
            return services;
        }

        private static object CreateIntance(
            this IServiceProvider services,
            ServiceDescriptor descriptor
        )
        {
            if(descriptor.ImplementationInstance != null)
            {
                return descriptor.ImplementationInstance;
            }
            if(descriptor.ImplementationFactory != null)
            {
                return descriptor.ImplementationFactory(services);
            }

            return ActivatorUtilities.GetServiceOrCreateInstance(services, descriptor.ImplementationType);
        }
    }
```

### Condition Registration

Implementing environment-specific service registration

```C#
    public static class ConditionalRegistration
    {
        public static IServiceCollection AddServiceIf<TService, TImplementation>(
            this IServiceCollection services,
            Func<IServiceProvider, bool> condition,
            ServiceLifetime lifetime = ServiceLifetime.Scoped
        ) where IImplementation : class, TService where TService : class
        {
            var descriptor = new ServiceDescriptor(
                typeof(TService),
                sp => condition(sp) ? ActivatorUtilities.CreateInstance<TImplementation>(sp) : null,
                lifetime
            );


            services.Add(descriptor);
            return services;
        }
    }

    // Usage

    services.AddServiceIf<IEmailService, SmtpEmailService>(
        sp => env.IsDevelopment(),
        ServiceLifetim.Singleton
    )
```

### Advanced Scoping Scenarios

### Custom Scoped Mangement

Creating custom scopes for background operations:

```C#
    public class BackgroundJobScope : IDisposable
    {
        private readonly IServiceScope _scope
        private readonly CancellationTokenSource _cts;
        

        public BackgroundJobScope(IServiceProvider serviceProvider)
        {
            _scoped = serviceProvider.CreateScope();
            _cts = new CancellationTokenSource();
        }


        public IServiceProvider ServiceProvider => _scope.ServiceProvider;
        public CancellationToken CancellationToken => _cts.Token;


        public void Dispose()
        {
            _cts.Cancel();
            _cts.Dispose();
            _scoped.Dispose();
        }
    }


    // Usage in a background service

    public class BackgroundJob : BackgroundService
    {
        private readonly IServiceProvider _serviceProvider;

        protected override async Task ExecuteAsync(CancellationToken stoppingToken)
        {
            while(!stoppingToken.IsCancellationRequested)
            {
                using var jobScope = new BackgroundJobScope(_serviceProvider);
                var workder = jobScope.ServiceProvider.GetRequiredService<IWorkder>();
                await worker.DoWorkAsync(jobScope.CancellationToken);
            }
        }
    } 
```

### Mapping Disposable Service

### Implementation Custom Disposal Patterns
For services that need special cleanup:

```C#
    public interface IAsyncDisposableService : IAsyncDisposable
    {
        Task InitializeAsync();
    }

    public class ServiceWithResourceManagement : IAsyncDisposable
    {
        private bool _initialized;
        private DbConnection _connection;

        public async Task InitializeAsync()
        {
            if(_initialized) return;

            _connection = new SqlConnection("connection-string");
            await _connection.OpenAsync();

            _initialized = true; 
        }

        public async ValueTask DisposeAsync()
        {
            if(_connection != null)
            {
                await _connection.DisposeAsync();
            }
        }
    }


    /// Registration helper

    public static class ServiceCollectionExtensions
    {
        public static IServiceCollection AddAsyncDisposable<TService, TImplementation>(
            this IServiceCollection services
        ) where TService: class where TImplementation : class, TService, IAsyncDisposableService 
        {
            services.AddScoped<TService>( sp => 
            {
                var service = ActivatorUtilities.CreateInstance<TImplementation>(sp);
                service.InitializeAsync().GetAwaiter().GetResult();
                return service;
            });

            return services;
        }
    }

```

### Best Practices and Recommendations

1. Service Lifetime Documentation

```C#
    [ServiceLifetime(ServiceLifetim.Scoped)]
    public interface IDocumentService
    {

    }

    public ServiceLifetimeAttribute : Attribute
    {
        private ServiceLifetime Lifetime {get;}
        
        public ServiceLifetimeAttribute(ServiceLifetime lifetime)
        {
            Lifetime = lifetime;
        }
    }
```
2. Dependency Validation at Startup

```C#
    public static class StartupExtensions
    {
        public static void ValidateService(this IServiceCollection services)
        {
            var provider = services.BuildServiceProvider();

            foreach(var service in services)
            {
                try
                {
                    provider.GetService(service.ServiceType);
                }
                catch (Exception ex)
                {
                    throw new Exception($"Error resolving {service.ServiceType.Name} : {ex.Message}")
                }
            }
        }
    }
```

### Common Pitfalls to Avoid

1. Capturing Scoped Services in Callbacks

```C#
    // 🚫 Anti-pattern
    services.AddSingleton<IHostedService>(sp =>
    {
        var scopedService = sp.GetRequiredService<IScopedService>();
        return new MyHostedService(scopedService); // Wrong!
    });

    // ✅ Correct pattern
    services.AddSingleton<IHostedService>(sp =>
    {
        var factory = sp.GetRequiredService<IServiceScopeFactory>();
        return new MyHostedService(factory);
    });
```
2. Service Locator Anti-pattern

```C#
    // 🚫 Anti-pattern
    public class ServiceLocator
    {
        public static IServiceProvider Provider { get; set; }
    }

    // ✅ Correct pattern: Use constructor injection
```

### Conclusion

Proper understanding and implementation of dependency injection patterns is crucial for building maintainable and scalable .NET applications. By following these advanced patterns and best practices, you can avoid common pitfalls and create more robust applications.

Remember to:

    - Always validate service lifetimes at startup
    - Use factory patterns for complex lifetime management
    - Implement proper disposal patterns for resources
    - Avoid service locator pattern
    - Document service lifetimes
The key to successful DI implementation is understanding not just how to use it, but also the implications of different lifetime scopes and their impact on your application's behaviour and performance.