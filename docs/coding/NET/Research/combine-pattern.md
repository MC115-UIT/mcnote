## Why Combining Design Pattern?
Think of software architecture like building a house : a hammer alone will not suffice - you need nails,
saws, and blueprints working together. Similarly, combining design pattern allows you to:

+ **Minimizing code duplication** through reusable, modular components
+ **Enabling extensibility** without rewriting existing logic
+ **Separating concerns** to improve testability mantainability
+ **Optimizing performance** by managing resources and workflows efficiently.

Let's explore powerful pattern combinations and their practical applications in the real world.

## 1. Chain of Responsibility + Result Pattern + Builder Factory : Crafting Resilient Pipelines

**Use Case** : API request processing with validation, authentication, and business logic

**Why combine these?**

+ **Chain of Responsibility** decouples sequential processing steps (e.g., authen -> validation -> business logic).
+ **Result Pattern** standardizes success/failure outcomes across handlers.
+ **Builder Factory** simplifies pipeline configuration for different workflows.

**Enhanced example** :

```c#
    // Result pattern for unified outcomes
    public record Result(bool IsSuccess, string? Error = null)
    {
        public static Result Success() => new(true);
        public static Result Failure(string error) => new(false, error);
    }

    // Chain handler interface
    public interface Handler<T>
    {
        Result Handle(T request);
    }

    // Authentication Handler
    public class AuthenticationHandler : IHandler<ApiRequest>
    {
        private readonly IHandler<ApiRequest> _next;
        public AuthenticationHandler(IHandler<ApiRequest> next) => this._next = next;

        public Result Handle(ApiRequest request)
        {
            if(!request.IsAuthenticated)
            {
                return Result.Failure("Unauthorized");
            }
            return _next?.Handle(request) ?? Result.Success();
        }
    }

    // Builder Factory for dynamic pipeline creation
    public static class PipelineBuilder
    {
        public static IHanlder<ApiRequest> CreateDefaultPipeline()
        {
            return new AuthenticationHandler(
                new ValidationHandler(
                    new BusinessLogicHandler(null)
                );
            )
        }
    }

    // Usage

    var pipeline = PipelineBuilder.CreateDefaultPipeline()
    var result = pipeline.Handle(request);

```

**Key Benifits**

+ **Testability** : Each handler can be testd in isolation
+ **Flexibility** : Swap handlers or reorder steps without disrupting the pipeline
+ **Clarity** : Results provide explicit success/failure context.

