---
title: .NET isolated process guide for .NET 5.0 in Azure Functions
description: Learn how to use a .NET isolated process to run your C# functions on .NET 5.0 out-of-process in Azure.  

ms.service: azure-functions
ms.topic: conceptual 
ms.date: 03/01/2021
ms.custom: template-concept 
---

# Guide for running functions on .NET 5.0 in Azure

_.NET 5.0 support is currently in preview._

This article is an introduction to using C# to develop .NET isolated process functions, which run out-of-process in Azure Functions. Running out-of-process lets you decouple your function code from the Azure Functions runtime. It also provides a way for you to create and run functions that target the current .NET 5.0 release. 

If you don't need to support .NET 5.0 or run your functions out-of-process, you might want to instead [develop C# class library functions](functions-dotnet-class-library.md).

## Why .NET isolated process?

Previously Azure Functions has only supported a tightly integrated mode for .NET functions, which run [as a class library](functions-dotnet-class-library.md) in the same process as the host. This mode provides deep integration between the host process and the functions. For example, .NET class library functions can share binding APIs and types. However, this integration also requires a tighter coupling between the host process and the .NET function. For example, .NET functions running in-process are required to run on the same version of .NET as the Functions runtime. To enable you to run outside these constraints, you can now choose to run in an isolated process. This process isolation also lets you develop functions that use current .NET releases (such as .NET 5.0), not natively supported by the Functions runtime.

