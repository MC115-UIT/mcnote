## Task.Run with DBcontext inner

1. Working version

```c#
    case DbConnectionType.Config:
        if(!await HandlingConfigConnectionAsync(providerConfiguration))
        {
            return CreateResponse(false, "Failed to create the connection using this configuration");
        }
        else
        {
            await _saveConfigAndRequestService.InsertConnectionConfiguratioAsync(
                request.ToConnectionConfigurationCatalog()
            )
        }
```

This works because it maintains the original request scope.

2. Error version

```c#
    case DbConnectionType.Config:
        _ = Task.Run(() => await _saveConfigAndRequestService.InsertConnectionConfigurationAsync(
            request.ToConnectionConfigurationCatalog()
        ));
        if(!await HandlingConfigConnectionAsync(providerConfiguration))
        {
            return CreateResponse(false, "Failed to create the connection using this configuration");
        }
```

This `fails` because `Task.Run` creates a new `Thread` where the original scope(including DbContext) is no long valid

If need to run this functio asynchronously, this is once suitable approach:

```c#
    case DbConnectionType.Config:
        _ = Task.Run( async () =>
        {
            using var scope = _scopeFactory.CreateScope();
            using savingService = scope.ServiceProvider.GetRequiredService<SavingConfigAndRequestService>();
            await savingService.InsertConnectionConfiguratioAsync(request.ToConnectionConfiguration());

        });

        if(!await HandlingConfigConnectionAsync(providerConfiguration))
        {
            return CreateResponse(false, "Failed to create the connection using this configuration");
        }
```