[Related docs](https://medium.com/@kohzadi90/elevate-your-c-code-implementing-chain-of-responsibility-result-and-builder-factory-patterns-8dffaedc37c5)

## 2. Factory Method + SingleTon : Resource Management Mastery

**Use Case** : Centralized access to shared resources (e.g., database connections, caches).

**Why combine these?**

+ **SingleTon ensures** a single instance of an expensive resource
+ **Factory Method** abstracts create logic, enabling lazy initialization or pooling.

**Enhanced example**:

```c#
    public interface IDatabaseConnection{}
    public class SqlConnection : IDatabaseConnection
    {
        private SqlConnection(){}

        // SingleTon instance via Lazy<T> for thread safety
        private static readonly Lazy<SqlConnection> _instance = new Lazy<SqlConnection>(() => new SqlConnection());
        public static IDatabaseConnection Instance => _instance.Value;
    }

    // Factory to abstract singleton creation

    public static class DatabaseFactory
    {
        public static IDatabaseConnection Create() => SqlConnection.Instance;
    }

    // Usage
    var db = DatabaseFactory.Create();
```

**Key benifits**:

+ **Performance**: Avoids redundant resource allocation
+ **Consistency**: Guarantees a single point of access.

[Related docs](https://medium.com/@kohzadi90/boosting-api-performance-in-c-how-replicant-supercharges-httpclient-with-smart-caching-999718a0bfb1)

## 3. Strategy + Decorator: Dynamic Behavior Evolution

**Use Case** : Payment processing with interchangeable strategies (Credit Card, Paypal,..) and extensible features (logging, retries).

**Why combine these?**

+ **Strategy** swap algorithms at runtime.
+ **Decorator** layers additional behaviors transparently.

**Enhanced example**:

```c#
    public interface IPaymentStrategy
    {
        Task<PaymentResult> ProccessAsync(decimal amount);
    }

    // Core strategies
    public class CreditCardStrategy : IPaymentStrategy {}
    public class PayPalStrategy : IPaymentStrategy {}

    // Decorator for cross-cutting concerns
    public class RetryDecorator : IPaymentStrategy
    {
        private readonly IPaymentStrategy _inner;
        public ReTryDecorator(IPaymentStrategy inner) => this._inner = inner;

        public async Task<PaymentResult> ProcessAsync(decimal amount)
        {
            for(int i = 0, i < 3, i++)
            {
                var result = await _inner.ProcessAsync(amount);
                if(result.IsSuccess) return result;
                await Task.Delay(1000);
            }

            return PaymentResult.Failure("Payment failed after retries");
        }
    }

    // Usage
    IPaymentStrategy strategy = new RetryDecorator(
        new LoggingDecorator(
            new CreditCardStrategy()
        );
    )

    await strategy.ProcessAsync(100.00);
```

**Key benifits**

+ **Extensibility**: Adding logging, retries, or security without modifying core logic.
+ **Separation of Concerns**: Each decorator handles one responsibility.

[Related docs](https://medium.com/@kohzadi90/why-you-should-replace-switch-statements-with-polymorphism-3478eeb4282e)


## 4. Observer + Mediator : Decoupled Event Architectures

**Use case**: Real-time notifications for order processing (email, SMS, analytics).

**Why combine these?**

+ **Observer** enables event-driven subcriptions.
+ **Mediator** centralizes communication, reducing direct dependencies.

**Enhanced example**:

```c#
    // Mediator act as the communication hub
    public class OrderMediator
    {
        private readonly List<INotificationHandler> _handlers = new();
        
        public void Register(INotificationHandler handler) => _handlers.Add(handler);

        public void Notify(string eventName, Order order)
        {
            foreach(var handler in _handlers)
            {
                handler.Handle(eventName, order);
            }
        }
    }


    // Observer handlers

    public class EmailHandler : INotificationHandler
    {
        public void Handle(string eventName, Order order)
        {
            if(eventName == "OrderProcessed")
                SendEmail(order.UserEmail, "Your order is confirmed");
        }
    }

    // Usage
    var mediator = new OrderMediator();
    mediator.Register(new EmailHandler());
    mediator.Register(new SmsHanlder());

    // In orde processing logic
    mediator.Notify("OrderProcessed", order);
```
**Key benifits**

+ **Decoupling**: Components interact through the mediator, not directly.
+ **Scalability**: Add new subcribers without altering publishers.

[Related docs](https://medium.com/@kohzadi90/understanding-azure-messaging-event-grid-event-hubs-and-service-bus-explained-e398ed4d1b45)

## Best practices for Combining Patterns

**1. Avoid Over-Engineering** : User patterns only when they sovle a clear problem.
+ **Example** : Don't apply SingleTon to stateless services - use Dependency Injection instead.

**2. Leverage .NET's Built-In features** :
+ **Use** `IServiceCollection` for DI, `IObservable<T>` for event streams.

**3. Profile Performace**

+ Monitor patterns like singleton or decorator in high-load scenarios ( see [Top 10 .NET Performance Anti-Patterns You Should Fix Today](https://medium.com/@kohzadi90/top-10-net-performance-anti-patterns-you-should-fix-today-d58f4a682340)).

**4. Favor Composition Over Inheritance**
+ Combine patterns through delegation (e.g., Decorator wraps a Strategy).

REF [here](https://medium.com/@kohzadi90/mastering-design-patterns-in-net-combining-patterns-for-scalable-and-maintainable-code-079a8394eae0)

