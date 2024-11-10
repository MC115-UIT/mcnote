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