# Load Balancing

Introduced: preview8

## Introduction

Whenever there are multiple healthy destinations available, YARP has to decide which one to use for a given request.
YARP ships with built-in load-balancing algorithms, but also offers extensibility for any custom load balancing approach.

## Configuration

### Services and middleware registration

Load balancing policies are registered in the DI container via the `AddLoadBalancingPolicies()` method, which is automatically called by `AddReverseProxy()`.

The middleware is added with `UseProxyLoadBalancing()`, which is included by default in the parameterless `MapReverseProxy` method.

### Cluster configuration

The algorithm used to determine the destination can be configured by setting the `LoadBalancingPolicy`.

If no policy is specified, `PowerOfTwoChoices` will be used.

#### File example

```JSON
"ReverseProxy": {
    "Clusters": {
        "cluster1": {
            "LoadBalancingPolicy": "RoundRobin",
            "Destinations": {
                "cluster1/destination1": {
                    "Address": "https://localhost:10000/"
                },
                "cluster1/destination2": {
                    "Address": "https://localhost:10010/"
                }
            }
        }
    }
}
```

#### Code example

```C#
var clusters = new[]
{
    new Cluster()
    {
        Id = "cluster1",
        LoadBalancingPolicy = LoadBalancingPolicies.RoundRobin,
        Destinations =
        {
            { "destination1", new Destination() { Address = "https://localhost:10000" } },
            { "destination2", new Destination() { Address = "https://localhost:10010" } }
        }
    }
};
```

## Built-in policies

YARP ships with the following built-in policies:
- `First`

    Select the first destination without considering load. This is useful for dual destination fail-over systems.
- `Random`

    Select a destination randomly.
- `PowerOfTwoChoices` (default)

    Select two random destinations and then select the one with the least assigned requests.
    This avoids the overhead of `LeastRequests` and the worst case for `Random` where it selects a busy destination.
- `RoundRobin`

    Select a destination by cycling through them in order.
- `LeastRequests`

    Select the destination with the least assigned requests. This requires examining all destinations.

## Extensibility

`ILoadBalancingPolicy` is responsible for picking a destination from a list of available healthy destinations.

A custom implementation can be provided in DI.

```c#
// Implement the ILoadBalancingPolicy
public sealed class LastLoadBalancingPolicy : ILoadBalancingPolicy
{
    public string Name => "Last";

    public DestinationInfo PickDestination(HttpContext context, IReadOnlyList<DestinationInfo> availableDestinations)
    {
        return availableDestinations[^1];
    }
}

// Register it in DI
services.AddSingleton<ILoadBalancingPolicy, LastLoadBalancingPolicy>();

// Set the LoadBalancingPolicy on the cluster
cluster.LoadBalancingPolicy = "Last";
```

Other information that may be necessary to decide on a destination, such as cluster configuration, can be accessed from the `HttpContext`:

```c#
public DestinationInfo PickDestination(HttpContext context, IReadOnlyList<DestinationInfo> availableDestinations)
{
    var proxyFeature = context.Features.Get<IReverseProxyFeature>();
    var clusterConfig = proxyFeature.ClusterConfig;
    // ...
}
```