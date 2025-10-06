---
title: "Enabling gRPC API for Grafana Tempo"
date: 2025-10-06T22:11:33+02:00
description: "A comprehensive guide to configuring Grafana Tempo with gRPC support using NGINX ingress in Kubernetes."
---

When running Grafana Tempo in a distributed Kubernetes environment, exposing both HTTP and gRPC APIs through ingress can significantly improve performance and flexibility for trace ingestion and querying. While the default Tempo gateway provides a convenient unified interface, directly exposing the distributor and querier services via NGINX ingress offers more control and better performance for gRPC-native clients.

This guide walks through configuring NGINX ingress to expose Tempo's gRPC endpoints alongside traditional HTTP APIs, enabling:

- **High-performance trace ingestion** via OpenTelemetry Protocol (OTLP) over gRPC
- **Efficient querying** using Tempo's native gRPC streaming APIs
- **Multi-protocol support** with simultaneous HTTP and gRPC endpoints on the same host
- **Production-ready setup** with TLS termination and multi-tenancy support

Whether you're pushing millions of spans per second or need low-latency trace queries, gRPC provides measurable benefits: reduced bandwidth usage, lower latency, bidirectional streaming, and type-safe communication through protocol buffers. By the end of this guide, you'll have a fully functional Tempo deployment with both HTTP and gRPC APIs exposed through Kubernetes ingress.

## Versions

This guide was tested with the following versions:
- **Grafana Tempo**: v2.8.2
- **Helm Chart**: `grafana/tempo-distributed` v1.48.0
- **Kubernetes**: v1.30+ (tested with v1.32.9)
- **NGINX Ingress Controller**: v1.13.3 (compatible with v1.0+)
- **cert-manager**: v1.0+ for TLS certificate management

