---
title: "Agones Game Server Client SDKs" 
linkTitle: "Client SDKs"
date: 2019-01-02T10:16:30Z
weight: 10
description: "The SDKs are integration points for game servers with Agones itself."
---

## Overview

The client SDKs are required for a game server to work with Agones.

The current supported SDKs are:

- [Unreal Engine]({{< relref "unreal.md" >}})
- [Unity]({{< relref "unity.md" >}})
- [C++]({{< relref "cpp.md" >}})
- [C#]({{< relref "csharp.md" >}})
- [Node.js]({{< relref "nodejs.md" >}})
- [Go]({{< relref "go.md" >}})
- [Rust]({{< relref "rust.md" >}})
- [REST]({{< relref "rest.md" >}})

You can also find some externally supported SDKs in our 
[Third Party Content]({{% ref "/docs/Third Party Content/libraries-tools.md#client-sdks" %}}).

Most SDKs are relatively thin wrappers around [gRPC](https://grpc.io) generated clients. However, in cases 
where gRPC client generation and compilation isn't well supported, the SDK may instead be using the REST API exposed via [grpc-gateway](https://github.com/grpc-ecosystem/grpc-gateway)) .

The SDK connects to a small process that Agones coordinates to run alongside the Game Server
in a Kubernetes [`Pod`](https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/).
This means that more languages can be supported in the future with minimal effort
(but pull requests are welcome! 😊 ).

There is also [local development tooling]({{< relref "local.md" >}}) for working against the SDK locally
without having to spin up an entire Kubernetes infrastructure.

## Connecting to the SDK Server

Starting with Agones 1.1.0, the port that the SDK Server listens on for incoming gRPC or HTTP requests is
configurable. This provides flexibility in cases where the default port conflicts with a port that is needed
by the game server.

Agones will automatically set the following environment variables on all game server containers:

* `AGONES_SDK_GRPC_PORT`: The port where the gRPC server is listening (defaults to 9357)
* `AGONES_SDK_HTTP_PORT`: The port where the grpc-gateway is listening (defaults to 9358)

The SDKs will automatically discover and connect to the gRPC port specified in Agones config by reading the environment variable.

If your game server cannot use any of the provided SDK clients and instead directly makes REST calls to the SDK server using an generic HTTP client, 
we recommend configuring your HTTP client to connect using the port from the `AGONES_SDK_HTTP_PORT` environment variable in case the SDK server is configured to listen on a different port.

## Function Reference

While each of the SDKs are canonical to their languages, they all have the following
functions that implement the core responsibilities of the SDK.

For language specific documentation have a look at the respective source (linked above) 
and the {{< ghlink href="examples" >}}examples{{< /ghlink >}}.

{{< alert title="Warning" color="warning">}}
Calling any of state-changing functions mentioned below does not guarantee that the `GameServer` Custom Resource object in Kubernetes will actually change its state immediately. For instance, if another event (such as a fleet scaling down) moves the GameServer to the `Shutdown` state while your SDK function is being processed, this could lead to your SDK call having no effect. If your approach depends on state transitions, you should verify the SDK call resulted in your desired pod state with callbacks by using the `WatchGameServer()` function.
{{< /alert >}}

Functions which change `GameServer` state or settings are:

1. `Ready()`
1. `Shutdown()`
1. `SetLabel()`
1. `SetAnnotation()`
1. `Allocate()`
1. `Reserve()`
1. `Alpha().SetCapacity()`
1. `Alpha().PlayerConnect()`
1. `Alpha().PlayerDisconnect()`

### Lifecycle Management

#### Ready()
This tells Agones that the game server is ready to take player connections.
This updates the Kubernetes `GameServer` record to the `Ready` state, and poulates the public 
IP address and connection port.

The preferred pattern is to call `Shutdown()` once a game has completed, and allow your defined 'fleet' and 'fleetAutoScaler' resources to start a new `GameServer` as necessary, which allows Agones to enforce the [scheduling policy]({{% ref "/docs/Advanced/scheduling-and-autoscaling.md" %}}) you've selected. 
This SDK call can also [change an `Allocated` `GameServer` back to the `Ready` state to be available for allocation again]({{% ref "/docs/Integration Patterns/reusing-gameservers.md" %}}), if this is a better fit for your usage pattern.

#### Health()
This sends a heartbeat to the SDK to designate that the `GameServer` is alive and
healthy. Failure to send heartbeats within the threshold configured in the `health` section of the 
`GameServer` spec will result in the `GameServer` being marked as `Unhealthy`. 

See the {{< ghlink href="examples/gameserver.yaml" >}}gameserver.yaml{{< /ghlink >}} for all health checking
configurations.

#### Reserve(seconds)

With some matchmaking systems, just-in-time allocation of game servers when a match is found is not a good fit. A common approach for these systems 'registers' a game server with the matchmaker while the search for a match is in progress.  The [Matchmaker registration pattern]({{% ref "/docs/Integration Patterns/matchmaker-registration.md" %}}) covers this style of matchmaker in more detail. For this use-case, Agones has the `Reserve(seconds)` SDK function, which:
*  Prevents the server from being deleted 
*  Signals to other systems that this server is in consideration by the matchmaker
*  **Crucially, will never** trigger a `FleetAutoscaler` scale up event, the way an `Allocate` call could 
The function accomplishes this by moving the `GameServer` into the `Reserved` state for the specified number of seconds (0 is forever), after which Agones moves it back to `Ready` state. While in `Reserved` state, the `GameServer` will not be deleted during a scale down or `Fleet` update event, and also it will not be returned when an external system requests a [GameServerAllocation]({{% ref "/docs/Reference/gameserverallocation.md" %}}) from the Agones controller.  

{{< alert title="Note" color="info">}}
To move a `GameServer` from `Reserved` to `Allocated` in this pattern, the **game server itself** should call `SDK.Allocate()`.  Note that state-changing SDK commands such as `Allocate()` or `Ready()` will attempt to update the state immediately, disregarding any timeout specified in a previous `Reserve(seconds)` call.
{{< /alert >}}

#### Allocate()

With some matchmaking strategies, game servers need to mark themselves as `Allocated`.
For those scenarios, Agones has the `Allocate()` SDK function.

The `agones.dev/last-allocated` annotation on the GameServer will be set to an RFC3339-formatted timestamp of the time of allocation, even if the GameServer was already in an `Allocated` state.

Note that if your usage pattern involves game servers using `SDK.Allocate()` in combination with external systems requesting [GameServerAllocation]({{% ref "/docs/Reference/gameserverallocation.md" %}}) from the Kubernetes API, it's possible for the `agones.dev/last-allocated` timestamp to move backwards if clocks are not synchronized between the Agones controller and the GameServer pod.

{{< alert title="Note" color="info">}}
Although game servers **can** mark themselves as allocated using this SDK function, requesting a 
[GameServerAllocation]({{% ref "/docs/Reference/gameserverallocation.md" %}}) from the Kubernetes 
API instead is the preferred pattern. 
This allows Agones to apply your [configured `fleet` scheduling strategy]({{% ref "/docs/Advanced/scheduling-and-autoscaling.md#fleet-scheduling" %}}), potentially resulting in better game server performance or cost savings. Having game servers call `Allocate()` relinquishs control of scheduling 
to an external service which likely doesn't have as much information about your Kubernetes usage as Agones.

Also, note that this function **does not** guarantee that the `GameServer` moves to the `Allocated` status. Please refer to the warning in the [Function Reference](#function-reference) section above about using the `WatchGameServer()` function.
{{< /alert >}}

#### Shutdown()
This tells Agones to shut down the game server which made the API call. The game server state will be set `Shutdown` and the 
backing Pod will be `Terminated`.

It's worth reading the [Termination of Pods](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-termination)
Kubernetes documentation, to understand the termination process, and the related configuration options.

As a rule of thumb, implement a graceful shutdown in your game sever process when it receives the TERM signal. 
Kubernetes will send this signal to your application when the backing Pod transitions to the `Terminated` state.

Be aware that if you use a variation of `System.exit(0)` after calling `SDK.Shutdown()`, your game server container may
restart for a brief period, inline with our [Health Checking]({{% ref "/docs/Guides/health-checking.md#health-failure-strategy" %}}) policies. 

It is possible for the SDK server to receive a TERM signal from Kubernetes without receiving a SDK.Shutdown() API request from
your GameServer. In this case, the SDK server will stay alive until the `terminationGracePeriodSeconds` in it's `podSpec` have
elapsed, or until it receives the `SDK.Shutdown()` API request from your game server, whichever comes first.

### Configuration Retrieval 

#### GameServer()

This returns most of the backing GameServer configuration and Status. This can be useful
for instances where you may want to know Health check configuration, or the IP and Port
the GameServer is currently allocated to.

Since the GameServer contains an entire [PodTemplate](https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/#pod-templates)
the returned object is limited to that configuration that was deemed useful. If there are
areas that you feel are missing, please [file an issue](https://github.com/googleforgames/agones/issues) or pull request.

The easiest way to see what is exposed, is to check
the {{% ghlink href="proto/sdk/sdk.proto" %}}`sdk.proto`{{% /ghlink %}} source file, specifically
the `message GameServer` portion.

For language specific documentation, have a look at the respective source (linked above) 
and the {{< ghlink href="examples" >}}examples{{< /ghlink >}}.

#### WatchGameServer(function(gameserver){...})

This executes the passed in callback with the current `GameServer` details whenever the underlying `GameServer` configuration is updated.
This can be useful to track `GameServer > Status > State` changes, `metadata` changes such as labels and annotations, and more.

In combination with this SDK, manipulating [Annotations](https://kubernetes.io/docs/concepts/overview/working-with-objects/annotations/) and
[Labels](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/) can also be a useful way to communicate information through Kubernetes to running game server processes 
from outside processes and systems.  This is especially useful when combined with
[metadata applied]({{< ref "/docs/Reference/gameserverallocation.md" >}}) during
`GameServerAllocation` .

Since the GameServer contains an entire [PodTemplate](https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/#pod-templates)
the returned object is limited to that configuration that was deemed useful. If there are
areas that you feel are missing, please [file an issue](https://github.com/googleforgames/agones/issues) or pull request.

The easiest way to see what is exposed, is to check
the {{% ghlink href="proto/sdk/sdk.proto" %}}`sdk.proto`{{% /ghlink %}} source file, specifically
the `message GameServer` portion..

For language specific documentation, have a look at the respective source (linked above) 
and the {{< ghlink href="examples" >}}examples{{< /ghlink >}}.


### Metadata Management

#### SetLabel(key, value)

This will set a [Label](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/) 
key/value pair on the backing `GameServer` record that is stored in Kubernetes. 

To maintain isolation, the `key` value is automatically prefixed with the value **"agones.dev/sdk-"**. This is done for 
two main reasons:
*  The prefix allows the developer to always know if they are accessing a value that could have come from, or 
   may be changed by the client SDK. Much like `private` vs `public` scope in a programming language, the Agones 
   SDK only provides write access to GameServer labels with this prefix.
*  The prefix allows for a smaller attack surface if the GameServer container gets compromised. Since a common approach is to expose GameServer containers directly to the internet to minimize network latency and the Agones project doesn't control what runs inside the container, limiting exposure if the pod were to become compromised is worth the extra development friction that comes with having this prefix in place.

{{< alert title="Warning" color="warning">}}
Kubernetes has [limits on label lengths and which characters are allowed in keys and values](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/#syntax-and-character-set). 
 
Be sure to account for the label prefix above when considering length limits! 
{{< /alert >}}

Setting `GameServer` labels can be useful if you want information from your running game server process to be 
observable or searchable through the Kubernetes API.  

#### SetAnnotation(key, value)

This will set an [Annotation](https://kubernetes.io/docs/concepts/overview/working-with-objects/annotations/) key/value pair
on the backing `GameServer` record that is stored in Kubernetes. 

To maintain isolation, the `key` value is automatically prefixed with **"agones.dev/sdk-"** for the same reasons as 
in [SetLabel(...)](#setlabelkey-value) above. The isolation is also important as Agones uses annotations on the 
`GameServer` as part of its internal processing.

Setting `GameServer` annotations can be useful if you want information from your running game server process to be 
observable through the Kubernetes API.

### Player Tracking

{{< alpha title="Player Tracking" gate="PlayerTracking" >}}

#### Alpha().PlayerConnect(playerID)

This function increases the SDK’s stored player count by one, and appends this playerID to 
`GameServer.Status.Players.IDs`.

[`GameServer.Status.Players.Count` and `GameServer.Status.Players.IDs`][playerstatus]
are then set to update the player count and id list a second from now,
unless there is already an update pending, in which case the update joins that batch operation.

`PlayerConnect()` returns true and adds the playerID to the list of playerIDs if this playerID was not already in the
list of connected playerIDs.

If the playerID exists within the list of connected playerIDs, `PlayerConnect()` will return false, and the list of
connected playerIDs will be left unchanged.

An error will be returned if the playerID was not already in the list of connected playerIDs but the player capacity for
the server has been reached. The playerID will not be added to the list of playerIDs.

{{< alert title="Note" color="info">}}
Do not use this method if you are manually managing `GameServer.Status.Players.IDs` and `GameServer.Status.Players.Count`
through the Kubernetes API, as indeterminate results will occur.  
{{< /alert >}}
    
#### Alpha().PlayerDisconnect(playerID)

This function decreases the SDK’s stored player count by one, and removes the playerID from 
[`GameServer.Status.Players.IDs`][playerstatus].

`GameServer.Status.Players.Count` and `GameServer.Status.Players.IDs` are then set to 
update the player count and id list a second from now,
unless there is already an update pending, in which case the update joins that batch operation.

`PlayerDisconnect()` will return true and remove the supplied playerID from the list of connected playerIDs if the
playerID value exists within the list.

If the playerID was not in the list of connected playerIDs, the call will return false, and the connected playerID list
will be left unchanged.

{{< alert title="Note" color="info">}}
Do not use this method if you are manually managing `GameServer.Status.Players.IDs` and `GameServer.Status.Players.Count`
through the Kubernetes API, as indeterminate results will occur.  
{{< /alert >}}

#### Alpha().SetPlayerCapacity(count)

Update the [`GameServer.Status.Players.Capacity`][playerstatus] value with a new capacity.

#### Alpha().GetPlayerCapacity()

This function retrieves the current player capacity. This is always accurate from what has been set through this SDK,
even if the value has yet to be updated on the GameServer status resource.

{{< alert title="Note" color="info">}}
If `GameServer.Status.Players.Capacity` is set manually through the Kubernetes API, use `SDK.GameServer()` or 
`SDK.WatchGameServer()` instead to view this value.
{{< /alert >}}

#### Alpha().GetPlayerCount()

This function retrieves the current player count. 
This is always accurate from what has been set through this SDK, even if the value has yet to be updated on the 
GameServer status resource.

{{< alert title="Note" color="info">}}
If `GameServer.Status.Players.IDs` is set manually through the Kubernetes API, use SDK.GameServer() 
or SDK.WatchGameServer() instead to retrieve the current player count.
{{< /alert >}}

#### Alpha().IsPlayerConnected(playerID)

This function returns if the playerID is currently connected to the GameServer. This is always accurate from what has
been set through this SDK,
even if the value has yet to be updated on the GameServer status resource.

{{< alert title="Note" color="info">}}
If `GameServer.Status.Players.IDs` is set manually through the Kubernetes API, use SDK.GameServer() 
or SDK.WatchGameServer() instead to determine connected status.
{{< /alert >}}

#### Alpha().GetConnectedPlayers()

This function returns the list of the currently connected player ids. This is always accurate from what has been set
through this SDK, even if the value has yet to be updated on the GameServer status resource.

{{< alert title="Note" color="info">}}
If `GameServer.Status.Players.IDs` is set manually through the Kubernetes API, use SDK.GameServer() 
or SDK.WatchGameServer() instead to list the connected players.
{{< /alert >}}

[playerstatus]: {{< ref "/docs/Reference/agones_crd_api_reference.html#agones.dev/v1.PlayerStatus" >}}

## Writing your own SDK

If there isn't an SDK for the language and platform you are looking for, you have several options:

### gRPC Client Generation

If client generation is well supported by [gRPC](https://grpc.io/docs/), then generate client(s) from
the proto files found in the {{% ghlink href="proto/sdk" %}}`proto/sdk`{{% /ghlink %}},
directory and look at the current {{< ghlink href="sdks" >}}sdks{{< /ghlink >}} to see how the wrappers are
implemented to make interaction with the SDK server simpler for the user.

### REST API Implementation

If client generation is not well supported by gRPC, or if there are other complicating factors, implement the SDK through
the [REST]({{< relref "rest.md" >}}) HTTP+JSON interface. This could be written by hand, or potentially generated from
the {{< ghlink href="sdks/swagger" >}}Swagger/OpenAPI Specifications{{< /ghlink >}}.

Finally, if you build something that would be usable by the community, please submit a pull request!

## SDK Conformance Test

There is a tool `SDK server Conformance` checker which will run Local SDK server and record all requests your client is performing.

In order to check that SDK is working properly you should write simple SDK test client which would use all methods of your SDK.

Also to test that SDK client is receiving valid Gameserver data, your binary should set the same `Label` value as creation timestamp which you will receive as a result of GameServer() call and `Annotation` value same as gameserver UID received by Watch gameserver callback.

Complete list of endpoints which should be called by your test client is the following:
```
ready,allocate,setlabel,setannotation,gameserver,health,shutdown,watch
```

In order to run this test SDK server locally use:
```
SECONDS=30 make run-sdk-conformance-local
```

Docker container would timeout in 30 seconds and give your the comparison of received requests and expected requests

For instance you could run Go SDK conformance test and see how the process goes: 
```
SDK_FOLDER=go make run-sdk-conformance-test
```

In order to add test client for your SDK, write `sdktest.sh` and `Dockerfile`. Refer to {{< ghlink href="build/build-sdk-images/go" >}}Golang SDK Conformance testing directory structure{{< /ghlink >}}.

## Building the Tools

If you wish to build the binaries from source
the `make` target `build-agones-sdk-binary` will compile the necessary binaries
for all supported operating systems (64 bit windows, linux and osx).

You can find the binaries in the `bin` folder in {{< ghlink href="cmd/sdk-server" >}}`cmd/sdk-server`{{< /ghlink >}}
once compilation is complete.

See {{< ghlink href="build" branch="main" >}}Developing, Testing and Building Agones{{< /ghlink >}} for more details.
