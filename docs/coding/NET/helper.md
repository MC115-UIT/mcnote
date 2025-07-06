## Reporting pdf from html razor views

```c#
/Install-Package Razor.Template.Core

app.MapGet('invoice-report',async(InvoiceFactory invoiceFactory)=>{

	Invoice invoice = invoiceFactory.Create();
	var html = await RazorTemplateEngine.RenderAysnc(
	'Views/InvoiceReport.cshtml',
	invoice
	);

	var renderer = new ChormePdfRenderer();
	using var pdfDocument = renderer.RendererHtmlAsPdf(html)

	return Results.File(
	pdfDocument.BinaryData,
	'application/pdf',
	$'invoice-{invoice.number}.pdf'
	);
	
});
```


```c#
try
{
	cts.CancelAfter(Timespan.FromMinutes(15))
}
catch (Exception ex)
{
	Log.Information($"Exception caught : {ex}")
	if (ex is OperationCanceledException)
	{
		Log.Information("One or more tasks were canceled")
	}

}
```

##Health check
Link [more detail](https://medium.com/@jeslurrahman/implementing-health-checks-in-net-8-c3ba10af83c3)
-- Memory health check
```c#
using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading;
using System.Threading.Tasks;
using Microsoft.Extensions.Diagnostics.HealthChecks;
using Microsoft.Extensions.Options;


namespace FeedbackService.Api.HealthCheck
{
	public class MemoryHealthCheck : IHealthCheck
	{
		private readonly IOptions<MemoryCheckOptions> _options;
		public MemoryHealthCheck (IOptions<MemoryCheckOptions> options)
		{
			_options = options;
		}
		public string Name => 'memory_check';
		public Task<HealthCheckResult> CheckHealthAsync (
			HealthCheckContext context,
			CancellationToken cancellationToken = default(CancellationToken)
		)
		{
			var options = _options.Get(context.Registration.Name)
				// Include GC information in the reported diagnostic.
			var allocated = GC.GetTotalMemory(forceFullCollection : false);
			var data = new Dictionary<string, object(0)>
			{
				{"Allocated": allocated},
				{"Gen0Collections": GC.CollectionCount(0)},
				{"Gen1Collections": GC.CollectionCount(1)}.
				{"Gen2Collections": GC.CollectionCount(2)}
			};
			var status = (allocated < options.Threshold) ? HealthStatus.Healthy : HealthStatus.UnHealthy;

			return Task.FromResult(new HealthCheck (
				status,
				description : "Reports degraded status if allocated bytes" +
				$">= {options.Threshold} bytes",
				exception : null,
				data : data
			));
		}
	}
	public class MemoryCheckOptions()
	{
		public string MemoryStatus {get; set;}
		// pulic int Threshold {get; set;}
		// Failure in bytes
		public long Threshold {get; set;} = 1024L*1024L*1024L;
	}
}
```

Configure `RemoteHealthCheck.cs` and `MemoryHealthCheck.cs` inside the `HealthCheck.cs`

```c# title="HealthCheck.cs"
public static void ConfigureHealthCheck(this IServiceCollection services, IConfiguration configuration)
{
	services.AddHealthChecks()
		.AddSqlServer(configuration["Connectionstring:Feedback"], healthQuery : "Select 1", name : "SQL Server", failureStatus :  HealthStatus.UnHealthy, tags : new[] {"Feedback,Database"})
			.AddCheck<RemoteHealthCheck>("Remote endpoints Health Check",failureStatus : HealthStatus.UnHealthy)
			.AddCheck<MemoryHealthCheck>($"Feedback Service memory check", failureStatus : HealtheStatus.UnHealthy ,tags : new[] {"Feedback service"})
			.AddUrlGroup(new Uri("https://localhost:44333/api/v1/heartbeats/ping"), name : "Base URL" , failureStatus : HealthStatus.UnHealthy);

	services.AddHealthCheckUI(opt => {
		opt.SetEvaluationTimeSeconds(10) // time in seconds between check
		opt.MaximumHistoryEntriesPerEnpoint(60) // maximum history of checks
		opt.SetApiMaxActiveRequests(1) // api request concurrency
		opt.AddHealthCheckEndpoint("feedback api","/api/health"); // map health check api
	})
		.AddInMemoryStorage();
}

//configure the headlthCheck in main (program.cs)

// Configuring the HealthCheck
builder.Services.ConfigureHealthCheck(builder.Configuration)

//HealthCheck MiddleWare

app.MapHealthChecks("api/health", new HealthCheckOptions()
{
	Predicate => true,
	ResponeWriter = UIResponseWriter.WriteHealthCheckUIRespone
});
app.UseHealthCheckUI(delegaate (Options options)
{
	options.UIPath = "/healthcheck-ui";
	options.AddCustomStyleSheet("./HealthCheck/Custom.css");
})
```

## Expression builder

```c# title="ExpressionBuilder.cs"
public class ExpressionBuilder<T>
{
	// create expression entity => entity.propertyName
	public Expression CreatePropertyExpression(
		Expression entityParameter,
		string propertyName, object? value
	)
	{
		var property= typeof(T).GetProperty(propertyName);
		var propertyAccess = Expression.MakeMemberAccess(entityParameter, property);
		var valueExpression = Expression.Constant(value, property.PropertyType);
		var equalsExpression = Expression.Equal(propertyAccess, valueExpression);

		return equalsExpression;
	}
	// create expression entity => entity.propertyName1 == value1 && entity.propertyName2 == value2 && ...

	public Func<T,bool> ComplexExpression(List<Tupple<string,object> conditions>)
	{
		Expression expression = null;
		var parameter = Expression.Parameter(typeOf(T), "entity");
		
		foreach(var condition in conditions)
		{
			var currentExpression = CreatePropertyExpression(parameter, condition.Item1, condition.Item2);
			if(expression == null)
			{
				expression = currentExpression;
				continue;
			}
			expression = Expression.AndAlso(expression, currentExpression);
		}
		var lambda = Expression.Lambda<Func<T,bool>>(expression, parameter);
		return lambda.Compile();
	}
}

```

Usage 
```c#
	var builder = new ExpressionBuilder<Foo>();

	// this can be create it at runtime
	var conditions = new List<Tupple<string,object>>{
		new Tupple<string,object>("Condition1", "Value1"),
		new Tupple<string,object>("Condition2", "Value2")
		//etc
	}
	var expression = builder.ComplexExpression(conditions);
	var filterd = list.Where(expression).Tolist()
```

## Comparing two object in effcient way

```c#
	public static bool DeepEquals(this object obj, object another)
	{
		if(ReferenceEqual(obj, another)) return true;
		if(obj == null || another == null) return false;
		if(obj.GetType() != another.GetType()) return false;

		var result = true;
		foreach(var property in obj.GetType().GetProperties())
		{
			var objectValue = property.GetValue(obj);
			var anotherValue = property.GetValue(another);
			if(!objectValue.Equals(anotherValue)) result = false
			return result;
		}
	}
	// if the property is also object 
	if(!objectValue.DeepEquals(anotherValue)) result = false

	// with the Json format
	public static bool JsonEquals(this object obj, object another)
	{
		if(ReferenceEqual(obj, another)) return true;
		if(obj == null || another == null) return false;
		if(obj.GetType() != another.GetType()) return false;

		var objJson = JsonConvert.SerializationObject(obj);  
		var anotherJson = JsonConvert.SerializationObject(another);
		
		return objJson == anotherJson;
	}

	// Comparing Two list object
	public static bool DeepEqual<T>( this IEnumerable<T> obj, IEnumerable<T> another)
	{
		if(ReferenceEquals(obb, another)) return true;
		if(obj == null || another == null) return false;

		bool result = true;
		using(var enumerator1 = obj.GetEnumerator())
		using(var enumerator2 = another.GetEnumerator())
		{
			while(true)
			{
				bool hasNext1 = enumerator1.MoveNext();
				bool hasNext2 = enumerator2.MoveNext();

				if(hasNext1 != hasNext2 || !enumerator1.Current.DeepEquals(enumerator2.Current))
				{
					result = false;
					break;
				}
				if(!hasNext1) break;
			}
		}
		return result;
	}
```
## Hepler funcitons with Func<>

``` C#
	public static class SmartHelper
	{
		// Cache for memoization
		private static readonly ConcurrentDictionary<string, object> _cache = new();

		// 1. Memoization helper for expensive computations
		public static Func<TResult> Memoize<TResult> (Func<TResult> func)
		{
			return () =>
			{
				var key = $"{func.Method.Name}";
				return (TResult)_cache.GetOrAdd(key, _ => func());
			};
		}
		
		public void Demo()
		{
			Func<decimal> expensiveCalculation = () =>
			{
				Thread.Sleep(2000);
				return decimal.Parse("123.213");
			}

			var memoizedCalc = SmartHelper.Memoize(expensiveCalculation);

			// firt call take times
			var result1 = memoizedCalc();
			var result2 = memoizedCalc();
			// second call is instant
		}
		

		// 2.Memoization for functions with parameters
		public static Func<T, TResult> Memoize<T, TResult> (Func<T,TResult> func) where T : notnull
		{
			return arg =>
			{
				var key = $"{func.Method.Name}";
				return (TResult)_cache.GetOrAdd(key, _ => func(arg));
			};
		}

		public void Demo()
		{
			 Func<string, int> stringProcessor = str =>
			{
				Thread.Sleep(1000); // Simulate processing
				return str.GetHashCode();
			};


			var memoizedProcessor = SmartHelpers.Memoize(stringProcessor);
        
			// First call for each unique input takes time
			var hash1 = memoizedProcessor("test"); // Takes 1 second
			var hash2 = memoizedProcessor("test"); // Instant (cached)
			var hash3 = memoizedProcessor("different"); // Takes 1 second
		}
		

		// 3. Retry machanism with exponential backoff
		public static Func<T, TResult> WithRetry<T, TResult>(
			Func<T, TResult> func,
			int maxAttempts = 3,
			int initialDelays = 100
		)
		{
			return arg =>
			{
				for(int attempt = 1, attempt <= maxAttempts, attempt ++)
				{
					try
					{
						return func(arg);
					}
					catch(Exception) when ( attempt < maxAttempts)
					{
						Thread.Sleep(initialDelays * (int)Math.Pow(2, attempt-1));
					}
				}
				return func(arg);
			};
		}

		public void Demo()
		{
			Func<int, string> unreliableService = id =>
			{
				if (Random.Shared.Next(0, 2) == 0)
					throw new Exception("Random failure");
				return $"Data for ID: {id}";
			};


			var reliableService = SmartHelper.WithRetry(unreliableService);
			try
			{
				var data = reliableService(123);
				Console.WriteLine(data);
			}
			catch(Exception ex)
			{
				Console.WriteLine("All retry attempts failed");
			}
		}
		
		// 4. Performance measurement decorator
		public static Func<T, TResult> MeasurePerformance<T, TResult>(
			Func<T, TResutl> func,
			Action<TimeSpan> performanceCallback
		)
		{
			return arg => 
			{
				var sw = StopWatch.StartNew();
				var result = func(arg);
				sw.Stop();
				performanceCallback(sw.Elapsed);
				return result;
			};
		}

		public void Demo()
		{
			Func<int, double> complexCalculation = x =>
			{
				Thread.Sleep(100); // Simulate work
				return Math.Pow(x, 2);
			};

			var measuredFunc = SmartHelpers.MeasurePerformance(
				complexCalculation,
				elapsed => Console.WriteLine($"Operation took: {elapsed.TotalMilliseconds}ms")
			);
			var result = measuredFunc(10); 

		}


		// 5. Expression compiler with caching
		private static readonly ConcurrentDictionary<string, Delegate> _compiledExpressions = new();

		public static Func<T, TResutl> CompileAndCache<T, TResult>(Expression<Func<T, TResult>> expression)
		{
			var key = expression.ToString();
			return (Func<T, TResult>)_compiledExpreesions.GetOrAdd(key, _ => expression.Compile());
		}

		public void Demo()
		{
			Expression<Func<int, bool>> isEvenExpr = num => num % 2 == 0;
			var compiledFunc = SmartHelpers.CompileAndCache(isEvenExpr);
			
			var isEven = compiledFunc(4); // true
			var isOdd = compiledFunc(3);  // false

		}

		// 6. Null-safe function wrapper
		public static Func<T, TResult> NullSafe<T, TResult>(Func<T, TResult> func, TResult defaultValue = default ) where T : class
		{
			return arg => arg == null ? defaultValue : func(arg);
		}

		public void Demo()
		{
			Func<string, int> stringLength = str => str.Length;
			var safeLengthFunc = SmartHelpers.NullSafe(stringLength, -1);

			var len1 = safeLengthFunc("test");  // Returns 4
			var len2 = safeLengthFunc(null);    // Returns -1 instead of throwing
		}

		//7. Async function with timeout
		public static async Task<TResult> WithTimeout<TResult>(
			Func<T, TResult> func,
			TimeSpan timeout
		)
		{
			using var cts = new CancellationTokenSource(timeout);
			try
			{
				return await func().WaitAsync(cts.Token);
			}
			catch(OperationCanceledException)
			{
				throw new TimeoutException($"Operation time out after {timeout.TotalSeconds} seconds");
			}
		}
		async Task DemoTimeoutAsync()
        {
            Func<Task<string>> longRunningTask = async () =>
            {
                await Task.Delay(5000); // Simulate long operation
                return "Done!";
            };

            try
            {
                // This will timeout after 2 seconds
                var result = await SmartHelpers.WithTimeout(
                    longRunningTask,
                    TimeSpan.FromSeconds(2)
                );
            }
            catch (TimeoutException ex)
            {
                Console.WriteLine("Operation timed out!");
            }
        }


		// Example combine with UserService
		public class UserService
		{
			private readonly HttpClient _client = new();

			public async Task<UserProfiel> GetUserProfileAsync(int userId)
			{
				// combine meoization, retry, and performance measurement

				Func<int, UserProfile> fetchUser = async id =>
				{
					var response = await _client.GetAsync($"api/users/{id}");
					response.EnsureSuccessStatusCode()
					return await response.Content.ReadFromJsonAsync<UserProfile>();
				};

				var withRetry = SmartHelper.WithRetry(fetchUser);

				var withPerformance = SmartHelper.Measurement(
					withRetry,
					elapsed => Console.WriteLine($"User fetch took {elapsed.TotalMiliseconds}m")
				);

				var resutl = await SmartHelper.WithTimeout(
					() => withPerformance(userId),
					TimeSpan.FromSecond(10)
				);

				return result;
			}
		}
	}
```

## The Result type
When our function needs to represent two states : a happy path and a failure path, we can model it with a generic type `Result<T, E>` where `T` represent
the value and `E` represent the error response. An example function that gets a user could look like this:

```C#
	publuc async Result<User, string> FindByEmailAsync(string email)
	{
		User user = await context.User.FirstOrDefault(u => EF.functions.Like(u.Email, $"%{email}%"));
		if(user is null)
		{
			return "No user found";
		}
		return user;
	}

	[HttpGet("{email}")]
	public async Task<ActionResult<User>> GetByEmail(string email)
	{
		if(string.IsNullOrEmpty(email)) {
			return BadRequest("email cannot be empty");
		}
		Result<User, string> result = await FindByEmail(email);
		return result.Match<ActionResult<User>>(
			user => Ok(user),
			_ => NotFound());
	}
```

Class `Result` type:
```C#
	public readonly struct Result<T, E>
	{
		private readonly bool _success;
		private readonly T Value;
		private readonly E Error;

		private Result<T, E>(T v, E e, bool success)
		{
			Value = v;
			Error = e;
			_success = success;
		}

		public bool isOk() => _success;
		public static Result<T, E> Ok(T v)
		{
			return new(v, default(E), true);
		}
		public static Result<T, E> Err(E e)
		{
			return new(default(T), e, false);
		}

		public static implicit operator Result<T, E>(T v) = new(v, default(E), true);
		public static implicit operator Result<T, E>(E e) = new(default(T), e, false);

		public R Match<R>(
			Func<T, R> success,
			Func<E, R> failure) = > _success ? success(Value) : failure(Error);


	}
```
## Reflection and Attribute combination
`Reflection` allows us to inspect and manipulate objects at runtime. We can use reflection to:

- Inspect Types: Get metadata about types, including properties, methods, fields, and more.
- Create Instances Dynamically: Create objects dynamically using type information obtained through reflection.
- Invoke Methods: Call methods on objects dynamically.
- Access Fields and Properties: Retrieve and set values of fields and properties at runtime.

Example where we use reflection to inspect a class’s metadata.
```C#
	using System;
	using Reflection;

	public class Person
	{
		public string Name {get; set;}
		public int Age {get; set}
		
		public void Introduce()
		{
			Console.WriteLine($"Hi, I'm {Name} and , I'm {Age} years old");
		}
	}
	class Program
	{
		public static void Main(string[] args)
		{
			// Get the type of the class Person
			Type personType = typeof(Person);

			// Get properties of the Person class
			Properties[] properties = personType.GetProperties();
			foreach(var prop in properties)
			{
				Console.WriteLine(prop.Name);
			}

			// Get the method of the Person class
			MethodInfo[] methods = personType.GetMethods();
			foreach(var method in methods)
			{
				Console.WriteLine(method.Name);
			}

			// Create an instance of Person dynamically
			object personInstance = Activator.CreateInstance(personType);
			PropertyInfo nameProperty = personInstance.GetProperty("Name");
			nameProperty.SetValue(personInstance,"Alex");
			// same with other properties

			// Calling the Introduce method dynamically
			MethodInfo introduceMethod = personInstance.GetMethod("Introduce");
			introduceMethod.Invoke(personInstance, null);
		}
	}

```
Create custome `Attributes`, We can define our own custom attributes by inheriting from System.Attribute. Here’s an example of how to create a simple custom attribute:
```C#
	[AttributeUsage(AttributeTargets.Class|AttributeTargets.Method)]
	public class DocumentationAttribute : Attribute
	{
		public string Author {get; set;}
		public string Description {get; set;}

		public DocumentationAttribute(string author, string description)
		{
			Author = author;
			Description = description;
		}
	}

	[Documentation("Alex","This class represents a person")]
	public class Person
	{
		[Documentation("Alex","This method introduce a person")]
		public void Introduce()
		{
			Console.WriteLine("Hello");
		}
	}

```

Now, let’s retrieve and display the custom attribute data using reflection:
```C#
	using System;
	using System.Reflection;
	
	class Program
	{
		public static void Main(string[] args)
		{
			// Get type of the Person class
			Type personType = typeOf(Person);

			// Get custom attributes applied to the Person class
			var classAttributes = personType.GetCustomAttributes(typeOf(DocumentationAttribute), false);
			foreach(DocumentationAttribute arr in classAttributes)
			{
				Console.WriteLine($"Class Author: {attr.Author}, Description: {attr.Description}");
			}

			// Get custom attributes applied to the Introduce method

			MethodInfo methodInfo = personType.GetMethod("Introduce");
			var methodAttributes = methodInfo.GetCustomAttributes(typeOf(DocumentationAttribute), false);
			foreach(DocumentationAttribute attr in methodAttributes)
			{
				Console.WriteLine($"Method Author: {attr.Author}, Description: {attr.Description}");
			}
		}
	}
```
One of the common use cases for reflection and attributes is creating a dynamic system that behaves differently based on the metadata associated with code elements. For example, we might use attributes to mark certain methods for logging, validation, or permission checking.

Let’s look at an example where we use reflection and attributes for method execution tracking:

```C#
	[AtrributeUage(AtrributeTargets.Method)]
	public class LogMethodAttribute : Attribute
	{
		public string Message {get; set;}
		publicc LogMethodAttribute(string message)
		{
			Message = message;
		}
	}

	public class Calculator
	{
		[LogMethod("Adding two numbers")]
		public int Add(int a, int b) => a + b;

		[LogMethod("Subtract two numbers")]
		public int Subtract(int a, int b) => a - b;
	}

	class Program
	{
		public static void Main(string[] args)
		{
			Type calculatorType = typeOf(Calculator);
			MethodInfo[] methodInfos = calculatorType.Getmethods();

			foreach(var method  in methods)
			{
				var logAttribute = method.GetCustomAttribute<LogMethodAttribute>();
				if(logAttribute != null)
				{
					Console.WriteLine($"Logging: {logAttribute.Message}");
					// call the method dynamically

					var instance = Activator.CreateInstance(calculatorType);
					var result = method.Invoke(instance, new object[] {10, 5});

					Console.WriteLine($"Result of {method.Name}: {result}");

				}
			}
		}
	}
```

## 	Stream Transfer Service 

1. Basic Stream pipeline approach
```c#

	public class StreamTransferService
	{
		private readonly HttpClient _httpClient;
		private const int BufferSize = 81290; // 80KB buffer

		public StreamTransferService(HttpClient httpclient)
		{
			this._httpClient = httpClient;
		}

		public async Task StreamTransferAsync(
			string downloadUrl,
			string uplaodUrl,
			IProgress<doulbe> progress = null,
			CancellationToken cancellationToken = default
		)
		{
			using var downloadResponse = await _httpClient.GetAsync(
				downloadUrl,
				HttpCompletionOptions.ResponseHeadersRead,
				cancellationToken
			);

			downloadResponse.EnsureSuccessStatusCode();
			var totalLength = downloadResponse.Content.Headers.ContentLength ?? -1L;

			using var downloadStream = await downloadResponse.Content.ReadAsStreamAsync(cancellationToken);
			using var streamContent = new ProgressStreamContent(
				downloadStream,
				totalLength,
				progress
			);

			using var uploadResponse = await _httpClient.PostAsync(
				uploadUrl,
				streamContent,
				cancellationToken
			);
			uploadResponse.EnsureSuccessStatusCode();
		}
	}
```

2. Advanced Implementation with Chunked Processing

```c#

	public class ChunkStreamTransfer
	{
		private readonly HttpClient _httpClient;
		private const int ChunkSize = 1024 * 1024; // 1MB chunk
		private const int MaxBufferSize = 5 * 1024 * 1024; // 5MB max buffer


		public ChunkStreamTransfer(HttpClient httpClient)
		{
			this._httpClient = httpClient;
		}

		public async Task StreamTransferWithChunkAsync(
			string downloadUrl,
			string uploadUrl,
			IProgress<doulbe> progress = null,
			CancellationToken cancellationToken = default
		)
		{
			using var downlaodResponse = await _httpClient.GetAsync(
				downloadUrl,
				HttpCompletionOption.ResponseHeadersRead,
				cancellationToken
			);

			downloadResponse.EnsureSuccessStatusCode();
			var totalLength = downloadResponse.Content.Headers.ContentLength ?? -1L;


			// Create chunked content
			var chunkedContent = new ChunkedTransferContent(
				downloadResponse.Content.ReadAsStreamAsync(cancellationToken),
				totalLength,
				ChunkSize,
				MaxBufferSize,
				progress);

			using var uploadResponse = await _httpClient.PostAsync(
				uploadUrl,
				chunkedContent,
				cancellationToken);
				
			uploadResponse.EnsureSuccessStatusCode();
		}
	}

	public class ChunkedTransferConntent : HttpContent
	{
		private readonly Task<Stream> _sourceStreamTask;
		private readonly long _totalLength;
		private readonly int _chunkSize;
		private readonly int _maxBufferSize;
		private readonly IProgress<double> _progress;
		private long _byteProcessed;


		public ChunkedTransferContent(
			Task<Stream> sourceStreamTask,
			long totalLength,
			int chunkSize,
			int maxBufferSize,
			IProgess<double> progress
		)
		{
			_sourceStreamTask = sourceStreamTask;
			_totalLength = totalLength;
			_chunkSize = chunkSize;
			_maxBufferSize = maxBufferSize;
			_progress = progress;

			// Set content header of needed
			Headers.ContentType = new MediaTypeHeaderValue("application/octet-stream");
		}

		protected override async Task SerializeToStreamAsync(
			Stream targetStream,
			TransportContext context,
			CancellationToken cancellationToken
		)
		{
			using var sourceStream = await _sourceStreamTask;
			var buffer = new byte[_chunkSize];
			var semaphore = new SemaphoreSlim(_maxBufferSize / _chunkSize);

			while(true)
			{
				await semaphore.WaitAsync(cancellationToken);

				try
				{
					int bytesRead = sourceStream.ReadAsync(
						buffer,
						cancellationToken
					);

					if(bytesRead == 0) break;

					await targetStream.WriteAsync(
						buffer.AsMemory(0, bytesRead),
						cancellationToken
					);

					_byteProcessed += bytesRead;
					ReportProgress();

				}
				finally
				{
					semaphore.Release();
				}
			}
		}
		private void ReportProgress()
		{
			if (_totalLength > 0 && _progress != null)
			{
				var progressPercentage = (double)_bytesProcessed / _totalLength * 100;
				_progress.Report(progressPercentage);
			}
		}
		protected override bool TryComputeLength(out long length)
		{
			length = _totalLength;
			return _totalLength > 0;
		}
	}

```

3.Implementation with Back-pressure Support:

```c#

	public class BackPressureStreamTransfer
	{
		private readonly HttpClient _httpClient;
		private readonly Channel<Memory<byte>> _channel;
		private const int MaxBufferSize = 5;


		public BackPressureStreamTransfer(
			string downloadUrl,
			string uploadUrl,
			IProgress<double> progress = null,
			CancellationToken cancellationToken = defautl
		)
		{
			using var downloadResponse = _httpClient.GetAsync(
				downloadUrl,
				HttpCompletionOption.ResponseHeadersRead,
				cancellationToken
			);

			downdloadResponse.EnsureSuccesStatusCode();
			var totalLength = downloadResponse.Headers.ContentLength ?? -1L;

			// Start producer and consumer tasks

			var downloadTask = DownloadDataAsync(
				downloadResponse,
				progress,
				cancellationToken
			);

			var uploadTask = UploadDataAsync(
				uploadUrl,
				totalLength,
				progress,
				cancellationToken
			);

			// wait for all tasks to complete
			await Task.WhenAll(downloadTask, uploadTask);
		}

		private async Task DownloadDataAsync(
			HttpResponseMessage response,
			IProgress<double> progress,
			CancellationToken cancellationToken
		)
		{
			try
			{
				using var sourceStream = await response.Content.ReadAsStreamAsync(cancellationToken);

				var buffer = new byte[81290] // 80KB chunks

				while(true)
				{
					var bytesRead = await sourceStream.ReadAsync(
						buffer,
						cancellationToken
					);

					if(bytesRead == 0) break;

					var chunk = new byte[bytesRead];
					Array.Copy(buffer, chunk, bytesRead);

					await _channel.WriteAsync(chunk, cancellationToken);
				}
			}
			finally
			{
				_channel.Writer.Complete();
			}
		}

		private async Task UploadDataAsync(
			string uploadUrl,
			long totalLength,
			IProgess<double> progess,
			CancellationToken cancellationToken
		)
		{
			using var content = new PipContent(
				_channel.Reader,
				totalLength,
				progress
			);


			using var response = await _httpClient.PostAsync(
				uploadUrl,
				content,
				cancellationToken
			);

			response.EnsureSuccessStatusCode();
		}

		private class PipeContent : HttpContent
		{
			private readonly ChannelRead<Memory<byte>> _reader;
			private readonly long _totalLength;
			private readonly IProgress<double> _progress;
			private long _byteProcessed;

			public PipeContent(
				ChannelReader<Memory<byte>> reader,
				long totalLength,
				IProgress<double> progress)
			{
				_reader = reader;
				_totalLength = totalLength;
				_progress = progress;
				_bytesProcessed = 0;
			}

			protected override async Task SerializeToStreamAsync(
				Stream targetStream,
				TransportContext context,
				CancellationToken cancellationToken
			)
			{
				await foreach ( var chunk in _reader.ReadAllAsync(cancellationToken))
				{
					await targetStream.WriteAsync(
						chunk,
						cancellationToken
					);

					ReportProgess();
				}
			}
			private void ReportProgress()
			{
				if (_totalLength > 0 && _progress != null)
				{
					var progressPercentage = (double)_bytesProcessed / _totalLength * 100;
					_progress.Report(progressPercentage);
				}
			}

			protected override bool TryComputeLength(out long length)
			{
				length = _totalLength;
				return _totalLength > 0;
			}
		}
	}
	
	/// Usage Example 

	async Task Example
	{
		using var httpClient = new HttpClient(configuration);
		var transfer = new BackPressureStreamTransfer(httpClient);

		var progress = new Progress<double>(percentage =>
		{
			await CommonService.LogService.LogInformation($"Transfer progess : {percentage:F2}%");
		});

		try
		{
			await transfer.StreamTransferWithBackPressureAsync(
				_downloadUrl,
				_uploadUrl,
				progress
			);
		}
		catch (Exception ex)
		{
			await CommonService.LogService.LogError($"Transfer failed : {ex.Message}");
		}
	}
```
## Complied Quries for commonly repeated queries

Ef Core supports the compiled queries for frequently executed queries

```c#
	private static readonly Func<AppDbContext, bool, Task<List<T>>> _getEntitiesByStatus<T> Where T : class, IActivatable = EF.CompileAsyncQuery((AppDbContext context, bool isActive) => 
		context.Set<T>().Where(e => e.IsActive == isActive).ToListAsync()
	);

	public interface IActivatable
	{
		bool IsActive {get; set;}
	}

	// Example entity
	public class User : IActivatable
	{
		public int Id {get; set;}
		public string Name {get; set;}
		public bool isActive {get; set}
	}

	// Example Usage
	var activeUsers = await _getEntitiesByStatus<User>(_contextDb, true);
```

## Polymorphism with `AddKeyedSingleton`, considered replacement of switch statement

```c#
	public interface IPaymentProcessor
	{
		void ProcessPayment();
	}

	public class CreditCardPaymentProcessor : IPaymentProcessor
	{
		public void ProcessPayment()
		{
			// Handle credit card payment
		}
	}

	public class PayPalPaymentProcessor : IPaymentProcessor
	{
		public void ProcessPayment()
		{
			// Handle PayPal payment
		}
	}

	public class BitcoinPaymentProcessor : IPaymentProcessor
	{
		public void ProcessPayment()
		{
			// Handle Bitcoin payment
		}
	}

	// registration of service

	builder.services.AddKeyedSingleton<IPaymentProcessor,CreditCardPaymentProcessor>(PaymentType.CrediCard);
	builder.services.AddKeyedSingleton<IPaymentProcessor,PayPalPaymentProcessor>(PaymentType.PayPa;);
	builder.services.AddKeyedSingleton<IPaymentProcessor,BitcoinPaymentProcessor>(PaymentType.Bitcoin);

	// then resolve it dynamically

	public class PaymentService
	{
		private readonly IServiceProvider _seviceProvider;

		public PaymentService(IServiceProvider serviceProvider)
		{
			this._serviceProvider = serviceProvider;
		}

		public void ProcessPayment(PaymentType paymentType)
		{
			var paymentProcessor = _serviceProvider.GetRequiredKeyedService<IPaymentProcessor>(paymentType);
			paymentProcessor?.ProcessPayment() ?? throw new InvalidOperationException($"No payment type such as {paymentType}");
		}
	}
```

## Cursor pagination

Example with `Date` and `Id` field to create a cursor for our `UserNotes` table. The cursor is a
composite of these two fields, allowing us to paginate effciently.

```c#

	app.MapGet("/cursor", async (
		AppDbContext dbContext,
		DateOnly? date = null,
		Guid? lastId = null,
		int limit = 10,
		CancellationToken cancellationToken = default
	) => {
		
		if(limit < 1 ) return Results.BadRequest("Limit must be greater than 0");
		if(limit > 100) return Result.BadRequest("Limit must be less than or equal 100");

		var query = dbContext.UserNotes.AsQueryable();

		if(date != null && lastId != null)
		{
			query = query.Where(x => x.Date < date || (x.Date == date && x.Id <= lastId));
		}


		var items = await query
			.OrderByDescending(x => x.Date)
			.ThenByDescending(x => x.Id)
			.Take(limit + 1)
			.ToListAsync(cancellationToken)

		bool hasMore = item.Count > limit;
		DateOnly? nextDate = hasMore ? items[^1].Date : null;
		Guid? nextId = hasMore ? item[^1].Id : null;

		items.RemoveAt(items.Count - 1);

		return Result.Ok(new {
			Items = items,
			NextDate = nextDate,
			NextLastId = nextLastId,
			HasMore = hasMore
		});
	});
```
**Encoding the Cursor**

Here's a small utility class for encoding and decoding the cursor. We'll use this to encode the cursor in the URL and decode it when fetching the next set or results.

The clients will receive the cursor as a Base64-encoded string. They don't need to know the internal structure of the cursor

```c#
	using Microsoft.AspNetCore.Authentication;

	public sealed record Cursor(DateOnly Date, Guid LastId)
	{
		public static string Encode(DateOnly date, Guid lastId)
		{
			var cursor = new Cursor(date, lastId);
			string json = JsonSerializer.Serialize(cursor);
			return Base64UrlTextEncoder.Encode(Encoding.UTF8.GetBytes(json));
		}

		public static string Cursor? Decode(string? cursor)
		{
			if(string.IsNullOrEmpty(cursor)) return null;

			try
			{
				string json = Encoding.UTF8.GetString(Base64UrlTextEncoder.Decode(cursor));
				return JsonSerializer.Deserialize<Cursor>(json);
			}
			catch
			{
				return null;
			}

		}
	}


	// Usage
	string encodedCursor = Cursor.Encode(
	new DateOnly(2025, 2, 15),
	Guid.Parse("019500f9-8b41-74cf-ab12-25a48d4d4ab4"));
	// Result:
	// eyJEYXRlIjoiMjAyNS0wMi0xNSIsIkxhc3RJZCI6IjAxOTUwMGY5LThiNDEtNzRjZi1hYjEyLTI1YTQ4ZDRkNGFiNCJ9

	Cursor decodedCursor = Cursor.Decode(encodedCursor);
	// Result:
	// {
	//     "Date": "2025-02-15",
	//     "LastId": "019500f9-8b41-74cf-ab12-25a48d4d4ab4"
	// }
```

## Sample class for customize the exception handler

```c#

	using Microsoft.AspNetCore.Diagnostics;
	using Microsoft.AspNetCore.Mvc;


	namespace CleanArchitecture.Web.Infrastructure;

	public class CustomExceptionHandler : IExceptionHandler
	{
		private readonly Dictionary<Type, Func<HttpContext, Exception, Task>> _exceptionHanlder;

		public CustomExceptionHandler()
		{
			this._exceptionHandler = new ()
			{
				{typeof(ValidationException), HandlerValidationException},
				{typeof(NotFoundException), HandlerNotFoundException},
				{typeof(UnAuthorizedException, HandlerUnAuthorizedException)},
				{typeof(ForbiddenAccessException), HandlerForbiddenAccessException},
			};
		}

		public async ValueTask<bool> TryHandlerAsync(HttpContext httpContext, Exception exception, CancellationToken cancellationToken)
		{
			var exceptionType = exception.GetType();
			if(_exceptionHandler.ContainsKey(exceptionType))
			{
				await _exceptionHandler[exceptionType].Invoke(httpContext, exception);
				return true;
			}
			return false;
		}

		private async Task HandlerValidationException(HttpContext httpContext, Exception ex)
		{
			var exception = (ValidationException)ex;

			httpContext.Response.StatusCode = StatusCodes.Status400BadRequest;

			await httpContext.Response.WriteAsync(new ValidationProblemDetails(exception.Error)
			{
				Status = StatusCodes.Status400BadRequest,
				Type = "https://tools.ietf.org/html/rfc7231#section-6.5.1"
			});
		}
	    // likely implementing for other exception handler

		private async Task HandlerNotFoundException(HttpContext httpContext, Exception ex)
		{
			var exception = (NotFoundExceptio)ex;

			httpContext.Response.StatusCode = StatusCodes.Status400BadRequest;

			await httpContext.Response.WriteAsync(new NotFoundException(exception.Error)
			{
				Status = StatusCodes.Status400BadRequest,
				Type = "linkforreferdocument"
			});
		}
	}
```

## Serialization and Deserialization with protobuf and automatically generating proto file

- To starting using protobuf in .NET projects, we need to install the `protobuf-net` Nuget package. This package provides support for defining Protobuf schemas using C# attributes and generating `.proto` files from our C# class.

- Define data models

```c#
	
	[ProtoContract]
	public class Person
	{
		[ProtoMember(1)]
		public int Id {get; set;}

		[ProtoMember(2)]
		public string Name {get; set;}

		[ProtoMember(3)]
		public string Email {get; set;}
	}
```
- Example class for ser/deserialize

```c#

	using Protobuf;
	using System.IO;

	public static class ProtoHelpers
	{
		public static byte[] Serialize<T>(T source)
		{
			using var stream = new MemoryStream();
			Serializer.Serialize(stream, source);
			return stream.ToArray();
		}

		public static T DeSerialize<T>(byte[] source)
		{
			using var stream = new MemoryStream(source);
			return Serializer.Deserialize<T>(stream);
 		}
	}
```

- Usage

```c#

	using System;

	public class Program
	{
		public static void Main(string[] args)
		{
			Person person = new Person
			{
				Id = 123,
				Name = "John Doe",
				Email = "john.doe@example.com"
			};

			// Serialize the person to a byte array
			byte[] serializedPerson = ProtoHelpers.Serialize(person);

			// Deserialize the byte array back into a Person object
			Person deserializedPerson = ProtoHelpers.Deserialize<Person>(serializedPerson);

			Console.WriteLine($"ID: {deserializedPerson.Id}");
			Console.WriteLine($"Name: {deserializedPerson.Name}");
			Console.WriteLine($"Email: {deserializedPerson.Email}");
		}
	}
```

- Generating .proto Files

```c#
	using ProtoBuf.Meta;
	using System;
	using System.Collections.Generic;
	using System.IO;
	using System.Linq;
	using System.Text;

	public static class ProtoUtils
	{
		public static string GenerateProto(IEnumerable<Type> protoTypes, string packageName)
		{
			StringBuilder sb = new StringBuilder();
			sb.AppendLine("syntax = \"proto2\";");
			sb.AppendLine("");
			sb.AppendLine($"package {packageName};");
			sb.AppendLine("option optimize_for = LITE_RUNTIME;");
			sb.AppendLine("option cc_enable_arenas = true;");
			sb.AppendLine("");

			var getProto = typeof(RuntimeTypeModel).GetMethod("GetSchema", new[] {typeof(Type)});

			Dictionary<string, string> allProto = new Dictionary<string, string>();
			List<string> removeNames = new List<string>{"\r", "package"};

			foreach(var type in protoTypes)
			{
				string result = (string)getProto.Invoke(RuntimeTypeModel.Default, new object[]{type});
				List<string> splitMsg = result.Split(new[] {"message"}, StringSplitOptions.None).ToList();

				foreach(string s in splitMsg)
				{
					List<string> splitEnum = s.Split(new[] {"enum"}, StringSplitOptions.None).ToList();

					string sMsg = splitEnum[0];
					string sMsgName = sMsg.Split(new[] {"{"}, StringSplitOptions.None)[0];
					if(removeNames.All(n => !sMsg.StartWith(n)))
					{
						allProto[sMsgName] = "message" + sMsg;
					}

					removeNames.Add(sMsgName);
					if(splitEnum.Count > 0)
					{
						for(int i = 0; i < splitEnum.Count; i++)
						{
							string sEnum = splitEnum[i];
							string sEnumName = sEnum.Split(new[] {"{"}, StringSplitOptions.None)[0];
							if(removeNames.All(n => !sEnum.StartWith(n)))
							{
								allProto[sEnumName] = "enum" + sEnum;
							}
							removeNames.Add(sEnumName);
						}
					}
				}
			}
		}
		string proto = string.Join("", allProto.OrderBy(a => a.Key).Select(a => a.Value));

        sb.Append(proto);
        string protoContent = sb.ToString();

        GenerateProtoFile(packageName, protoContent);
        return protoContent;
	}

	public static void GenerateProtoFile(string protoFileName, string protoContent)
    {
        if (string.IsNullOrWhiteSpace(protoFileName)) return;

        if (!protoFileName.EndsWith(".proto"))
        {
            protoFileName += ".proto";
        }

        string filePath = Path.Combine(AppContext.BaseDirectory, protoFileName);

        // Overwrite if already exists
        File.WriteAllText(filePath, protoContent);
    }

	// Usage 
	public static string GeneratePersonProto()
	{
		string packageName = "PersonProto";

		List<Type> protoTypes = new List<Type>
		{
			typeof(Person)
		};

		return ProtoUtils.GenerateProto(protoTypes, packageName);
	}
	
	// This will generate a `person.proto` file that can be used for interproperability with other systems and languages.2w2
```

##  Expression-based approach - dynamic linq

```c#
	public class ExpressionBuilder
	{
		public static Expression<Func<T,bool>> BuilderBuildPredicate<T>(string propertyName, string comparision,
		object value)
		{
			var parameter = Expression.Parameter(typeof(T), "x");
			var property = Expression.Property(parameter, propertyName);
			var constant = Expreesion.Constant(value);

			Expression condition;
			switch(comparision.ToLower())
			{
				case "equals":
					condition = Expression.Equal(property, constant);
					break;
				case "contains":
					var method = typeof(string).GetMethod("Contains", new[] {typeof(string)});
					condition = Expression.Call(property, method, constant);
					break;
				case "greaterthan":
					condition = Expression.GreaterThan(property, constant);
					break;
				case "lessthan":
					condition = Expression.LessThan(property, constant);
					break;
				default:
					throw new ArgumentException("Invalid comparision operator");
			}
			
			return Expression.Lamda<Func<T,bool>>(condition, parameter);
		}
	}


	// Usage

	public IQueryable<Product> GetFilteredProductsWithExpressions(Dictionary<string, object> filters)
	{
		var query = _dbContext.Products.AsQueryable();
		foreach(var filter in filters)
		{
			var predicate = ExpressionBuilder.BuildPredicate<Product>(filter.Key, "equals", filter.Value);
			query = query.Where(predicate);
		}

		return query;
	}
```
## HandFire on background-job

### Helper functions
```c#

namespace HandFire
{
	#region Main HandFireHelper

	public static class HandFireHelper
	{
		private static readonly IHandFireClient _handFireClient;

		public static HandFireHelper()
		{
			_handFireClient = new HandFireClient();
		}


		#region Fire-and-Forget Jobs

		public static string Enqueue<T>(Expression<Action<T>> methodCall, RetryOptions)
		{
			retryOptions ?? = DefaultRetryOptions;

			return ExecuteWithRetry(() => 
			{
				var jobId = _handFireClient.Enqueue(methodCall);
				_handFireClient.SetJobParamters(jobId, "RetryOptions", retryOptions);
				return jobId;
			}, "Enqueue");
		}

		public static string EnqueueAsync<T>(Expression<Func<T, Task>> methodCall, RetryOptions retryOptions = null)
        {
            retryOptions ??= DefaultRetryOptions;

            return ExecuteWithRetry(() =>
            {
                var jobId = _handFireClient.Enqueue(methodCall);
                _handFireClient.SetJobParameter(jobId, "RetryOptions", retryOptions);
                return jobId;
            }, "EnqueueAsync");
        }

		#endregion

		#region Delayed Jobs

		public static string Schedule<T>(Expression<Action<T>> methodCall, TimeSpan delay, RetryOptions retryOptions = null)
		{
			retryOptions ??= DefaultRetryOptions;

            return ExecuteWithRetry(() =>
            {
                var jobId = _handFireClient.Schedule(methodCall, delay);
                _handFireClient.SetJobParameter(jobId, "RetryOptions", retryOptions);
                return jobId;
            }, "Schedule");
		}

		 public static string ScheduleAt<T>(Expression<Action<T>> methodCall, DateTimeOffset enqueueAt, RetryOptions retryOptions = null)
        {
            retryOptions ??= DefaultRetryOptions;

            return ExecuteWithRetry(() =>
            {
                var jobId = _handFireClient.Schedule(methodCall, enqueueAt);
                _handFireClient.SetJobParameter(jobId, "RetryOptions", retryOptions);
                return jobId;
            }, "ScheduleAt");
        }

		#endregion

		#region Recurring Jobs

		public static void AddOrUpdateRecurring<T>(
			string recurringJobId,
			Expression<Action<T>> methodCall,
			string cronExpression,
			RecurringJobOptions options = null
		)
		{
			var manager = new RecurringJobManager();

			manager.AddOrUpdate(recurringJobId, methodCall, cronExpression, options ?? new RecurringOptions());
		}

		public static void AddOrUpdateRecurringAsync<T>(
			string recurringJobId,
			Expression<Func<T, Task>> methodCall,
			string cronExpression,
			RecurringOptions options = null
		)
		{
			var manager = new RecurringJobManager();
			manager.AddOrUpdate(recurringJobId, methodCall, cronExpression, options ?? new RecurringOptions());
		}

		public static void RemoveRecurring(string recurringJobId)
        {
            var manager = new RecurringJobManager();
            manager.RemoveIfExists(recurringJobId);
        }

		#endregion

		public static string ContinueWith<T>(
            string parentJobId, 
            Expression<Action<T>> methodCall, 
            RetryOptions retryOptions = null)
        {
            retryOptions ??= DefaultRetryOptions;

            return ExecuteWithRetry(() =>
            {
                var jobId = _handFireClient.ContinueJobWith(parentJobId, methodCall);
                _handFireClient.SetJobParameter(jobId, "RetryOptions", retryOptions);
                return jobId;
            }, "ContinueWith");
        }

        public static string ContinueWithOnState<T>(
            string parentJobId, 
            Expression<Action<T>> methodCall, 
            JobContinuationOptions options,
            RetryOptions retryOptions = null)
        {
            retryOptions ??= DefaultRetryOptions;

            return ExecuteWithRetry(() =>
            {
                var jobId = _handFireClient.ContinueJobWith(parentJobId, methodCall, options);
                _handFireClient.SetJobParameter(jobId, "RetryOptions", retryOptions);
                return jobId;
            }, "ContinueWithOnState");
        }

        #endregion

		#region Error Handling & Retry Configuration

		public class ReTryOptions
		{
			public int MaxRetries {get; set;} = 3;
			public TimeSpan InitialRetryInterval {get; set;} = TimeSpan.FromSeconds(30);
			public TimeSpan MaxRetryInterval {get; set;} = TimeSpan.FromHours(1);
			public bool UseExponentialBackoff {get; set;} = true;
			public Type[] RetryableExceptions {get; set;} = new[] {typeof(Exception)};
			public Action<Exception, int, TimeSpan> OnReTry {get; set;}

			private static RetryOptions DefaultRetryOptions = new RetryOptions();

			public static void ConfigureDefaultRetryOptions(Action<RetryOptions> configure)
        	{
            configure(DefaultRetryOptions);
        	}
		}
		
		#endregion

		#region Job Recovery and Monitoring

        public static class JobRecovery
        {
            public static async Task RecoverFailedJobs(
                TimeSpan failedJobAge,
                int batchSize = 100,
                CancellationToken cancellationToken = default)
            {
                var failedJobs = await _handFireClient.GetFailedJobs(failedJobAge, batchSize);
                
                foreach (var job in failedJobs)
                {
                    if (cancellationToken.IsCancellationRequested)
                        break;

                    try
                    {
                        await _handFireClient.RetryJob(job.Id);
                        CommonServices.LoggingService.WriteInformation("Recovered failed job {JobId}", job.Id);
                    }
                    catch (Exception ex)
                    {
                        CommonServices.LoggingService.WriteError.(ex, "Failed to recover job {JobId}", job.Id);
                    }
                }
            }
        }

		public static class JobMonitoring
		{
			public static async Task<JobHealthStatus> GetJobHealthStatus()
			{
				return new JobHealthStatus
				{
					FailedJobs = await _handFireClient.GetFailedJobCount(),
                    ProcessingJobs = await _handFireClient.GetProcessingJobCount(),
                    ScheduledJobs = await _handFireClient.GetScheduledJobCount(),
                    RetryingJobs = await _handFireClient.GetRetryingJobCount()
				}
			}
		}

		#endregion

		#region Supporting Classes

		public class RecurringJobOptions
		{
			public TimeZoneInfo TimeZone { get; set; }
			public string QueueName { get; set; }
			public bool EnableRetries { get; set; } = true;
		}

		public class JobContinuationOptions
		{
			public JobContinuationState[] States { get; set; }
		}

		public class JobHealthStatus
		{
			public long FailedJobs { get; set; }
			public long ProcessingJobs { get; set; }
			public long ScheduledJobs { get; set; }
			public long RetryingJobs { get; set; }
		}

		public enum JobContinuationState
		{
			Succeeded,
			Failed,
			Deleted
		}

		public class JobFailedException : Exception
		{
			public IReadOnlyList<Exception> RetryExceptions { get; }

			public JobFailedException(string message, Exception innerException, List<Exception> retryExceptions)
				: base(message, innerException)
			{
				RetryExceptions = retryExceptions;
			}
		}

		public class HandFireOperationException : Exception
		{
			public HandFireOperationException(string message, Exception innerException)
				: base(message, innerException)
			{
			}
		}

		#endregion

		#region Extension Methods

		public static class HandFireExtensions
		{
			public static string EnqueueHandFire<T>(this T instance, Expression<Action<T>> methodCall)
			{
				return HandFireHelper.Enqueue(methodCall);
			}

			public static string ScheduleHandFire<T>(this T instance, Expression<Action<T>> methodCall, TimeSpan delay)
			{
				return HandFireHelper.Schedule(methodCall, delay);
			}
		}

		#endregion
	}

	#endregion
}
```

### Example using
`
```c#
	#region Example Implementation

    public interface IMyService
    {
        void DoWork();
        Task DoWorkAsync();
        void ProcessLater();
        void DailyCleanup();
        void AfterWorkCompleted();
    }

    public class MyService : IMyService
    {
        public void DoWork() 
        {
            // Implementation
        }

        public async Task DoWorkAsync()
        {
            await Task.Delay(1000);
        }

        public void ProcessLater() 
        {
            // Implementation
        }

        public void DailyCleanup() 
        {
            // Implementation
        }

        public void AfterWorkCompleted() 
        {
            // Implementation
        }
    }

    public class HandFireImplementationExample
    {
        public static void ConfigureAndUse()
        {
            // Configure default retry options
            HandFireHelper.ConfigureDefaultRetryOptions(options =>
            {
                options.MaxRetries = 5;
                options.InitialRetryInterval = TimeSpan.FromSeconds(10);
                options.MaxRetryInterval = TimeSpan.FromMinutes(30);
                options.UseExponentialBackoff = true;
                options.RetryableExceptions = new[]
                {
                    typeof(TimeoutException),
                    typeof(HttpRequestException),
                    typeof(DbException)
                };
                options.OnRetry = (exception, retryCount, delay) =>
                {
                    Console.WriteLine($"Retry {retryCount} scheduled after {delay.TotalSeconds} seconds");
                };
            });

            // Fire-and-forget job
            var fireForgetJobId = HandFireHelper.Enqueue<IMyService>(x => x.DoWork());

            // Delayed job
            var delayedJobId = HandFireHelper.Schedule<IMyService>(
                x => x.ProcessLater(),
                TimeSpan.FromHours(1));

            // Recurring job
            HandFireHelper.AddOrUpdateRecurring<IMyService>(
                "daily-cleanup",
                x => x.DailyCleanup(),
                "0 0 * * *", // Daily at midnight
                new RecurringJobOptions 
                { 
                    TimeZone = TimeZoneInfo.Utc,
                    EnableRetries = true
                });

            // Continuation job
            var continuationJobId = HandFireHelper.ContinueWith<IMyService>(
                fireForgetJobId,
                x => x.AfterWorkCompleted());

            // Using extension method
            var myService = new MyService();
            var extensionJobId = myService.EnqueueHandFire(x => x.DoWork());
        }

        public static async Task MonitorAndRecover()
        {
            // Monitor job health
            var healthStatus = await HandFireHelper.JobMonitoring.GetJobHealthStatus();
            Console.WriteLine($"Failed Jobs: {healthStatus.FailedJobs}");
            Console.WriteLine($"Processing Jobs: {healthStatus.ProcessingJobs}");
            Console.WriteLine($"Scheduled Jobs: {healthStatus.ScheduledJobs}");
            Console.WriteLine($"Retrying Jobs: {healthStatus.RetryingJobs}");

            // Recover failed jobs
            await HandFireHelper.JobRecovery.RecoverFailedJobs(
                failedJobAge: TimeSpan.FromHours(24),
                batchSize: 50);
        }
    }

    #endregion
```