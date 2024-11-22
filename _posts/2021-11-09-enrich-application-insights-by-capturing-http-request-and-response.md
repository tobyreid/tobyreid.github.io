---
layout: post
title: Enrich Application Insights by capturing HTTP request & response
date: 2021-11-09 00:00:00 +0000
summary: "Log everything"
minutes: 5

---

> *Disclaimer* - this work was heavily inspired from #insert blog link here when i find it!#, but i've also chosen to log HTTP request and response headers which is why I'm repeating it here.

Application Insights is a great tool out-of-the-box solution for APM (Application Performance Management) and diagnosing inter-system issues, when paired with the .NET framework.

The [default dependency auto-collection](https://docs.microsoft.com/en-us/azure/azure-monitor/app/auto-collect-dependencies) does a good job of capturing the essential metadata to allow basic activity monitoring, but the HTTP auto-collection functionality doesn't include capturing HTTP body and header information.

Fortunately, Application Insights offers extensibility with the classes implementing the [ITelemetryInitializer](https://docs.microsoft.com/en-us/dotnet/api/microsoft.applicationinsights.extensibility.itelemetryinitializer?view=azure-dotnet) interface.

First we want to create some custom middleware that reads the incoming and outgoing http request stream and converts it to a string, so we can log it later on-

```csharp
/// <summary>
/// Intercepts and writes HTTP request body to <see cref="RequestTelemetry"/> as RequestBody custom property
/// </summary>
public class HttpRequestBodyLoggingMiddleware : IMiddleware
{
    /// <inheritdoc />
    public async Task InvokeAsync(HttpContext context, RequestDelegate next)
    {
        var method = context.Request.Method;

        // Only if we are dealing with POST or PUT, GET and others shouldn't have a body
        if (context.Request.Body.CanRead && (method == HttpMethods.Post || method == HttpMethods.Put || method == HttpMethods.Patch))
        {
            // Ensure the request body can be read multiple times
            context.Request.EnableBuffering();
            // Leave stream open so next middleware can read it
            using var reader = new StreamReader(
                context.Request.Body,
                Encoding.UTF8,
                detectEncodingFromByteOrderMarks: false,
                bufferSize: 512, leaveOpen: true);

            var requestBody = await reader.ReadToEndAsync();

            // Reset stream position, so next middleware can read it
            context.Request.Body.Position = 0;

            // Write request body to App Insights
            var requestTelemetry = context.Features.Get<RequestTelemetry>();
            requestTelemetry?.Properties.Add("RequestBody", requestBody);
        }

        // Call next middleware in the pipeline
        await next(context);
    }
}
```

 to simply create your own custom initializer, and register it with your configured dependency injection provider as follows:

```c#
public class HttpDependencyEnrichedTelemetryInitializer : ITelemetryInitializer
{
    /// <inheritdoc />
    public void Initialize(ITelemetry telemetry)
    {
        if (!(telemetry is DependencyTelemetry dependencyTelemetry) ||
            !dependencyTelemetry.TryGetOperationDetail("HttpResponse", out var reponseObject)) return;

        if (!dependencyTelemetry.TryGetOperationDetail("HttpRequest", out var requestObject)) return;

        if (!(reponseObject is HttpResponseMessage response))
            return;

        if (!(requestObject is HttpRequestMessage request))
            return;

        foreach (var (key, value) in request.Headers)
            dependencyTelemetry.Properties.Add($"RequestHeader-{key}", string.Join(", ", value));

        foreach (var (key, value) in response.Headers)
            dependencyTelemetry.Properties.Add($"ResponseHeader-{key}", string.Join(", ", value));

        if (request.Content != null)
        {
            ExtractRequest(request, dependencyTelemetry);
        }

        if (response.Content != null)
        {
            ExtractResponse(response, dependencyTelemetry);
        }
    }

    private static void ExtractRequest(HttpRequestMessage request, ISupportProperties dependencyTelemetry)
    {
        if (request.Content == null)
            return;
        if (!(request.Content is StringContent)
            && !IsTextBasedContentType(request.Headers)
            && !IsTextBasedContentType(request.Content?.Headers))
            return;

        var content = request.Content.ReadAsStringAsync().GetAwaiter().GetResult();
        dependencyTelemetry.Properties.Add("RequestContent", content);
    }

    private static void ExtractResponse(HttpResponseMessage response, ISupportProperties dependencyTelemetry)
    {
        if (response.Content == null)
            return;
        foreach (var (key, value) in response.Content.Headers)
            dependencyTelemetry.Properties.Add($"ContentHeader-{key}", string.Join(", ", value));

        if (!(response.Content is StringContent)
            && !IsTextBasedContentType(response.Headers)
            && !IsTextBasedContentType(response.Content?.Headers))
            return;

        var content = response.Content.ReadAsStringAsync().GetAwaiter().GetResult();
        dependencyTelemetry.Properties.Add("ResponseContent", content);
    }

    private static bool IsTextBasedContentType(HttpHeaders headers)
    {
        if (headers == null)
            return false;
        if (!headers.TryGetValues("Content-Type", out var values))
            return false;

        var header = string.Join(" ", values).ToLowerInvariant();
        var textBasedTypes = new[] { "html", "text", "xml", "json", "txt", "x-www-form-urlencoded" };
        return textBasedTypes.Any(t => header.Contains(t));
    }
}
```

Then register it with the DI container:

```c#
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        services.AddSingleton<ITelemetryInitializer, HttpDependencyEnrichedTelemetryInitializer>();
        services.AddTransient<HttpRequestBodyLoggingMiddleware>();
        services.AddTransient<HttpResponseBodyLoggingMiddleware>();
        // other service configuration
    }
    public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
    {
        app.UseMiddleware<HttpResponseBodyLoggingMiddleware>();
        app.UseMiddleware<HttpRequestBodyLoggingMiddleware>()
        // other application builder configuration
    }
}
```
