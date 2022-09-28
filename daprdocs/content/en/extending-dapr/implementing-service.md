### Implementing your service

<img src="/images/pluggable-component-arch.png" width=800 alt="Diagram showing the final MemStore design">

Go to our `MemStoreService.cs` and let's override the ListFeature method

```csharp
using Dapr.Client.Autogen.Grpc.v1;
using Dapr.Proto.Components.V1;
using Grpc.Core;
namespace DaprMemStoreComponent.Services;
public class MemStoreService : StateStore.StateStoreBase
{
    private readonly ILogger<MemStoreService> _logger;
    public MemStoreService(ILogger<MemStoreService> logger)
    {
        _logger = logger;
    }
    public override Task<FeaturesResponse> Features(FeaturesRequest _, ServerCallContext context)
    {
        var resp = new FeaturesResponse();
        resp.Feature.Add("TRANSACTIONAL");
        return Task.FromResult(resp);
    }
}
```

{{% /codetab %}}

{{< /tabs >}}