# protoc-gen-typescript-http

Generates Typescript types and service clients from protobuf definitions
annotated with
[http rules](https://github.com/googleapis/googleapis/blob/master/google/api/http.proto).
The generated types follow the
[canonical JSON encoding](https://developers.google.com/protocol-buffers/docs/proto3#json).

**Experimental**: This library is under active development and breaking changes
to config files, APIs and generated code are expected between releases.

## Using the plugin

For examples of correctly annotated protobuf defintions and the generated code,
look at [examples](./examples).

### Install the plugin

```bash
go install github.com/go-kratos/protoc-gen-typescript-http@latest
```

Or download a prebuilt binary from [releases](./releases).

### Invocation

```bash
protoc 
  --typescript-http_out [OUTPUT DIR] \
  [.proto files ...]
```

______________________________________________________________________

The generated clients can be used with any HTTP client that returns a Promise
containing JSON data.

```typescript
const rootUrl = "...";

type Request = {
  path: string,
  method: string,
  body: string | null
}

function fetchRequestHandler({path, method, body}: Request) {
  return fetch(rootUrl + path, {method, body}).then(response => response.json())
}

export function siteClient() {
  return createShipperServiceClient(fetchRequestHandler);
}
```

### Streaming

Server-streaming RPCs (`returns (stream ...)`) are generated as **SSE** (Server-Sent Events).
Bidirectional streaming RPCs (`stream ... returns (stream ...)`) are generated as **WebSocket**.
Both work alongside the existing `RequestHandler` — no extra configuration needed.

Example proto:

```protobuf
service LogService {
  rpc GetLog(GetLogRequest) returns (GetLogResponse) {
    option (google.api.http) = {get: "/v1/logs"};
  }

  // Server-streaming → SSE
  rpc TailLogs(TailLogsRequest) returns (stream LogEntry) {
    option (google.api.http) = {get: "/v1/logs:tail"};
  }

  // Bidirectional streaming → WebSocket
  rpc Chat(stream ChatMessage) returns (stream ChatMessage) {
    option (google.api.http) = {get: "/v1/chat"};
  }
}
```

Generated usage:

```typescript
const client = createLogServiceClient(fetchRequestHandler);

// Unary — unchanged
const log = await client.GetLog({ name: "log/123" });

// Server-streaming (SSE)
const tail = client.TailLogs({ name: "log/123" });
tail.subscribe((entry) => {
  console.log("log entry:", entry.message);
});
tail.onError((err) => {
  console.error("tail error:", err);
});
// tail.close();

// Bidirectional streaming (WebSocket)
const chat = client.Chat();
chat.subscribe((msg) => {
  console.log("received:", msg.text);
});
chat.onError((err) => {
  console.error("chat error:", err);
});
chat.send({ text: "hello" });
// chat.close();
```
