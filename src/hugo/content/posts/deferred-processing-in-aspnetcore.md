---
title: "Deferred processing in ASP.NET Core 6"
date: 2022-07-17T18:01:55-04:00
date: 2022-07-17T18:01:55-04:00
summary: How to process tasks in the background after a request has returned in ASP.NET Core 6
tags:
- ASP.NET Core
- .NET
- Channels
keywords: .Net, ASP.NET Core, Channels
---

Nowadays, a common pattern we see when treating HTTP requests is to do a minimal amount of work before returning a response, while treating longer operations (ex: sending email) in the background after the request has been completed.

Unfortunately, in ASP.NET, we often see this being done in questionable ways, mainly: 
- Not using `await` on `Task` object
- Wrapping some code in `Task.Run` and not awaiting the results.

This can lead to dependency injection issues (if your request ends, all of our scoped objected are gone), or having exceptions thrown into the wind.

Fortunately, .NET has it's own internal publisher/subscriber-like feature, called Channels.

# Channels

Channels were first introducted in .NET Core 3.0. While they've had a few improvements since, the abstract classes were mostly unchanged.

## The basics

A lot has been written about how to usage channels, so I won't go in too deep here.

First, let's go over the building blocks:
- `Channel`: Static class used to create channels
- `Channel<T>`: Abstract class representing a channel
- `ChannelReader<T>` and `ChannelWriter<T>`: Abstract classes representing readers and writers to a channel

A simple setup would be:

```csharp
var channel = Channel.CreateUnbounded<Model>();
var reader = channel.Reader;
var writer = channel.Writer;
```

# Integration with ASP.NET Core

We'll be assuming that we want to process a model named, appropriately, `Model`.

## Background service

First, let's created a [HostedService](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/host/hosted-services?view=aspnetcore-6.0&tabs=visual-studio) to receive the messages.

We'll be inheriting from `BackgroundService` since it makes the implementation a little simpler.

```csharp
public class ModelService : BackgroundService
{
    private readonly ChannelReader<Model> _channelReader;
    private readonly ILogger<Service> _logger;

    public ModelService(ChannelReader<Model> channelReader, ILogger<Service> logger)
    {
        _channelReader = channelReader;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (await _channelReader.WaitToReadAsync(stoppingToken))
        {
            while (_channelReader.TryRead(out var item))
            {
                _logger.LogInformation("Message is `{Message}`", item.Message);
            }
        }
    }
}
```

Pretty simple, right? 

We pass a `ChannelReader<Model>` to the constructor

In the first `while`, we wait for a message to be available in the channel. When there's one, we enter the second while to read every messages available until exiting.

We could simplify this into a single while, but this version is a little more performant since it always to get multiple messages from a channel without having to `await`. More information on [An Introduction to System.Threading.Channels](https://devblogs.microsoft.com/dotnet/an-introduction-to-system-threading-channels/)

### Scoped services

Note that the `BackgroundService` here is a singleton, meaning that it won't be able to access scoped services, such as EntityFramework Core's `DbContext`, which is often registered as scope.

To get around that, you should inject an `IServiceProvider` into your background service. Depending on performance consideration, I'd typically start with creating a new scope per incoming message, and looking into batching if there's an issue.

## Wiring the dependency injection

We start by wiring the channel itself, as long as its reader and writer.

Note that I'm using minimal API syntax here, but it should be easy to adapt to the older style syntax like ASP.NET Core 5.

```csharp
builder.Services.AddSingleton(Channel.CreateUnbounded<Model>());
builder.Services.AddSingleton(p => p.GetRequiredService<Channel<Model>>().Reader);
builder.Services.AddSingleton(p => p.GetRequiredService<Channel<Model>>().Writer);
```

`CreateUnbounded` means that there won't be any item limits on the channel, other than the amount of memory available to your application.

then, we wire the service.

```csharp
builder.Services.AddHostedService<ModelService>();
```

## Setting up the route

We then setup a route to receive the messages. To make things easier, I made one that receives an array.

```csharp
app.MapPost("/messages", async (ChannelWriter<Model> writer, ICollection<Model> models, CancellationToken cancellationToken) =>
{
    foreach (var model in models)
    {
        await writer.WriteAsync(model, cancellationToken);
    }
    return StatusCodes.Status202Accepted;
});
```

## Testing it

You can try it by making a POST to `/Messages` with a payload such as 

```json
[
    { "Message": "hello world" },
    { "Message": "hello too" },
    { "Message": "hello three" }
]
```

If you want to make sure that the request returns before the messages are processed, you can add a `Task.Delay` in `ModelService` after each log message. The messages will slowly scroll in your application's console, long after the request has completed.

# Parting words

This was a very simple demo of channels with ASP.NET Core. You can find a working demo at https://github.com/EricStG/AspNetCoreChannels.

If you're considering using channels in your application, consider spending some time configuring your channels. For example, I'd look at whether I should use a bounded or unbounded channel, and if I'll be using a single reader or writer. Having it properly configured will improve performance and avoid unplanned nastiness.
