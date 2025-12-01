# GenesisDB gRPC Endpoint

This document explains how to work with the GenesisDB gRPC endpoint.

## Overview

GenesisDB provides a gRPC API alongside its HTTP API. The gRPC server runs on port 50051 by default and provides access to all core event store operations.

## Configuration

### Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `GENESISDB_GRPC_ENABLED` | Enable/disable the gRPC server | `false` |
| `GENESISDB_GRPC_PORT` | Port for the gRPC server | `50051` |

### Enabling gRPC

Set the environment variable before starting GenesisDB:

```bash
export GENESISDB_GRPC_ENABLED=true
export GENESISDB_GRPC_PORT=50051
./genesisdb
```

## Authentication

All gRPC methods (except `Ping`) require authentication via metadata headers.

Pass the authorization token in the `authorization` metadata key:

```go
md := metadata.New(map[string]string{
    "authorization": "your-api-token",
})
ctx := metadata.NewOutgoingContext(context.Background(), md)
```

## Proto Definition

The service is defined in `proto/genesisdb.proto`. The package name is `genesisdb` and the Go package is `genesisdb.io/genesisdb/proto`.

## Available RPCs

### Core Operations

| RPC | Description | Request | Response |
|-----|-------------|---------|----------|
| `Commit` | Write events to the store | `CommitRequest` | `CommitResponse` |
| `Stream` | Retrieve events for a subject | `StreamRequest` | `StreamResponse` |
| `Observe` | Watch for new events (streaming) | `ObserveRequest` | `stream Event` |
| `Query` | Execute a GQL query (Enterprise) | `QueryRequest` | `QueryResponse` |

### Status & Health

| RPC | Description | Request | Response |
|-----|-------------|---------|----------|
| `Ping` | Health check (no auth required) | `PingRequest` | `PingResponse` |
| `GetStatus` | Detailed server status | `GetStatusRequest` | `GetStatusResponse` |
| `GetSubjects` | List all subjects | `GetSubjectsRequest` | `GetSubjectsResponse` |
| `GetTypes` | List all event types | `GetTypesRequest` | `GetTypesResponse` |

### Backup & Restore

| RPC | Description | Request | Response |
|-----|-------------|---------|----------|
| `CreateBackup` | Export all events | `CreateBackupRequest` | `CreateBackupResponse` |
| `RestoreBackup` | Import events from backup | `RestoreBackupRequest` | `RestoreBackupResponse` |

### Enterprise Features

| RPC | Description | Request | Response |
|-----|-------------|---------|----------|
| `EraseSubject` | Delete events for GDPR compliance | `EraseSubjectRequest` | `EraseSubjectResponse` |
| `RegisterSchema` | Register JSON schema for event type | `RegisterSchemaRequest` | `RegisterSchemaResponse` |
| `GetSchemas` | Retrieve registered schemas | `GetSchemasRequest` | `GetSchemasResponse` |
| `Audit` | Check integrity of event chains | `AuditRequest` | `AuditResponse` |

## Message Types

### Event

The core event structure returned by the API:

```protobuf
message Event {
  string source = 1;
  string subject = 2;
  string type = 3;
  string specversion = 4;
  string id = 5;
  google.protobuf.Timestamp time = 6;
  string datacontenttype = 7;
  string predecessorhash = 8;
  google.protobuf.Struct data = 9;
  string hash = 10;
}
```

### EventInput

Used when committing events:

```protobuf
message EventInput {
  string source = 1;
  string subject = 2;
  string type = 3;
  google.protobuf.Struct data = 4;
  google.protobuf.Struct ref = 5;
  EventOptions options = 6;
}
```

**Required fields:**
- `source` - Origin of the event
- `subject` - Must start with `/`
- `type` - Event type identifier
- `data` or `ref` - One must be provided (not both)

### StreamOptions

Filter events when streaming:

```protobuf
message StreamOptions {
  string lower_bound = 1;           // Event ID to start from
  bool include_lower_bound_event = 2;
  string upper_bound = 3;           // Event ID to end at
  bool include_upper_bound_event = 4;
  string latest_by_event_type = 5;  // Return only latest event of this type
}
```

### Precondition

Conditional commits:

```protobuf
message Precondition {
  string type = 1;
  google.protobuf.Struct payload = 2;
}
```

## Usage Examples

### Using grpcurl

The gRPC server has reflection enabled, so you can use `grpcurl` for debugging.

**List available services:**
```bash
grpcurl -plaintext localhost:50051 list
```

**Describe a service:**
```bash
grpcurl -plaintext localhost:50051 describe genesisdb.GenesisDBService
```

**Ping (no auth):**
```bash
grpcurl -plaintext localhost:50051 genesisdb.GenesisDBService/Ping
```

**Get status:**
```bash
grpcurl -plaintext \
  -H "authorization: your-token" \
  localhost:50051 genesisdb.GenesisDBService/GetStatus
```

**Commit an event:**
```bash
grpcurl -plaintext \
  -H "authorization: your-token" \
  -d '{
    "events": [{
      "source": "my-app",
      "subject": "/users/123",
      "type": "user.created",
      "data": {
        "name": "John Doe",
        "email": "john@example.com"
      }
    }]
  }' \
  localhost:50051 genesisdb.GenesisDBService/Commit
```

**Stream events:**
```bash
grpcurl -plaintext \
  -H "authorization: your-token" \
  -d '{"subject": "/users/123"}' \
  localhost:50051 genesisdb.GenesisDBService/Stream
```