**Protocol Documentation:**
- [Tempo gRPC Protocol](https://github.com/grafana/tempo/blob/v2.8.2/pkg/tempopb/tempo.proto) - Tempo's native gRPC service definitions
- [OpenTelemetry gRPC Protocol](https://github.com/open-telemetry/opentelemetry-proto/blob/v1.8.0/opentelemetry/proto/trace/v1/trace.proto) - OTLP trace protocol definitions

## API Layout

This setup exposes Tempo services for both read (query) and write (ingestion) operations. For detailed information about Tempo's architecture, see the [official Tempo architecture documentation](https://grafana.com/docs/tempo/latest/operations/architecture/).

```
┌─────────────┬──────────────────────────────────────────────────────┬──────────────────────┐
│ Protocol    │ Endpoint                                             │ Operation            │
├─────────────┼──────────────────────────────────────────────────────┼──────────────────────┤
│ HTTP        │ /api/*                           (Tempo Protocol)    │ Read                 │
│             │ ├── /api/v2/search/tags                              │ Query tags           │
│             │ ├── /api/v2/traces/<trace_id>                        │ Query trace by ID    │
│             │ └── ...                                              │                      │
├─────────────┼──────────────────────────────────────────────────────┼──────────────────────┤
│ HTTP        │ /v1/*                            (OTLP Protocol)     │ Write                │
│             │ ├── /v1/traces                                       │ Ingest traces        │
│             │ └── ...                                              │                      │
├─────────────┼──────────────────────────────────────────────────────┼──────────────────────┤
│ gRPC        │ /tempopb.*                       (Tempo Protocol)    │ Read                 │
│             │ ├── /tempopb.StreamingQuerier.SearchTagsV2           │ Query tags           │
│             │ ├── /tempopb.StreamingQuerier.MetricsQueryRange      │ Query metrics        │
│             │ └── ...                                              │                      │
├─────────────┼──────────────────────────────────────────────────────┼──────────────────────┤
│ gRPC        │ /opentelemetry.*                 (OTLP Protocol)     │ Write                │
│             │ ├── /opentelemetry.proto.collector.trace.v1          │ Ingest traces        │
│             │ │   .TraceService.Export                             │                      │
│             │ └── ...                                              │                      │
└─────────────┴──────────────────────────────────────────────────────┴──────────────────────┘
```

**Key points:**
- **Read operations** use the **query-frontend** component
- **Write operations** route to the **distributor** (OTLP)
- Both HTTP and gRPC protocols are supported for each operation type
- Multi-tenancy is enforced via `X-Scope-OrgID` header
- Same host can serve both read and write paths with different endpoints

## Implementation Steps

### 1. Configure Tempo Helm Values

The configuration is structured for the `tempo-distributed` Helm chart.

**Resources:**
- [GiantSwarm Tempo App Example](https://github.com/giantswarm/tempo-app#readme)
- [Tempo Helm Charts Getting Started](https://grafana.com/docs/helm-charts/tempo-distributed/next/get-started-helm-charts/)

**Key customization:**
- `gateway.enabled: false` - Gateway disabled; using direct ingress instead
  - **Note**: The gateway provides a unified HTTP interface supporting multiple protocols (Jaeger, Zipkin, etc.). See the [default gateway configuration](https://github.com/grafana/helm-charts/blob/tempo-distributed-1.48.0/charts/tempo-distributed/values.yaml#L2179-L2255) for all supported protocols.
  - This setup uses direct ingress to services, supporting **only Tempo read paths** (queries) and **OTLP write paths** (ingestion) via HTTP and gRPC.
  - If you need other protocols (Jaeger, Zipkin, etc.), either enable the gateway or create additional ingress routes to the distributor.

**Standard ports:**
- **4318** - OTLP HTTP endpoint (distributor)
- **4317** - OTLP gRPC endpoint (distributor)
- **3200** - Tempo internal HTTP API (query-frontend)
- **9095** - Tempo internal gRPC API (query-frontend)

### 2. Deploy Tempo

Deploy using the `tempo-distributed` Helm chart with your customized values:

```bash
# Add the Grafana Helm repository
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# Install or upgrade Tempo
helm upgrade --install tempo grafana/tempo-distributed \
  -f traces/tempo.values.grpc.yaml \
  -n tempo --create-namespace
```

### 3. Create Ingress Resources

After Tempo is deployed, create separate ingress resources to expose the services. We need two separate ingress resources because they use different protocols and routing mechanisms:

1. **HTTP Ingress** - Standard HTTP/1.1 traffic for REST API queries and OTLP ingestion (uses path prefix matching)
2. **gRPC Ingress** - HTTP/2 with gRPC protocol for native gRPC services (uses regex pattern matching)

The key differences:
- gRPC requires the `nginx.ingress.kubernetes.io/backend-protocol: "GRPC"` annotation to communicate with the backend service using HTTP/2 instead of HTTP/1.x, which is required for gRPC protocol
- gRPC requires the `nginx.ingress.kubernetes.io/use-regex: "true"` annotation to match gRPC service paths (e.g., `/tempopb.StreamingQuerier/SearchTagsV2`)
- HTTP uses standard path prefix matching without regex and HTTP/1.x for backend communication

Combining them in a single ingress is not possible due to these protocol and routing differences.

**HTTP Ingress** - Handles both Tempo queries and OTLP HTTP trace ingestion:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-production
  name: tempo
  namespace: tempo
spec:
  ingressClassName: nginx
  rules:
  - host: tempo.your-domain.com
    http:
      paths:
      # OTLP HTTP trace ingestion
      - backend:
          service:
            name: tempo-distributor
            port:
              number: 4318
        path: /v1/traces
        pathType: ImplementationSpecific
      # Tempo API queries
      - backend:
          service:
            name: tempo-query-frontend
            port:
              number: 3200
        path: /api
        pathType: ImplementationSpecific
  tls:
  - hosts:
    - tempo.your-domain.com
    secretName: tempo-ingress-cert
```

This ingress routes:
- `/api/v2/*` → Tempo query-frontend (search, trace lookup)
- `/v1/traces` → Tempo distributor (OTLP HTTP trace ingestion)

**gRPC Ingress** - Routes gRPC requests using path prefixes:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-production
    nginx.ingress.kubernetes.io/backend-protocol: "GRPC"
    nginx.ingress.kubernetes.io/use-regex: "true"
  name: tempo-grpc
  namespace: tempo
spec:
  ingressClassName: nginx
  rules:
  - host: tempo.your-domain.com
    http:
      paths:
      # OpenTelemetry gRPC requests
      - backend:
          service:
            name: tempo-distributor
            port:
              number: 4317
        path: /opentelemetry
        pathType: ImplementationSpecific
      # Tempo internal gRPC requests
      - backend:
          service:
            name: tempo-query-frontend
            port:
              number: 9095
        path: /tempopb
        pathType: ImplementationSpecific
  tls:
  - hosts:
    - tempo.your-domain.com
    secretName: tempo-grpc-ingress-cert
```

**Important**: While gRPC uses protobuf service definitions internally (like `/opentelemetry.proto.collector.trace.v1.TraceService`), NGINX ingress can route based on path prefixes. The paths `/opentelemetry` and `/tempopb` are conventional prefixes that match the beginning of the full gRPC service paths. The `use-regex` annotation enables pattern matching for routing.

Apply the ingress resources:

```bash
kubectl apply -f tempo-http-ingress.yaml
kubectl apply -f tempo-grpc-ingress.yaml
```

### 4. Verify Connectivity

Test both HTTP and gRPC endpoints to ensure they're working correctly.

#### Required Testing Tools

Install the following tools to test your Tempo endpoints:

**1. tempo-cli** - Tempo query CLI
```bash
go install github.com/grafana/tempo/cmd/tempo-cli@latest
```
- Repository: https://github.com/grafana/tempo
- Used for querying traces via HTTP and gRPC

**2. telemetrygen** - OpenTelemetry trace generator
```bash
go install github.com/open-telemetry/opentelemetry-collector-contrib/cmd/telemetrygen@latest
```
- Repository: https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/cmd/telemetrygen
- Used for generating and sending test traces via OTLP

**Testing Notes:**
- Replace `your-tenant-name` with your actual tenant identifier

#### Query Operations

**Test HTTP Query (Tempo protocol):**
```bash
tempo-cli query api search-tags \
  --org-id your-tenant-name \
  --secure \
  tempo.your-domain.com:443
```

Expected output (compact JSON):
```json
{"tagNames":["service.name","http.method","http.status_code","span.kind"]}
```

**Test gRPC Query (Tempo protocol):**
```bash
tempo-cli query api search-tags \
  --org-id your-tenant-name \
  --secure \
  --use-grpc \
  tempo.your-domain.com:443
```

Expected output (compact JSON):
```json
{"tagNames":["service.name","http.method","http.status_code","span.kind"]}
```

#### Write Operations

**Test HTTP Write (OTLP protocol):**
```bash
telemetrygen traces \
  --otlp-endpoint tempo.your-domain.com:443 \
  --otlp-http \
  --otlp-http-url-path /v1/traces \
  --traces 1 \
  --otlp-header 'X-Scope-OrgID="your-tenant-name"' \
  --otlp-attributes 'tenant="your-tenant-name"'
```

Expected output (successful):
```
2025-10-07T00:45:23.456Z	INFO	telemetrygen/main.go:123	Starting trace generator
2025-10-07T00:45:23.457Z	INFO	telemetrygen/traces.go:89	generation of traces isn't being throttled
2025-10-07T00:45:23.458Z	INFO	telemetrygen/traces.go:156	traces generated	{"worker": 0, "traces": 1}
2025-10-07T00:45:23.459Z	INFO	telemetrygen/main.go:145	stop the batch span processor
```

All lines should start with today's date otherwise they indicate errors.

**Test gRPC Write (OTLP protocol):**
```bash
telemetrygen traces \
  --otlp-endpoint tempo.your-domain.com:443 \
  --traces 1 \
  --otlp-header 'X-Scope-OrgID="your-tenant-name"' \
  --otlp-attributes 'tenant="your-tenant-name"'
```

Expected output (successful):
```
2025-10-07T00:45:23.456Z	INFO	telemetrygen/main.go:123	Starting trace generator
2025-10-07T00:45:23.457Z	INFO	telemetrygen/traces.go:89	generation of traces isn't being throttled
2025-10-07T00:45:23.458Z	INFO	telemetrygen/traces.go:156	traces generated	{"worker": 0, "traces": 1}
2025-10-07T00:45:23.459Z	INFO	telemetrygen/main.go:145	stop the batch span processor
```

Output should not contain WARN or ERR messages. Such messages indicate connection or export failures.

### 5. Configure Clients

Update your trace exporters to use the gRPC endpoints.

**Grafana Alloy** - Using `otelcol.exporter.otlp` component:

```hcl
otelcol.exporter.otlp "tempo" {
  client {
    endpoint = "tempo.your-domain.com:443"

    tls {
      insecure             = false
      insecure_skip_verify = false
    }

    headers = {
      "X-Scope-OrgID" = "your-tenant-name"
    }
  }
}

// Connect to your pipeline
otelcol.receiver.otlp "default" {
  grpc {
    endpoint = "0.0.0.0:4317"
  }

  output {
    traces = [otelcol.exporter.otlp.tempo.input]
  }
}
```

For more information on Alloy OTLP exporter configuration, see:
- [Official Alloy OTLP Exporter Documentation](https://grafana.com/docs/alloy/latest/reference/components/otelcol.exporter.otlp/)
- [GiantSwarm Alloy Configuration](https://github.com/giantswarm/logging-operator/blob/v0.33.0/pkg/resource/events-logger-config/alloy/events-logger.alloy.template) - Real-world Alloy configuration example

**Application SDKs:**
- Set OTLP exporter to gRPC mode
- Point to `tempo.your-domain.com:443`
- Ensure gRPC protocol is selected (not HTTP)
- Add `X-Scope-OrgID` header for multi-tenancy support

### 6. Configure Grafana Tempo Datasource

Configure Grafana to query traces and enable streaming. You can provision the datasource using a YAML configuration file.

**Example Tempo Datasource Configuration:**

```yaml
apiVersion: 1

datasources:
  - name: Tempo
    type: tempo
    uid: tempo
    access: proxy
    url: https://tempo.your-domain.com
    jsonData:
      httpMethod: GET
      tracesToLogsV2:
        # Optional: Configure trace to logs correlation
        datasourceUid: 'loki'
      serviceMap:
        datasourceUid: 'prometheus'
      nodeGraph:
        enabled: true
      search:
        hide: false
      lokiSearch:
        datasourceUid: 'loki'
      traceQuery:
        timeShiftEnabled: true
        spanStartTimeShift: '1h'
        spanEndTimeShift: '-1h'
      # Enable streaming for live trace updates
      tracesStreaming:
        enabled: true
```

For more information, see the [official Grafana Tempo datasource configuration documentation](https://grafana.com/docs/grafana/latest/datasources/tempo/configure-tempo-data-source/#provision-the-data-source).

**Important Caveats:**

1. **OAuth and Streaming Incompatibility**: OAuth authentication does not work with streaming enabled due to authorization headers not being correctly forwarded for gRPC requests. Use Basic Auth instead. See [grafana#110132](https://github.com/grafana/grafana/issues/110132) for details.

2. **gRPC over TLS Configuration**: The configuration for gRPC over TLS may be non-intuitive. Basic Auth must be enabled (even without it being configured) for gRPC to work over TLS, see [grafana#111770](https://github.com/grafana/grafana/pull/111770) for the upstream fix.

3. **WebSocket Requirement for Streaming**: Grafana uses WebSocket (`/api/live/ws`) for streaming trace and live updates. **WebSocket support is mandatory for streaming to work**. If you access Grafana through a proxy (like Teleport) that doesn't support WebSocket, streaming features will not work. For Teleport-specific guidance, see [teleport/discussions/8821#discussioncomment-14553239](https://github.com/gravitational/teleport/discussions/8821#discussioncomment-14553239).

## Important Notes

- **Path-based routing for gRPC**: NGINX can route gRPC traffic using path prefixes (e.g., `/opentelemetry`, `/tempopb`). These prefixes match the beginning of the full gRPC service paths defined in protobuf (e.g., `/opentelemetry.proto.collector.trace.v1.TraceService/Export`).
- **No path rewriting**: Unlike HTTP, gRPC paths cannot be rewritten. The paths `/opentelemetry` and `/tempopb` are conventions used by the services and must match what the gRPC services expect.
- **NGINX annotations required**:
  - `nginx.ingress.kubernetes.io/backend-protocol: "GRPC"` - Tells NGINX to communicate with the backend service using HTTP/2 (required for gRPC)
  - `nginx.ingress.kubernetes.io/use-regex: "true"` - Enables pattern matching for routing
- **Client configuration**: gRPC clients connect to the host endpoint (e.g., `tempo.your-domain.com:443`) and the gRPC framework automatically includes the service path in requests.
- **Both protocols supported**: HTTP and gRPC can coexist on the same hostname with different ingress resources.

## Additional Resources

- [Grafana Alloy Documentation](https://grafana.com/docs/alloy/latest/)
- [Alloy OTLP Exporter Component Reference](https://grafana.com/docs/alloy/latest/reference/components/otelcol.exporter.otlp/)

## Conclusion

That's it! You now have Tempo running with gRPC support, enabling faster trace ingestion and real-time streaming in Grafana. If you run into any issues, check the caveats section above or reach out to the community. Happy tracing!

## Disclaimer

Please review the code before running it in production
