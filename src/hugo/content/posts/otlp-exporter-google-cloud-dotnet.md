---
title: "Exporting OpenTelemetry to Google Cloud with .NET"
date: 2026-09-03T12:35:00-04:00
lastmod: 2026-09-03T12:35:00-04:00
summary: Google Cloud supports sending OpenTelemetry data through OTLP. This is how we can do it in .NET
tags:
- aspnet.net core
- dotnet
- google cloud
- opentelemetry
keywords: Google Cloud, gcp, .NET, dotnet, OpenTelemetry, otlp, ASP.NET Core

---

A while back, Google  [Cloud Trace support for OpenTelemetry Protocol](https://cloud.google.com/blog/products/management-tools/opentelemetry-now-in-google-cloud-observability). This is great news, everyone should support OTLP!

There's just one issue though. I happen to work with .NET, I could not find anything about it in their [documentation](https://docs.cloud.google.com/stackdriver/docs/instrumentation/migrate-to-otlp-endpoints).

So here it is, how to use [OpenTelemetry.Exporter.OpenTelemetryProtocol](https://www.nuget.org/packages/opentelemetry.exporter.opentelemetryprotocol/) to send traces to Google Cloud.

# Prerequisites

The documentation is already pretty clear on what permissions are needed, so I won't go into that.

I will also assume that you know:
- How to setup credentials for your service, and that you're already connecting to the platform
- How to setup OpenTelemetry tracing in your application

# Step by step

Let's start by making sure we have the required nuget packages:
- Google.Apis.Auth
- OpenTelemetry.Exporter.OpenTelemetryProtocol
- OpenTelemetry.Instrumentation.AspNetCore

Then, we need to setup resources for your service when adding OpenTelemetry to your service collection.

You will need at minimum:
- The name of your service
- A GCP project ID where to send the traces

```csharp
.ConfigureResource(resource => resource
    .AddService(serviceName: builder.Environment.ApplicationName) 
    .AddAttributes([KeyValuePair.Create<string, object>("gcp.project_id", "<YOUR PROJECT ID>")]))
```

Finally that, we setup the OTLP exporter, while connecting the Google Credential to set the proper `Authorization` header. Use the `https://telemetry.googleapis.com` endpoint, and create a DelegatingHandler from the Google credential instance.

```csharp
.AddOtlpExporter(options =>
{
    options.Endpoint = new Uri("https://telemetry.googleapis.com"); // This should be the same for most folks
    options.HttpClientFactory = () =>
    {
        return new HttpClient(credential.ToDelegatingHandler(new HttpClientHandler())); // A DelegatingHandler requires an inner handler, this is why we have a new HttpClientHandler.
    };
});
```

And that's pretty much it!

# Putting it all together

Here's a complete example.

```csharp
using Google.Apis.Auth.OAuth2;
using Google.Apis.Http;
using OpenTelemetry.Resources;
using OpenTelemetry.Trace;

var builder = WebApplication.CreateBuilder(args);

var credential = GoogleCredential.GetApplicationDefault();
var scopedCredential = credential.CreateScoped("https://www.googleapis.com/auth/cloud-platform"); // Not purely required, but can be useful to limit the token scopes

builder.Services.AddOpenTelemetry()
    .ConfigureResource(resource => resource
        .AddService(serviceName: builder.Environment.ApplicationName) 
        .AddAttributes([KeyValuePair.Create<string, object>("gcp.project_id", "<YOUR PROJECT ID>")]))
    .WithTracing(tracing => tracing
        .AddAspNetCoreInstrumentation()
        .AddOtlpExporter(options =>
        {
            options.Endpoint = new Uri("https://telemetry.googleapis.com");
            options.HttpClientFactory = () =>
            {
                return new HttpClient(scopedCredential.ToDelegatingHandler(new HttpClientHandler()));
            };
        }));

var app = builder.Build();

app.MapGet("/", () => "Hello World!");

app.Run();
```

# Troubleshooting

Not seeing anything in Google Trace? Enable [OpenTelemetry diagnostics](https://opentelemetry.io/docs/languages/dotnet/troubleshooting/#self-diagnostics), the answer is likely in there.