**Observe (server streaming):**
```bash
grpcurl -plaintext \
  -H "authorization: your-token" \
  -d '{"subject": "/users/123", "interval": 1000}' \
  localhost:50051 genesisdb.GenesisDBService/Observe
```

### Using Go Client

```go
package main

import (
    "context"
    "log"

    pb "genesisdb.io/genesisdb/proto"
    "google.golang.org/grpc"
    "google.golang.org/grpc/credentials/insecure"
    "google.golang.org/grpc/metadata"
    "google.golang.org/protobuf/types/known/structpb"
)

func main() {
    // Connect to gRPC server
    conn, err := grpc.Dial("localhost:50051",
        grpc.WithTransportCredentials(insecure.NewCredentials()))
    if err != nil {
        log.Fatalf("Failed to connect: %v", err)
    }
    defer conn.Close()

    client := pb.NewGenesisDBServiceClient(conn)

    // Create authenticated context
    md := metadata.New(map[string]string{
        "authorization": "your-api-token",
    })
    ctx := metadata.NewOutgoingContext(context.Background(), md)

    // Commit an event
    data, _ := structpb.NewStruct(map[string]interface{}{
        "name":  "John Doe",
        "email": "john@example.com",
    })

    resp, err := client.Commit(ctx, &pb.CommitRequest{
        Events: []*pb.EventInput{{
            Source:  "my-app",
            Subject: "/users/123",
            Type:    "user.created",
            Data:    data,
        }},
    })
    if err != nil {
        log.Fatalf("Commit failed: %v", err)
    }

    log.Printf("Committed %d events", len(resp.Events))

    // Stream events
    streamResp, err := client.Stream(ctx, &pb.StreamRequest{
        Subject: "/users/123",
    })
    if err != nil {
        log.Fatalf("Stream failed: %v", err)
    }

    for _, event := range streamResp.Events {
        log.Printf("Event: %s - %s", event.Id, event.Type)
    }
}
```

### Observing Events (Streaming)

```go
func observeEvents(client pb.GenesisDBServiceClient, ctx context.Context) {
    stream, err := client.Observe(ctx, &pb.ObserveRequest{
        Subject:  "/users/123",
        Interval: 1000, // milliseconds
    })
    if err != nil {
        log.Fatalf("Observe failed: %v", err)
    }

    for {
        event, err := stream.Recv()
        if err != nil {
            log.Printf("Stream ended: %v", err)
            break
        }
        log.Printf("New event: %s - %s", event.Id, event.Type)
    }
}
```

## Generating Client Code

To generate client code for other languages, use the protobuf compiler with the gRPC plugin.

### Prerequisites

Install the Protocol Buffer compiler and gRPC plugins:

```bash
# macOS
brew install protobuf
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest

# Ensure $GOPATH/bin is in your PATH
export PATH="$PATH:$(go env GOPATH)/bin"
```

### Generate Go Code

```bash
protoc --go_out=. --go_opt=paths=source_relative \
       --go-grpc_out=. --go-grpc_opt=paths=source_relative \
       proto/genesisdb.proto
```

### Generate Python Code

```bash
pip install grpcio-tools
python -m grpc_tools.protoc -I. --python_out=. --grpc_python_out=. proto/genesisdb.proto
```

### Generate Node.js Code

```bash
npm install @grpc/proto-loader grpc
# Use dynamic loading with @grpc/proto-loader, or generate static code with grpc-tools
```

## Error Handling

The gRPC API uses standard gRPC status codes:

| Code | Usage |
|------|-------|
| `OK` | Success |
| `INVALID_ARGUMENT` | Invalid request parameters |
| `UNAUTHENTICATED` | Missing or invalid auth token |
| `PERMISSION_DENIED` | Enterprise feature without license |
| `NOT_FOUND` | Resource not found |
| `ALREADY_EXISTS` | Resource already exists (e.g., schema) |
| `FAILED_PRECONDITION` | Precondition check failed |
| `RESOURCE_EXHAUSTED` | Event limit reached |
| `INTERNAL` | Server error |

## Event Validation Rules

When committing events:

1. `source` - Cannot be empty
2. `subject` - Cannot be empty, must start with `/`
3. `type` - Cannot be empty
4. `data` or `ref` - Exactly one must be provided
5. Event size must not exceed 64KB (including all fields)

## Enterprise Features

The following RPCs require an active Enterprise license:

- `Query` - GQL query execution
- `EraseSubject` - GDPR compliance data erasure
- `RegisterSchema` - JSON schema registration
- `GetSchemas` - Schema retrieval
- `Audit` - Event chain integrity verification

Without a license, the Enterprise Edition is limited to 10,000 events.

## TLS Configuration

For production deployments, configure TLS on the gRPC server. The current implementation uses plaintext connections. For TLS support, you would need to modify the server startup to include credentials:

```go
creds, err := credentials.NewServerTLSFromFile("cert.pem", "key.pem")
s := grpc.NewServer(
    grpc.Creds(creds),
    grpc.UnaryInterceptor(unaryAuthInterceptor),
    grpc.StreamInterceptor(streamAuthInterceptor),
)
```

## Debugging

gRPC reflection is enabled by default, which allows tools like `grpcurl` and gRPC GUI clients to discover the API schema dynamically.

To verify the server is running:

```bash
grpcurl -plaintext localhost:50051 list
# Should output: genesisdb.GenesisDBService

grpcurl -plaintext localhost:50051 genesisdb.GenesisDBService/Ping
# Should output: { "status": "ok" }
```

## License

MIT

## Author

* E-Mail: mail@genesisdb.io
* URL: https://www.genesisdb.io
* Docs: https://docs.genesisdb.io
