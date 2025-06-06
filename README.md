[![GitHub Workflow Status](https://img.shields.io/github/actions/workflow/status/martinothamar/Mediator/build.yml?branch=main)](https://github.com/martinothamar/Mediator/actions)
[![GitHub](https://img.shields.io/github/license/martinothamar/Mediator?style=flat-square)](https://github.com/martinothamar/Mediator/blob/main/LICENSE)
[![Downloads](https://img.shields.io/nuget/dt/mediator.abstractions?style=flat-square)](https://www.nuget.org/packages/Mediator.Abstractions/)<br/>
[![Abstractions NuGet current](https://img.shields.io/nuget/v/Mediator.Abstractions?label=Mediator.Abstractions)](https://www.nuget.org/packages/Mediator.Abstractions)
[![SourceGenerator NuGet current](https://img.shields.io/nuget/v/Mediator.SourceGenerator?label=Mediator.SourceGenerator)](https://www.nuget.org/packages/Mediator.SourceGenerator)<br/>
[![Abstractions NuGet prerelease](https://img.shields.io/nuget/vpre/Mediator.Abstractions?label=Mediator.Abstractions)](https://www.nuget.org/packages/Mediator.Abstractions)
[![SourceGenerator NuGet prerelease](https://img.shields.io/nuget/vpre/Mediator.SourceGenerator?label=Mediator.SourceGenerator)](https://www.nuget.org/packages/Mediator.SourceGenerator)<br/>

# Mediator

> **Note**
>
> **Version 3.0** is currently being finalized. It is recommended to start using v3 prereleases as opposed to v2.1. See status and provide feedback [here (#98)](https://github.com/martinothamar/Mediator/issues/98)

This is a high performance .NET implementation of the Mediator pattern using the [source generators](https://devblogs.microsoft.com/dotnet/introducing-c-source-generators/) feature introduced in .NET 5.
The API and usage is mostly based on the great [MediatR](https://github.com/jbogard/MediatR) library, with some deviations to allow for better performance.
Packages are .NET Standard 2.0 compatible.

The mediator pattern is great for implementing cross cutting concern (logging, metrics, etc) and avoiding "fat" constructors due to lots of injected services.

Goals for this library
* High performance
  * Runtime performance can be the same for both runtime reflection and source generator based approaches, but it's easier to optimize in the latter case
* AOT friendly
  * MS are investing time in various AOT scenarios, and for example iOS requires AOT compilation
* Build time errors instead of runtime errors
  * The generator includes diagnostics, i.e. if a handler is not defined for a request, a warning is emitted

In particular, source generators in this library is used to
* Generate code for DI registration
* Generate code for `IMediator` implementation
  * Request/Command/Query `Send` methods are monomorphized (1 method per T), the generic `ISender.Send` methods rely on these
  * You can use both `IMediator` and `Mediator`, the latter allows for better performance
* Generate diagnostics related messages and message handlers

NuGet packages:

```pwsh
dotnet add package Mediator.SourceGenerator --version 3.0.*-*
dotnet add package Mediator.Abstractions --version 3.0.*-*
```
or
```xml
<PackageReference Include="Mediator.SourceGenerator" Version="3.0.*-*">
  <PrivateAssets>all</PrivateAssets>
  <IncludeAssets>runtime; build; native; contentfiles; analyzers</IncludeAssets>
</PackageReference>
<PackageReference Include="Mediator.Abstractions" Version="3.0.*-*" />
```

See this great video by [@Elfocrash / Nick Chapsas](https://github.com/Elfocrash), covering both similarities and differences between Mediator and MediatR

[![Using MediatR in .NET? Maybe replace it with this](https://img.youtube.com/vi/aaFLtcf8cO4/0.jpg)](https://www.youtube.com/watch?v=aaFLtcf8cO4)

## Table of Contents

- [Mediator](#mediator)
  - [Table of Contents](#table-of-contents)
  - [2. Benchmarks](#2-benchmarks)
  - [3. Usage and abstractions](#3-usage-and-abstractions)
    - [3.1. Message types](#31-message-types)
    - [3.2. Handler types](#32-handler-types)
    - [3.3. Pipeline types](#33-pipeline-types)
      - [3.3.1. Message validation example](#331-message-validation-example)
      - [3.3.2. Error logging example](#332-error-logging-example)
    - [3.4. Configuration](#34-configuration)
  - [4. Getting started](#4-getting-started)
    - [4.1. Add packages](#41-add-packages)
    - [4.2. Add Mediator to DI container](#42-add-mediator-to-di-container)
    - [4.3. Create `IRequest<>` type](#43-create-irequest-type)
    - [4.4. Use pipeline behaviors](#44-use-pipeline-behaviors)
    - [4.5. Constrain `IPipelineBehavior<,>` message with open generics](#45-constrain-ipipelinebehavior-message-with-open-generics)
    - [4.6. Use notifications](#46-use-notifications)
    - [4.7. Polymorphic dispatch with notification handlers](#47-polymorphic-dispatch-with-notification-handlers)
    - [4.8. Notification handlers also support open generics](#48-notification-handlers-also-support-open-generics)
    - [4.9. Notification publishers](#49-notification-publishers)
    - [4.10. Use streaming messages](#410-use-streaming-messages)
  - [5. Diagnostics](#5-diagnostics)
  - [6. Differences from MediatR](#6-differences-from-mediatr)
  - [7. Versioning](#7-versioning)

## 2. Benchmarks

Here is a brief comparison benchmark hightighting the difference between MediatR and this library using `Singleton` lifetime.
Note that this library yields the best performance when using the `Singleton` service lifetime.

* `<ColdStart | Notification | Request | StreamRequest>_Mediator`: the concrete `Mediator` class generated by this library
* `<ColdStart | Notification | Request | StreamRequest>_IMediator`: call through the `IMediator` interface in this library
* `<ColdStart | Notification | Request | StreamRequest>_MediatR`: the [MediatR](https://github.com/jbogard/MediatR) library

Benchmark category descriptions
* `ColdStart` - time to resolve `IMediator` from `IServiceProvider` and send a single request
* `Notification` - publish a single notification
* `Request` - publish a single request
* `StreamRequest` - stream a single request which yields 3 responses without delay

See the [benchmarks/ folder](/benchmarks/README.md) for more detailed information, including varying lifetimes and project sizes.

![Benchmarks](/img/benchmarks.png "Benchmarks")

## 3. Usage and abstractions

There are two NuGet packages needed to use this library
* Mediator.SourceGenerator
  * To generate the `IMediator` implementation and dependency injection setup.
* Mediator.Abstractions
  * Message types (`IRequest<,>`, `INotification`), handler types (`IRequestHandler<,>`, `INotificationHandler<>`), pipeline types (`IPipelineBehavior`)

You install the source generator package into your edge/outermost project (i.e. ASP.NET Core application, Background worker project),
and then use the `Mediator.Abstractions` package wherever you define message types and handlers.
Standard message handlers are automatically picked up and added to the DI container in the generated `AddMediator` method.
*Pipeline behaviors need to be added manually (including pre/post/exception behaviors).*

For example implementations, see the [/samples](/samples) folder.
See the [ASP.NET Core clean architecture sample](/samples/apps/ASPNET_Core_CleanArchitecture) for a more real world setup.

### 3.1. Message types

* `IMessage` - marker interface
* `IStreamMessage` - marker interface
* `IBaseRequest` - marker interface for requests
* `IRequest` - a request message, no return value (`ValueTask<Unit>`)
* `IRequest<out TResponse>` - a request message with a response (`ValueTask<TResponse>`)
* `IStreamRequest<out TResponse>` - a request message with a streaming response (`IAsyncEnumerable<TResponse>`)
* `IBaseCommand` - marker interface for commands
* `ICommand` - a command message, no return value (`ValueTask<Unit>`)
* `ICommand<out TResponse>` - a command message with a response (`ValueTask<TResponse>`)
* `IStreamCommand<out TResponse>` - a command message with a streaming response (`IAsyncEnumerable<TResponse>`)
* `IBaseQuery` - marker interface for queries
* `IQuery<out TResponse>` - a query message with a response (`ValueTask<TResponse>`)
* `IStreamQuery<out TResponse>` - a query message with a streaming response (`IAsyncEnumerable<TResponse>`)
* `INotification` - a notification message, no return value (`ValueTask`)

As you can see, you can achieve the exact same thing with requests, commands and queries. But I find the distinction in naming useful if you for example use the CQRS pattern or for some reason have a preference on naming in your application.

### 3.2. Handler types

* `IRequestHandler<in TRequest>`
* `IRequestHandler<in TRequest, TResponse>`
* `IStreamRequestHandler<in TRequest, out TResponse>`
* `ICommandHandler<in TCommand>`
* `ICommandHandler<in TCommand, TResponse>`
* `IStreamCommandHandler<in TCommand, out TResponse>`
* `IQueryHandler<in TQuery, TResponse>`
* `IStreamQueryHandler<in TQuery, out TResponse>`
* `INotificationHandler<in TNotification>`

These types are used in correlation with the message types above.

### 3.3. Pipeline types

* `IPipelineBehavior<TMessage, TResponse>`
* `IStreamPipelineBehavior<TMessage, TResponse>`
* `MessagePreProcessor<TMessage, TResponse>`
* `MessagePostProcessor<TMessage, TResponse>`
* `MessageExceptionHandler<TMessage, TResponse, TException>`

#### 3.3.1. Message validation example

```csharp
// As a normal pipeline behavior
public sealed class MessageValidatorBehaviour<TMessage, TResponse> : IPipelineBehavior<TMessage, TResponse>
    where TMessage : IValidate
{
    public ValueTask<TResponse> Handle(
        TMessage message,
        CancellationToken cancellationToken,
        MessageHandlerDelegate<TMessage, TResponse> next
    )
    {
        if (!message.IsValid(out var validationError))
            throw new ValidationException(validationError);

        return next(message, cancellationToken);
    }
}

// Or as a pre-processor
public sealed class MessageValidatorBehaviour<TMessage, TResponse> : MessagePreProcessor<TMessage, TResponse>
    where TMessage : IValidate
{
    protected override ValueTask Handle(TMessage message, CancellationToken cancellationToken)
    {
        if (!message.IsValid(out var validationError))
            throw new ValidationException(validationError);

        return default;
    }
}

// Register as IPipelineBehavior<,> in either case
services.AddSingleton(typeof(IPipelineBehavior<,>), typeof(MessageValidatorBehaviour<,>))
```

#### 3.3.2. Error logging example

```csharp
// As a normal pipeline behavior
public sealed class ErrorLoggingBehaviour<TMessage, TResponse> : IPipelineBehavior<TMessage, TResponse>
    where TMessage : IMessage
{
    private readonly ILogger<ErrorLoggingBehaviour<TMessage, TResponse>> _logger;

    public ErrorLoggingBehaviour(ILogger<ErrorLoggingBehaviour<TMessage, TResponse>> logger)
    {
        _logger = logger;
    }

    public async ValueTask<TResponse> Handle(
        TMessage message,
        CancellationToken cancellationToken,
        MessageHandlerDelegate<TMessage, TResponse> next
    )
    {
        try
        {
            return await next(message, cancellationToken);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error handling message of type {messageType}", message.GetType().Name);
            throw;
        }
    }
}

// Or as an exception handler
public sealed class ErrorLoggingBehaviour<TMessage, TResponse> : MessageExceptionHandler<TMessage, TResponse>
    where TMessage : notnull, IMessage
{
    private readonly ILogger<ErrorLoggingBehaviour<TMessage, TResponse>> _logger;

    public ErrorLoggingBehaviour(ILogger<ErrorLoggingBehaviour<TMessage, TResponse>> logger)
    {
        _logger = logger;
    }

    protected override ValueTask<ExceptionHandlingResult<TResponse>> Handle(
        TMessage message,
        Exception exception,
        CancellationToken cancellationToken
    )
    {
        _logger.LogError(exception, "Error handling message of type {messageType}", message.GetType().Name);
        // Let the exception bubble up by using the base class helper NotHandled:
        return NotHandled;
        // Or if the exception is properly handled, you can just return your own response,
        // using the base class helper Handle().
        // This requires you to know something about TResponse,
        // so TResponse needs to be constrained to something,
        // typically with a static abstract member acting as a consructor on an interface or abstract class.
        return Handled(null!);
    }
}

// Register as IPipelineBehavior<,> in either case
services.AddSingleton(typeof(IPipelineBehavior<,>), typeof(ErrorLoggingBehaviour<,>))
```

### 3.4. Configuration

There are two ways to configure Mediator. Configuration values are needed during compile-time since this is a source generator:
* Assembly level attribute for configuration: `MediatorOptionsAttribute`
* Options configuration delegate in `AddMediator` function.

```csharp
services.AddMediator((MediatorOptions options) =>
{
    options.Namespace = "SimpleConsole.Mediator";
    options.ServiceLifetime = ServiceLifetime.Transient;
});

// or

[assembly: MediatorOptions(Namespace = "SimpleConsole.Mediator", ServiceLifetime = ServiceLifetime.Transient)]
```

* `Namespace` - where the `IMediator` implementation is generated
* `ServiceLifetime` - the DI service lifetime
  * `Singleton` - (default value) everything registered as singletons, minimal allocations
  * `Transient` - mediator and handlers registered as transient
  * `Scoped`    - mediator and handlers registered as scoped

Singleton lifetime is highly recommended as it yields the best performance.
Every application is different, but it is likely that a lot of your message handlers doesn't keep state and have no need for transient or scoped lifetime.
In a lot of cases those lifetimes only allocate lots of memory for no particular reason.

## 4. Getting started

In this section we will get started with Mediator and go through a sample
illustrating the various ways the Mediator pattern can be used in an application.

See the full runnable sample code in the [Showcase sample](/samples/Showcase/).

### 4.1. Add packages

```pwsh
dotnet add package Mediator.SourceGenerator --version 3.0.*-*
dotnet add package Mediator.Abstractions --version 3.0.*-*
```
or
```xml
<PackageReference Include="Mediator.SourceGenerator" Version="3.0.*-*">
  <PrivateAssets>all</PrivateAssets>
  <IncludeAssets>runtime; build; native; contentfiles; analyzers</IncludeAssets>
</PackageReference>
<PackageReference Include="Mediator.Abstractions" Version="3.0.*-*" />
```

### 4.2. Add Mediator to DI container

In `ConfigureServices` or equivalent, call `AddMediator` (unless `MediatorOptions` is configured, default namespace is `Mediator`).
This registers your handler below.

```csharp
using Mediator;
using Microsoft.Extensions.DependencyInjection;
using System;

var services = new ServiceCollection(); // Most likely IServiceCollection comes from IHostBuilder/Generic host abstraction in Microsoft.Extensions.Hosting

services.AddMediator();
using var serviceProvider = services.BuildServiceProvider();
```

### 4.3. Create `IRequest<>` type

```csharp
var mediator = serviceProvider.GetRequiredService<IMediator>();
var ping = new Ping(Guid.NewGuid());
var pong = await mediator.Send(ping);
Debug.Assert(ping.Id == pong.Id);

// ...

public sealed record Ping(Guid Id) : IRequest<Pong>;

public sealed record Pong(Guid Id);

public sealed class PingHandler : IRequestHandler<Ping, Pong>
{
    public ValueTask<Pong> Handle(Ping request, CancellationToken cancellationToken)
    {
        return new ValueTask<Pong>(new Pong(request.Id));
    }
}
```

As soon as you code up message types, the source generator will add DI registrations automatically (inside `AddMediator`).
P.S - You can inspect the code yourself - open `Mediator.g.cs` in VS from Project -> Dependencies -> Analyzers -> Mediator.SourceGenerator -> Mediator.SourceGenerator.MediatorGenerator,
or just F12 through the code.

### 4.4. Use pipeline behaviors

The pipeline behavior below validates all incoming `Ping` messages.
Pipeline behaviors currently must be added manually.

```csharp
services.AddMediator();
services.AddSingleton<IPipelineBehavior<Ping, Pong>, PingValidator>();

public sealed class PingValidator : IPipelineBehavior<Ping, Pong>
{
    public ValueTask<Pong> Handle(Ping request, MessageHandlerDelegate<Ping, Pong> next, CancellationToken cancellationToken)
    {
        if (request is null || request.Id == default)
            throw new ArgumentException("Invalid input");

        return next(request, cancellationToken);
    }
}
```

### 4.5. Constrain `IPipelineBehavior<,>` message with open generics

Add open generic handler to process all or a subset of messages passing through Mediator.
This handler will log any error that is thrown from message handlers (`IRequest`, `ICommand`, `IQuery`).
It also publishes a notification allowing notification handlers to react to errors.
Message pre- and post-processors along with the exception handlers can also constrain the generic type parameters in the same way.

```csharp
services.AddMediator();
services.AddSingleton(typeof(IPipelineBehavior<,>), typeof(ErrorLoggerHandler<,>));

public sealed record ErrorMessage(Exception Exception) : INotification;
public sealed record SuccessfulMessage() : INotification;

public sealed class ErrorLoggerHandler<TMessage, TResponse> : IPipelineBehavior<TMessage, TResponse>
    where TMessage : IMessage // Constrained to IMessage, or constrain to IBaseCommand or any custom interface you've implemented
{
    private readonly ILogger<ErrorLoggerHandler<TMessage, TResponse>> _logger;
    private readonly IMediator _mediator;

    public ErrorLoggerHandler(ILogger<ErrorLoggerHandler<TMessage, TResponse>> logger, IMediator mediator)
    {
        _logger = logger;
        _mediator = mediator;
    }

    public async ValueTask<TResponse> Handle(TMessage message, MessageHandlerDelegate<TMessage, TResponse> next, CancellationToken cancellationToken)
    {
        try
        {
            var response = await next(message, cancellationToken);
            return response;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error handling message");
            await _mediator.Publish(new ErrorMessage(ex));
            throw;
        }
    }
}
```

### 4.6. Use notifications

We can define a notification handler to catch errors from the above pipeline behavior.

```csharp
// Notification handlers are automatically added to DI container

public sealed class ErrorNotificationHandler : INotificationHandler<ErrorMessage>
{
    public ValueTask Handle(ErrorMessage error, CancellationToken cancellationToken)
    {
        // Could log to application insights or something...
        return default;
    }
}
```

### 4.7. Polymorphic dispatch with notification handlers

We can also define a notification handler that receives all notifications.

```csharp

public sealed class StatsNotificationHandler : INotificationHandler<INotification> // or any other interface deriving from INotification
{
    private long _messageCount;
    private long _messageErrorCount;

    public (long MessageCount, long MessageErrorCount) Stats => (_messageCount, _messageErrorCount);

    public ValueTask Handle(INotification notification, CancellationToken cancellationToken)
    {
        Interlocked.Increment(ref _messageCount);
        if (notification is ErrorMessage)
            Interlocked.Increment(ref _messageErrorCount);
        return default;
    }
}
```

### 4.8. Notification handlers also support open generics

```csharp
public sealed class GenericNotificationHandler<TNotification> : INotificationHandler<TNotification>
    where TNotification : INotification // Generic notification handlers will be registered as open constrained types automatically
{
    public ValueTask Handle(TNotification notification, CancellationToken cancellationToken)
    {
        return default;
    }
}
```

### 4.9. Notification publishers

> [!IMPORTANT]
> This API is currently in the latest 3.0 previews, not available in stable.

Notification publishers are responsible for dispatching notifications to a collection of handlers.
There are two built in implementations:

* `ForeachAwaitPublisher` - the default, dispatches the notifications to handlers in order 1-by-1
* `TaskWhenAllPublisher` - dispatches notifications in parallel

Both of these try to be efficient by handling a number of special cases (early exit on sync completion, single-handler, array of handlers).
Below we implement a custom one by simply using `Task.WhenAll`.

```csharp
services.AddMediator((MediatorOptions options) =>
{
    options.NotificationPublisherType = typeof(FireAndForgetNotificationPublisher);
});

public sealed class FireAndForgetNotificationPublisher : INotificationPublisher
{
    public async ValueTask Publish<TNotification>(
        NotificationHandlers<TNotification> handlers,
        TNotification notification,
        CancellationToken cancellationToken
    )
        where TNotification : INotification
    {
        try
        {
            await Task.WhenAll(handlers.Select(handler => handler.Handle(notification, cancellationToken).AsTask()));
        }
        catch (Exception ex)
        {
            // Notifications should be fire-and-forget, we just need to log it!
            // This way we don't have to worry about exceptions bubbling up when publishing notifications
            Console.Error.WriteLine(ex);

            // NOTE: not necessarily saying this is a good idea!
        }
    }
}
```


### 4.10. Use streaming messages

Since version 1.* of this library there is support for streaming using `IAsyncEnumerable`.

```csharp
var mediator = serviceProvider.GetRequiredService<IMediator>();

var ping = new StreamPing(Guid.NewGuid());

await foreach (var pong in mediator.CreateStream(ping))
{
    Debug.Assert(ping.Id == pong.Id);
    Console.WriteLine("Received pong!"); // Should log 5 times
}

// ...

public sealed record StreamPing(Guid Id) : IStreamRequest<Pong>;

public sealed record Pong(Guid Id);

public sealed class PingHandler : IStreamRequestHandler<StreamPing, Pong>
{
    public async IAsyncEnumerable<Pong> Handle(StreamPing request, [EnumeratorCancellation] CancellationToken cancellationToken)
    {
        for (int i = 0; i < 5; i++)
        {
            await Task.Delay(1000, cancellationToken);
            yield return new Pong(request.Id);
        }
    }
}
```

## 5. Diagnostics

Since this is a source generator, diagnostics are also included. Examples below

* Missing request handler

![Missing request handler](/img/missing_request_handler.png "Missing request handler")

* Multiple request handlers found

![Multiple request handlers found](/img/multiple_request_handlers.png "Multiple request handlers found")


## 6. Differences from [MediatR](https://github.com/jbogard/MediatR)

This is a work in progress list on the differences between this library and MediatR.

* `RequestHandlerDelegate<TResponse>()` -> `MessageHandlerDelegate<TMessage, TResponse>(TMessage message, CancellationToken cancellationToken)`
  * This is to avoid excessive closure allocations. I think it's worthwhile when the cost is simply passing along the message and the cancellation token.
* No `ServiceFactory`
  * This library relies on the `Microsoft.Extensions.DependencyInjection`, so it only works with DI containers that integrate with those abstractions.
* Singleton service lifetime by default
  * MediatR in combination with `MediatR.Extensions.Microsoft.DependencyInjection` does transient service registration by default, which leads to a lot of allocations. Even if it is configured for singleton lifetime, `IMediator` and `ServiceFactory` services are registered as transient (not configurable).
* Methods return `ValueTask<T>` instead of `Task<T>`, to allow for fewer allocations (for example if the handler completes synchronously, or using async method builder pooling/`PoolingAsyncValueTaskMethodBuilder<T>`)
* This library doesn't support generic requests/notifications

## 7. Versioning

For versioning this library I try to follow [semver 2.0](https://semver.org/) as best as I can, meaning

* Major bump for breaking changes
* Minor bump for new backward compatible features
* Patch bump for bugfixes