Because these functions run in a separate process, there are some [feature and functionality differences](#differences-with-net-class-library-functions) between .NET isolated function apps and .NET class library function apps.

### Benefits of running out-of-process

When running out-of-process, your .NET functions can take advantage of the following benefits: 

+ Fewer conflicts: because the functions run in a separate process, assemblies used in your app won't conflict with different version of the same assemblies used by the host process.  
+ Full control of the process: you control the start-up of the app and can control the configurations used and the middleware started.
+ Dependency injection: because you have full control of the process, you can use current .NET behaviors for dependency injection and incorporating middleware into your function app. 

## Supported versions

The only version of .NET that is currently supported to run out-of-process is .NET 5.0.

## .NET isolated project

A .NET isolated function project is basically a .NET console app project that targets .NET 5.0. The following are the basic files required in any .NET isolated project:

+ [host.json](functions-host-json.md) file.
+ [local.settings.json](functions-run-local.md#local-settings-file) file.
+ C# project file (.csproj) that defines the project and dependencies.
+ Program.cs file that's the entry point for the app.

## Package references

When running out-of-process, your .NET project uses a unique set of packages, which implement both core functionality and binding extensions. 

### Core packages 

The following packages are required to run your .NET functions in an isolated process:

+ [Microsoft.Azure.Functions.Worker](https://www.nuget.org/packages/Microsoft.Azure.Functions.Worker/)
+ [Microsoft.Azure.Functions.Worker.Sdk](https://www.nuget.org/packages/Microsoft.Azure.Functions.Worker.Sdk/)

### Extension packages

Because functions that run in a .NET isolated process use different binding types, they require a unique set of binding extension packages. 

You'll find these extension packages under [Microsoft.Azure.Functions.Worker.Extensions](https://www.nuget.org/packages?q=Microsoft.Azure.Functions.Worker.Extensions).
 
## Start-up and configuration 

When using .NET isolated functions, you have access to the start-up of your function app, which is usually in Program.cs. You're responsible for creating and starting your own host instance. As such, you also have direct access to the configuration pipeline for your app. You can much more easily inject dependencies and run middleware when running out-of-process. 

The following code shows an example of a `HostBuilder` pipeline:

:::code language="csharp" source="~/azure-functions-dotnet-worker/samples/FunctionApp/Program.cs" range="20-33":::

A `HostBuilder` is used to build and return a fully initialized `IHost` instance, which you run asynchronously to start your function app. 

:::code language="csharp" source="~/azure-functions-dotnet-worker/samples/FunctionApp/Program.cs" range="35":::

### Configuration

Having access to the host builder pipeline means that you can set any app-specific configurations during initialization. These configurations apply to your function app running in a separate process. To make changes to the functions host or trigger and binding configuration, you'll still need to use the [host.json file](functions-host-json.md).      

The following example shows how to add configuration `args`, which are read as command-line arguments: 
 
:::code language="csharp" source="~/azure-functions-dotnet-worker/samples/FunctionApp/Program.cs" range="21-24" :::

The `ConfigureAppConfiguration` method is used to configure the rest of the build process and application. This example also uses an [IConfigurationBuilder](/dotnet/api/microsoft.extensions.configuration.iconfigurationbuilder?view=dotnet-plat-ext-5.0&preserve-view=true), which makes it easier to add multiple configuration items. Because `ConfigureAppConfiguration` returns the same instance of [`IConfiguration `](/dotnet/api/microsoft.extensions.configuration.iconfiguration?view=dotnet-plat-ext-5.0&preserve-view=true), you can also just call it multiple times to add multiple configuration items. You can access the full set of configurations from both [`HostBuilderContext.Configuration`](/dotnet/api/microsoft.extensions.hosting.hostbuildercontext.configuration?view=dotnet-plat-ext-5.0&preserve-view=true) and [`IHost.Services`](/dotnet/api/microsoft.extensions.hosting.ihost.services?view=dotnet-plat-ext-5.0&preserve-view=true).

To learn more about configuration, see [Configuration in ASP.NET Core](/aspnet/core/fundamentals/configuration/?view=aspnetcore-5.0&preserve-view=true). 

### Dependency injection

Dependency injection is simplified, compared to .NET class libraries. Rather than having to create a startup class to register services, you just have to call `ConfigureServices` on the host builder and use the extension methods on [`IServiceCollection`](/dotnet/api/microsoft.extensions.dependencyinjection.iservicecollection?view=dotnet-plat-ext-5.0&preserve-view=true) to inject specific services. 

The following example injects a singleton service dependency:  
 
:::code language="csharp" source="~/azure-functions-dotnet-worker/samples/FunctionApp/Program.cs" range="29-32" :::

To learn more, see [Dependency injection in ASP.NET Core](/aspnet/core/fundamentals/dependency-injection?view=aspnetcore-5.0&preserve-view=true).

### Middleware

.NET isolated also supports middleware registration, again by using a model similar to what exists in ASP.NET. This model gives you the ability to inject logic into the invocation pipeline, and before and after functions execute.

While the full middleware registration set of APIs is not yet exposed, middleware registration is supported and we've added an example to the sample application under the Middleware folder.

:::code language="csharp" source="~/azure-functions-dotnet-worker/samples/FunctionApp/Program.cs" range="25-28" :::

## Execution context

.NET isolated passes a `FunctionContext` object to your function methods. This object lets you get an [`ILogger`](/dotnet/api/microsoft.extensions.logging.ilogger?view=dotnet-plat-ext-5.0&preserve-view=true) instance to write to the logs by calling the `GetLogger` method and supplying a `categoryName` string. To learn more, see [Logging](#logging). 

## Bindings 

Bindings are defined by using attributes on methods, parameters, and return types. A function method is a method with a `Function` and a trigger attribute applied to an input parameter, as shown in the following example:

:::code language="csharp" source="~/azure-functions-dotnet-worker/samples/SampleApp/Queue/QueueFunction.cs" range="11-14" :::

The trigger attribute specifies the trigger type and binds input data to a method parameter. The previous example function is triggered by a queue message, and the queue message is passed to the method in the `myQueueItem` parameter.

The `Function` attribute marks the method as a function entry point. The name must be unique within a project, start with a letter and only contain letters, numbers, `_`, and `-`, up to 127 characters in length. Project templates often create a method named `Run`, but the method name can be any valid C# method name.

Because .NET isolated projects run in a separate worker process, bindings can't take advantage of rich binding classes, such as `ICollector<T>`, `IAsyncCollector<T>`, and `CloudBlockBlob`. There's also no direct support for types inherited from underlying service SDKs, such as [DocumentClient](/dotnet/api/microsoft.azure.documents.client.documentclient) and [BrokeredMessage](/dotnet/api/microsoft.servicebus.messaging.brokeredmessage). Instead, bindings rely on strings, arrays, and serializable types, such as plain old class objects (POCOs). 

For HTTP triggers, you must use `HttpRequestData` and `HttpResponseData` to access the request and response data. This is because you  don't have access to the original HTTP request and response objects when running out-of-process. 

### Input bindings

A function can have zero or more input bindings that can pass data to a function. Like triggers, input bindings are defined by applying a binding attribute to an input parameter. When the function executes, the runtime tries to get data specified in the binding. The data being requested is often dependent on information provided by the trigger using binding parameters.  

### Output bindings

To write to an output binding, you must apply an output binding attribute to the function method, which defined how to write to the bound service. The value returned by the method is written to the output binding. For example, the following example writes a string value to a message queue named `functiontesting2` by using an output binding:

:::code language="csharp" source="~/azure-functions-dotnet-worker/samples/SampleApp/Queue/QueueFunction.cs" range="11-21" :::

### Multiple output bindings

The data written to an output binding is always the return value of the function. If you need to write to more than one output binding, you must create a custom return type. This return type must have the output binding attribute applied to one or more properties of the class. The following example writes to both an HTTP response and a queue output binding:

:::code language="csharp" source="~/azure-functions-dotnet-worker/samples/FunctionApp/Function1/Function1.cs" range="14-33":::

### HTTP trigger

HTTP triggers translates the incoming HTTP request message into an `HttpRequestData` object that is passed to the function. This object provides data from the request, including `Headers`, `Cookies`, `Identities`, `URL`, and optional a message `Body`. This object is a representation of the HTTP request object and not the request itself. 

Likewise, the function returns an `HttpReponseData` object, which provides data used to create the HTTP response, including message `StatusCode`, `Headers`, and optionally a message `Body`.  

The following code is an HTTP trigger 

:::code language="csharp" source="~/azure-functions-dotnet-worker/samples/SampleApp/Http/HttpFunction.cs" range="13-27" :::

## Logging

In .NET isolated, you can write to logs by using an [`ILogger`](/dotnet/api/microsoft.extensions.logging.ilogger?view=dotnet-plat-ext-5.0&preserve-view=true) instance obtained from a `FunctionContext` object passed to your function. Call the `GetLogger` method, passing a string value that is the name for the category in which the logs are written. The category is usually the name of the specific function from which the logs are written. To learn more about categories, see the [monitoring article](functions-monitoring.md#log-levels-and-categories). 

The following example shows how to get an `ILogger` and write logs inside a function:

:::code language="csharp" source="~/azure-functions-dotnet-worker/samples/SampleApp/Http/HttpFunction.cs" range="17-18" ::: 

Use various methods of `ILogger` to write various log levels, such as `LogWarning` or `LogError`. To learn more about log levels, see the [monitoring article](functions-monitoring.md#log-levels-and-categories).

An [`ILogger`](/dotnet/api/microsoft.extensions.logging.ilogger?view=dotnet-plat-ext-5.0&preserve-view=true) is also provided when using [dependency injection](#dependency-injection).

## Differences with .NET class library functions

This section describes the current state of the functional and behavioral differences running on .NET 5.0 out-of-process compared to .NET class library functions running in-process:

| Feature/behavior |  In-process (.NET Core 3.1) | Out-of-process (.NET 5.0) |
| ---- | ---- | ---- |
| .NET versions | LTS (.NET Core 3.1) | Current (.NET 5.0) |
| Core packages | [Microsoft.NET.Sdk.Functions](https://www.nuget.org/packages/Microsoft.NET.Sdk.Functions/) | [Microsoft.Azure.Functions.Worker](https://www.nuget.org/packages/Microsoft.Azure.Functions.Worker/)<br/>[Microsoft.Azure.Functions.Worker](https://www.nuget.org/packages/Microsoft.Azure.Functions.Worker.Sdk) | 
| Binding extension packages | [`Microsoft.Azure.WebJobs.Extensions.*`](https://www.nuget.org/packages?q=Microsoft.Azure.WebJobs.Extensions)  | Under [`Microsoft.Azure.Functions.Worker.Extensions.*`](https://www.nuget.org/packages?q=Microsoft.Azure.Functions.Worker.Extensions) | 
| Logging | [`ILogger`](/dotnet/api/microsoft.extensions.logging.ilogger?view=dotnet-plat-ext-5.0&preserve-view=true) passed to the function | [`ILogger`](/dotnet/api/microsoft.extensions.logging.ilogger?view=dotnet-plat-ext-5.0&preserve-view=true) obtained from `FunctionContext` |
| Cancellation tokens | [Supported](functions-dotnet-class-library.md#cancellation-tokens) | Not supported |
| Output bindings | Out parameters | Return values |
| Output binding types |  `IAsyncCollector`, [DocumentClient](/dotnet/api/microsoft.azure.documents.client.documentclient), [BrokeredMessage](/dotnet/api/microsoft.servicebus.messaging.brokeredmessage), and other client-specific types | Simple types, JSON serializable types, and arrays. |
| Multiple output bindings | Supported | [Supported](#multiple-output-bindings) |
| HTTP trigger | [`HttpRequest`](/dotnet/api/microsoft.aspnetcore.http.httprequest?view=aspnetcore-5.0&preserve-view=true)/[`ObjectResult`](/dotnet/api/microsoft.aspnetcore.mvc.objectresult?view=aspnetcore-5.0&preserve-view=true) | `HttpRequestData`/`HttpResponseData` |
| Durable Functions | [Supported](durable/durable-functions-overview.md) | Not supported | 
| Imperative bindings | [Supported](functions-dotnet-class-library.md#binding-at-runtime) | Not supported |
| function.json artifact | Generated | Not generated |
| Configuration | [host.json](functions-host-json.md) | [host.json](functions-host-json.md) and [custom initialization](#configuration) |
| Dependency injection | [Supported](functions-dotnet-dependency-injection.md)  | [Supported](#dependency-injection) |
| Middleware | Not supported | [Supported](#middleware) |
| Cold start times | Typical | Longer, because of just-in-time start-up. Run on Linux instead of Windows to reduce potential delays. |
| ReadyToRun | [Supported](functions-dotnet-class-library.md#readytorun) | _TBD_ |


## Next steps

+ [Learn more about triggers and bindings](functions-triggers-bindings.md)
+ [Learn more about best practices for Azure Functions](functions-best-practices.md)
