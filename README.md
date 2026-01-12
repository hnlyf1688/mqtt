# System.Net.MQTT

A high-performance, lightweight MQTT client and broker library for .NET.

## Features

- **Multi-target support**: .NET 6.0, .NET 8.0, .NET 10.0
- **MQTT 3.1.1 compliant**: Full protocol support
- **High performance**: Async/await, zero unnecessary allocations
- **TLS/SSL support**: Secure connections with certificate support
- **QoS levels**: At most once (0), At least once (1), Exactly once (2)
- **Retained messages**: Message persistence support
- **Will messages**: Last Will and Testament (LWT) support
- **Topic wildcards**: Single-level (+) and multi-level (#) wildcards
- **Authentication**: Pluggable authentication and authorization
- **Auto-reconnect**: Automatic reconnection on disconnect

## Solution Structure

```
MQTT/
├── MQTT.slnx                          # Solution file (slnx format)
├── README.md                          # This file
├── src/
│   ├── System.Net.MQTT/              # Client library
│   │   ├── IMqttClient.cs            # Client interface
│   │   ├── MqttClient.cs             # Client implementation
│   │   ├── MqttClientOptions.cs      # Connection options
│   │   ├── MqttApplicationMessage.cs # Message model
│   │   ├── MqttQualityOfService.cs   # QoS enum
│   │   ├── MqttProtocolVersion.cs    # Protocol versions
│   │   ├── MqttPacketType.cs         # Packet types
│   │   ├── MqttConnectResult.cs      # Connection result
│   │   ├── MqttSubscription.cs       # Subscription models
│   │   ├── MqttEventArgs.cs          # Event arguments
│   │   └── MqttExceptions.cs         # Exception types
│   │
│   └── System.Net.MQTT.Broker/       # Broker library
│       ├── MqttBroker.cs             # Broker implementation
│       ├── MqttBrokerOptions.cs      # Broker options
│       ├── MqttClientSession.cs      # Client session
│       ├── MqttAuthentication.cs     # Auth interfaces
│       └── MqttBrokerEventArgs.cs    # Broker events
│
└── samples/
    ├── MqttClient.Sample/            # Client sample
    └── MqttBroker.Sample/            # Broker sample
```

## Quick Start

### Install

```bash
# Client library
dotnet add package System.Net.MQTT

# Broker library
dotnet add package System.Net.MQTT.Broker
```

### Client Usage

```csharp
using System.Net.MQTT;

// Create client
var options = new MqttClientOptions
{
    Host = "localhost",
    Port = 1883,
    ClientId = "my-client",
    CleanSession = true
};

using var client = new MqttClient(options);

// Event handlers
client.MessageReceived += (s, e) =>
{
    Console.WriteLine($"Received: {e.Message.Topic} = {e.Message.PayloadAsString}");
};

// Connect
await client.ConnectAsync();

// Subscribe
await client.SubscribeAsync("sensors/#", MqttQualityOfService.AtLeastOnce);

// Publish
await client.PublishAsync("sensors/temperature", "25.5");

// Disconnect
await client.DisconnectAsync();
```

### Broker Usage

```csharp
using System.Net.MQTT.Broker;

// Create broker
var options = new MqttBrokerOptions
{
    Port = 1883,
    AllowAnonymous = true
};

using var broker = new MqttBroker(options);

// Event handlers
broker.ClientConnected += (s, e) =>
{
    Console.WriteLine($"Client connected: {e.Session.ClientId}");
};

broker.MessagePublished += (s, e) =>
{
    Console.WriteLine($"Message: {e.Message.Topic} = {e.Message.PayloadAsString}");
};

// Start broker
await broker.StartAsync();

// Wait for shutdown signal
await Task.Delay(Timeout.Infinite);

// Stop broker
await broker.StopAsync();
```

## Client Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| Host | string | - | Broker host address |
| Port | int | 1883 | Broker port |
| ClientId | string | GUID | Client identifier |
| Username | string? | null | Authentication username |
| Password | string? | null | Authentication password |
| UseTls | bool | false | Enable TLS/SSL |
| CleanSession | bool | true | Start with clean session |
| KeepAliveSeconds | ushort | 60 | Keep alive interval |
| AutoReconnect | bool | true | Auto reconnect on disconnect |
| ReconnectDelayMs | int | 5000 | Reconnect delay in milliseconds |

## Broker Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| BindAddress | string | 0.0.0.0 | IP address to bind to |
| Port | int | 1883 | Listen port |
| UseTls | bool | false | Enable TLS |
| TlsPort | int | 8883 | TLS port |
| MaxConnections | int | 10000 | Max concurrent connections |
| AllowAnonymous | bool | true | Allow anonymous connections |
| EnableRetainedMessages | bool | true | Enable retained messages |
| EnablePersistentSessions | bool | true | Enable persistent sessions |

## Authentication

```csharp
// Simple username/password authentication
var authenticator = new SimpleAuthenticator()
    .AddUser("admin", "password")
    .AddUser("user1", "pass123");

broker.Authenticator = authenticator;
```

### Custom Authentication

```csharp
public class MyAuthenticator : IMqttAuthenticator
{
    public async Task<bool> AuthenticateAsync(
        string clientId,
        string? username,
        string? password,
        CancellationToken cancellationToken)
    {
        // Your authentication logic
        return await ValidateCredentialsAsync(username, password);
    }
}
```

## TLS Configuration

### Client

```csharp
var options = new MqttClientOptions
{
    Host = "broker.example.com",
    Port = 8883,
    UseTls = true,
    ClientCertificate = new X509Certificate2("client.pfx", "password"),
    SkipCertificateValidation = false // Set true for self-signed certs
};
```

### Broker

```csharp
var options = new MqttBrokerOptions
{
    UseTls = true,
    TlsPort = 8883,
    ServerCertificate = new X509Certificate2("server.pfx", "password"),
    RequireClientCertificate = false
};
```

## Quality of Service

| Level | Name | Description |
|-------|------|-------------|
| 0 | At most once | Fire and forget, no acknowledgment |
| 1 | At least once | Acknowledged delivery, may duplicate |
| 2 | Exactly once | Assured delivery, no duplicates |

```csharp
// QoS 0 - Fire and forget
await client.PublishAsync("topic", "message", MqttQualityOfService.AtMostOnce);

// QoS 1 - At least once
await client.PublishAsync("topic", "message", MqttQualityOfService.AtLeastOnce);

// QoS 2 - Exactly once
await client.PublishAsync("topic", "message", MqttQualityOfService.ExactlyOnce);
```

## Topic Wildcards

| Wildcard | Description | Example |
|----------|-------------|---------|
| + | Single level | sensors/+/temperature |
| # | Multi level | sensors/# |

```csharp
// Match: sensors/living-room/temperature, sensors/bedroom/temperature
await client.SubscribeAsync("sensors/+/temperature");

// Match: sensors/*, sensors/room1/*, sensors/room1/temp, etc.
await client.SubscribeAsync("sensors/#");
```

## Will Message (LWT)

```csharp
var options = new MqttClientOptions
{
    Host = "localhost",
    WillMessage = MqttApplicationMessage.Create(
        topic: "clients/my-client/status",
        payload: "offline",
        qos: MqttQualityOfService.AtLeastOnce,
        retain: true
    )
};
```

## Building

```bash
# Build all projects
dotnet build MQTT.slnx

# Run broker sample
dotnet run --project samples/MqttBroker.Sample

# Run client sample (in another terminal)
dotnet run --project samples/MqttClient.Sample
```

## Requirements

- .NET 6.0, .NET 8.0, or .NET 10.0
- Visual Studio 2022 17.10+ (for slnx support) or VS Code with C# Dev Kit

## License

MIT License

<a href="https://contributors-img.web.app/image?repo=hnlyf1688/mqtt">
  <img src="https://contributors-img.web.app/image?repo=hnlyf1688/mqtt" />
</a>
