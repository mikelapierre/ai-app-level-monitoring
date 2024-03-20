# Collect application-level metrics with Application Insights on IIS Servers

## Prerequisites

- An Application Insights resource
- [The Application Insights Agent (formerly named Status Monitor V2) configured](https://learn.microsoft.com/en-us/azure/azure-monitor/app/application-insights-asp-net-agent?tabs=getting-started)

## .NET Framework

### [Add the application pools to the Performance Monitor Readers group](https://learn.microsoft.com/en-us/azure/azure-monitor/app/performance-counters?tabs=net-core-new)

```net localgroup "Performance Monitor Users" /add "IIS APPPOOL\NameOfYourPool"```

### [Enable Application Domain Resource Monitoring](https://learn.microsoft.com/en-us/dotnet/standard/garbage-collection/app-domain-resource-monitoring)
    
Add ```<appDomainResourceMonitoring enabled="true"/>``` in the machine Aspnet.config (don't forget 32 and 64 bit)

### Add relevant performance counters to the configuration

Make a copy of C:\Program Files\WindowsPowerShell\Modules\Az.ApplicationMonitor\2.0.0\content\Runtime\ApplicationInsights-recommended.config and add the following counters:

```
<Counters>
    <Add PerformanceCounter="\ASP.NET Applications(??APP_W3SVC_PROC??)\% Managed Processor Time (estimated)/">
    <Add PerformanceCounter="\ASP.NET Applications(??APP_W3SVC_PROC??)\Managed Memory Used (estimated)"/>
</Counters>
```

### Set the Application Insights configuration path with APPINSIGHTS_CONFIGPATH system environment variable 

Set the value to the full path of the copy of ApplicationInsights.config file you created curing the previous step. Make sure the application pool has access to the location (ex: C:\inetpub\ApplicationInsights.config).

### Restart IIS

### Query the counters

Use the following query, the application name will be inclued in the **instance** field (ex: _LM_W3SVC_1_ROOT_app1):

```
performanceCounters
| where category == "ASP.NET Applications"
```

## .NET (formerly Core)

### Add application name and relevant event counters through code

```
builder.Services.ConfigureTelemetryModule<EventCounterCollectionModule>(
    (module, o) =>
    {
        module.Counters.Add(new EventCounterCollectionRequest("System.Runtime", "cpu-usage"));
        module.Counters.Add(new EventCounterCollectionRequest("System.Runtime", "working-set"));
    }    
);
builder.Services.AddSingleton<ITelemetryInitializer, AppNameTelemetryInitializer>();
```

```
public class AppNameTelemetryInitializer : ITelemetryInitializer
{
    public void Initialize(ITelemetry telemetry)
    {
        telemetry.Context.GlobalProperties["AspNetAppName"] = typeof(Program).Assembly.GetName().Name;
    }
}
```

### Query the counters

Use the following query, the application name will be included in the **custom dimentions** in AspNetAppName:

```
customMetrics
| where name contains "System.Runtime"
```