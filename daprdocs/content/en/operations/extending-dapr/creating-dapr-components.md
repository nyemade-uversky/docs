---
type: docs
title: "Creating external gRPC Components"
linkTitle: "Developing gRPC Components"
weight: 250
description: "Extend dapr with external gRPC-based components"
---

Pluggable Components are [gRPC-based](https://grpc.io/) dapr components generally running as containers or processes communicating via [Unix Domain Sockets](https://en.wikipedia.org/wiki/Unix_domain_socket).

## Declaring

Pluggable Componets are defined using the [Component CRD]({{< ref component-schema.md >}}) when no built-in components are found with the specified `type`.

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: prod-mystore
spec:
  type: state.custom-store
  version: v1
```

## Developing a Pluggable Component from scratch

{{< tabs ".NET" >}}
{{% codetab %}}

### Step 1: Prerequisites

For this tutorial we assume that you have minimal knowledge of [gRPC and protocol buffers](https://grpc.io/), and that you choose a programming language [that supports gRPC](https://grpc.io/docs/languages/).

For simplicity, all code samples will use the generated code from the [protoc](https://developers.google.com/protocol-buffers/docs/downloads) tool.

As prior mentioned, dapr uses a Unix Domain Socket to communicate with Pluggable Components, which means that as a prerequisite you also need a UNIX-like system, or for windows users [WSL](https://learn.microsoft.com/en-us/windows/wsl/install) should be sufficient.

And also [.NET Core 6+](https://dotnet.microsoft.com/en-us/download)

This tutorial is based on the [official microsoft documentation](https://learn.microsoft.com/en-us/aspnet/core/grpc/basics?view=aspnetcore-6.0#c-tooling-support-for-proto-files).

### Step 2: Generating gRPC code for the target language

The protobuf definitions for Pluggable Components reside in the [dapr](https://github.com/dapr/dapr) runtime repository, you must clone it first to get a copy of proto definitions in your machine.

```shell
git clone https://github.com/dapr/dapr.git
```

> You can also [download as a zip file](https://github.com/dapr/dapr/archive/refs/heads/master.zip), and extract using the [unzip](https://linux.die.net/man/1/unzip) command.
Once you have the proto files in your machine, go to the repository root folder, copy out the proto directory [dapr/proto/components/v1](https://github.com/dapr/dapr/tree/master/dapr/proto/components/v1).

> Note that you only need the `components/v1` folder leave the rest untouched.
### Step 3: Prepare the environment

You can choose either, [StateStore](https://github.com/dapr/dapr/blob/9da16f4a2ee9658dc7b26cf154d75c2e7a3128b7/dapr/proto/components/v1/state.proto#L119), [PubSub](https://github.com/dapr/dapr/blob/9da16f4a2ee9658dc7b26cf154d75c2e7a3128b7/dapr/proto/components/v1/pubsub.proto#L23) or [Bindings](https://github.com/dapr/dapr/blob/master/dapr/proto/components/v1/bindings.proto#L1) to implement your component, pick one of them and implement the desired service using your desired language.

Start with an empty folder and create a new .NET gRPC service by running:

```shell
dotnet new grpc -o DaprMemStoreComponent
```

Now our directory tree should be something like this:

```shell
➜  memstore-dotnet tree .
.
└── DaprMemStoreComponent
    ├── DaprMemStoreComponent.csproj
    ├── Program.cs
    ├── Properties
    │   └── launchSettings.json
    ├── Protos
    │   └── greet.proto
    ├── Services
    │   └── GreeterService.cs
    ├── appsettings.Development.json
    ├── appsettings.json
    └── obj
        ├── DaprMemStoreComponent.csproj.nuget.dgspec.json
        ├── DaprMemStoreComponent.csproj.nuget.g.props
        ├── DaprMemStoreComponent.csproj.nuget.g.targets
        ├── project.assets.json
        └── project.nuget.cache
5 directories, 12 files
```

Remove the file `Protos/greet.proto` and copy out the `dapr/proto/components/v1` folder from dapr repository to the `DaprMemStoreComponent/Protos` directory.

```shell
➜  memstore-dotnet tree .
.
└── DaprMemStoreComponent
    ├── DaprMemStoreComponent.csproj
    ├── Program.cs
    ├── Properties
    │   └── launchSettings.json
    ├── Protos
    │   └── dapr
    │       └── proto
    │           └── components
    │               └── v1
    │                   ├── bindings.proto
    │                   ├── common.proto
    │                   ├── pubsub.proto
    │                   └── state.proto
    ├── Services
    │   └── GreeterService.cs
    ├── appsettings.Development.json
    ├── appsettings.json
    └── obj
        ├── DaprMemStoreComponent.csproj.nuget.dgspec.json
        ├── DaprMemStoreComponent.csproj.nuget.g.props
        ├── DaprMemStoreComponent.csproj.nuget.g.targets
        ├── project.assets.json
        └── project.nuget.cache
9 directories, 16 files
```

Now, open the project file `DaprMemStoreComponent.csproj`, and find the live above

```xml
    <Protobuf Include="Protos\greet.proto" GrpcServices="Server" />
```

Remove that line, and add the following ones:

```xml
    <Protobuf Include="Protos\dapr\proto\components\v1\bindings.proto" ProtoRoot="Protos" GrpcServices="Client,Server" />
    <Protobuf Include="Protos\dapr\proto\components\v1\pubsub.proto" ProtoRoot="Protos" GrpcServices="Client,Server" />
    <Protobuf Include="Protos\dapr\proto\components\v1\state.proto" ProtoRoot="Protos" GrpcServices="Client,Server" />
    <Protobuf Include="Protos\dapr\proto\components\v1\common.proto" ProtoRoot="Protos" GrpcServices="Client" />
```

Now let's create our `MemStoreService`. First, remove the `GreeterService.cs` and then create our `MemStoreService.cs` with the following content,

```csharp
using Dapr.Proto.Components.V1;
namespace DaprMemStoreComponent.Services;
public class MemStoreService : StateStore.StateStoreBase
{
    private readonly ILogger<MemStoreService> _logger;
    public MemStoreService(ILogger<MemStoreService> logger)
    {
        _logger = logger;
    }
}
```

Now, go to the `Program.cs` and find the line `app.MapGrpcService<GreeterService>();` and replace `GreeterService` by our `MemStoreService`, the final result will be,

```csharp
using DaprMemStoreComponent.Services;
using Microsoft.AspNetCore.Server.Kestrel.Core;
var builder = WebApplication.CreateBuilder(args);
// Additional configuration is required to successfully run gRPC on macOS.
// For instructions on how to configure Kestrel and gRPC clients on macOS, visit https://go.microsoft.com/fwlink/?linkid=2099682
// Add services to the container.
builder.WebHost.ConfigureKestrel(options =>
            {
                options.ListenLocalhost(5000, o => o.Protocols = HttpProtocols.Http2); // necessary for mac users.
            });
builder.Services.AddGrpc();
var app = builder.Build();
// Configure the HTTP request pipeline.
app.MapGrpcService<MemStoreService>();
app.MapGet("/", () => "Communication with gRPC endpoints must be made through a gRPC client. To learn how to create a client, visit: https://go.microsoft.com/fwlink/?linkid=2086909");
app.Run();
```

At this point we are ready to build and run our service, let's test it by running `dotnet build`

```shell
➜  DaprMemStoreComponent dotnet build
[redacted]
Build succeeded.
    0 Warning(s)
    0 Error(s)
Time Elapsed 00:00:04.94
```

Run the state store service by running `dotnet run`

```shell
➜  DaprMemStoreComponent dotnet run
Building...
warn: Microsoft.AspNetCore.Server.Kestrel[0]
      Overriding address(es) 'http://localhost:5259, https://localhost:7089'. Binding to endpoints defined via IConfiguration and/or UseKestrel() instead.
info: Microsoft.Hosting.Lifetime[14]
      Now listening on: http://localhost:5000
info: Microsoft.Hosting.Lifetime[0]
      Application started. Press Ctrl+C to shut down.
info: Microsoft.Hosting.Lifetime[0]
      Hosting environment: Development
info: Microsoft.Hosting.Lifetime[0]
```

To test that our service are running I'll use the [grpcurl](https://github.com/fullstorydev/grpcurl) library, you can download and use it or use your preferred tool.

Let's call the `Features` method from StateStore service, within the `DaprMemStoreComponent` folder run the following command: `grpcurl --plaintext --import-path "$PWD/Protos" --proto dapr/proto/components/v1/state.proto localhost:5000 dapr.proto.components.v1.StateStore/Features`

```shell
➜  DaprMemStoreComponent grpcurl --plaintext --import-path "$PWD/Protos" --proto dapr/proto/components/v1/state.proto localhost:5000 dapr.proto.components.v1.StateStore/Features
ERROR:
  Code: Unimplemented
  Message:
```

If you see a `Unimplemented` is because it is working!

### Step 4: Unix Socket and gRPC Reflection

As prior mentioned, gRPC components are backed by an Unix Domain Socket, you must have noticed that our service is not using an unix socket, instead we are running via HTTP2. Let's change that.

Open the `Program.cs` file and add Kestrel configuration to listen to a socket, for now we are going use the `/tmp/` directory for simplicity.

```csharp
using DaprMemStoreComponent.Services;
var socket = "/tmp/custom-store.sock";
if (File.Exists(socket))
{
    Console.WriteLine("Removing existing socket"); // cleanup step.
    File.Delete(socket);
}
var builder = WebApplication.CreateBuilder(args);
// Additional configuration is required to successfully run gRPC on macOS.
// For instructions on how to configure Kestrel and gRPC clients on macOS, visit https://go.microsoft.com/fwlink/?linkid=2099682
// Add services to the container.
builder.WebHost.ConfigureKestrel(options =>
            {
                options.ListenUnixSocket(socket); // listen to the unix socket
            });
builder.Services.AddGrpc();
var app = builder.Build();
// Configure the HTTP request pipeline.
app.MapGrpcService<MemStoreService>();
app.MapGet("/", () => "Communication with gRPC endpoints must be made through a gRPC client. To learn how to create a client, visit: https://go.microsoft.com/fwlink/?linkid=2086909");
app.Run();
```

Let's run again our service

```shell
➜  DaprMemStoreComponent dotnet run
Building...
warn: Microsoft.AspNetCore.Server.Kestrel[0]
      Overriding address(es) 'http://localhost:5259, https://localhost:7089'. Binding to endpoints defined via IConfiguration and/or UseKestrel() instead.
info: Microsoft.Hosting.Lifetime[14]
      Now listening on: http://unix:/tmp/custom-store.sock
info: Microsoft.Hosting.Lifetime[0]
      Application started. Press Ctrl+C to shut down.
info: Microsoft.Hosting.Lifetime[0]
      Hosting environment: Development
info: Microsoft.Hosting.Lifetime[0]
```

Great! Now we have our component running using a unix socket, let's try out the same call we did, but now we should hint that the connection is a unix socket.

```shell
grpcurl -unix --plaintext --import-path "$PWD/Protos" --proto dapr/proto/components/v1/state.proto /tmp/custom-store.sock dapr.proto.components.v1.StateStore/Features
```

```shell
➜  DaprMemStoreComponent grpcurl -unix --plaintext --import-path "$PWD/Protos" --proto dapr/proto/components/v1/state.proto /tmp/custom-store.sock dapr.proto.components.v1.StateStore/Features
ERROR:
  Code: Unimplemented
  Message:
```

Great! Now we have our service backed by a unix socket. The next required change is to enabling gRPC reflection as it is the way the dapr runtime know which services your component implements, it works out-of-the-box by just using reflection API.

First add the `Grpc.AspNetCore.Server.Reflection` lib

```shell
dotnet add package Grpc.AspNetCore.Server.Reflection --version 2.49.0
```

Next, add enable gRPC reflection to our service:

```csharp
using DaprMemStoreComponent.Services;
using Microsoft.AspNetCore.Server.Kestrel.Core;
var socket = "/tmp/custom-store.sock";
if (File.Exists(socket))
{
    Console.WriteLine("Removing existing socket");
    File.Delete(socket);
}
var builder = WebApplication.CreateBuilder(args);
// Additional configuration is required to successfully run gRPC on macOS.
// For instructions on how to configure Kestrel and gRPC clients on macOS, visit https://go.microsoft.com/fwlink/?linkid=2099682
// Add services to the container.
builder.WebHost.ConfigureKestrel(options =>
            {
                options.ListenUnixSocket(socket, listenOptions =>
                {
                    listenOptions.Protocols = HttpProtocols.Http2;
                });
            });
builder.Services.AddGrpc();
builder.Services.AddGrpcReflection();
var app = builder.Build();
// Configure the HTTP request pipeline.
app.MapGrpcReflectionService();
app.MapGrpcService<MemStoreService>();
app.MapGet("/", () => "Communication with gRPC endpoints must be made through a gRPC client. To learn how to create a client, visit: https://go.microsoft.com/fwlink/?linkid=2086909");
app.Run();
```

Now we are ready to go,
