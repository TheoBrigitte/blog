---
title: "Mimir Multi-Tenant Authentication & Authorization"
date: 2025-10-01T14:00:00+02:00
description: "Implementing secure multi-tenant access control in Grafana Mimir using NGINX and Perl"
---

Securing multi-tenant Mimir deployments requires robust authentication and authorization. This post demonstrates how to implement tenant-level access control using the Mimir Gateway's native NGINX capabilities.

## The Challenge

Multi-tenant Mimir deployments face several security challenges:

- **Tenant Isolation**: Ensuring tenants can only access their own metrics and alerting rules
- **Scalable Access Control**: Managing authentication and authorization for potentially hundreds of users across multiple tenants
- **Operational Overhead**: Minimizing the complexity of user and tenant management
- **Security**: Protecting sensitive observability data from unauthorized access

## Solution Overview

The approach leverages the **Mimir Gateway's native NGINX capabilities** to implement a robust authentication and authorization layer that:

- ✅ Integrates seamlessly with existing Mimir deployments
- ✅ Provides fine-grained tenant-level access control
- ✅ Supports dynamic configuration updates without service disruption
- ✅ Scales efficiently for enterprise deployments
- ✅ Maintains security best practices with industry-standard authentication

**Architecture Assumption**: All external traffic to Mimir is routed through the Mimir Gateway.

## Architecture

### High-Level Components

```
┌─────────────────┐    ┌─────────────────────────────────────────┐    ┌─────────────────┐
│   External      │    │           Mimir Gateway                 │    │   Mimir Cluster │
│   Clients       │───▶│  ┌─────────────────────────────────────┐│───▶│                 │
│                 │    │  │   NGINX          Authentication &   ││    │  ┌───────────┐  │
│ • Grafana       │    │  │                  Authorization      ││    │  │ Queriers  │  │
│ • Prometheus    │    │  │ ┌─────────┐                         ││    │  │           │  │
│ • AlertManager  │    │  │ │ Basic   │     ┌─────────────────┐ ││    │  │ Ingesters │  │
│ • mimirtool     │    │  │ │ Auth    │     │ Tenant Access   │ ││    │  │           │  │
└─────────────────┘    │  │ └─────────┘     │ Control (Perl)  │ ││    │  │ Rulers    │  │
                       │  │ ┌─────────┐     └─────────────────┘ ││    │  └───────────┘  │
                       │  │ │ Config  │   ┌────────────────────┐││    └─────────────────┘
                       │  │ │ reloader│   │ Authorized Tenants │││
                       │  │ └─────────┘   │ Mapping            │││
                       │  │               └────────────────────┘││
                       │  └─────────────────────────────────────┘│
                       └─────────────────────────────────────────┘
```

### Component Responsibilities

| Component | Responsibility |
|-----------|----------------|
| **NGINX Basic Auth** | Validates user credentials using htpasswd format |
| **Config Reloader** | Monitors configuration changes and triggers NGINX reload |
| **Tenant Access Control** | Perl-based authorization logic matching users to allowed tenants |
| **Authorized Tenants Mapping** | File-based mapping of users to their permitted tenant list |

## Security Model

The security model follows **least privilege** and **defense in depth** principles:

1. **Deny by Default**: All requests are rejected unless explicitly authenticated and authorized
2. **Explicit Authorization**: Users must be explicitly granted access to specific tenants
3. **Multi-Tenant Isolation**: Cross-tenant access requires explicit permission for each tenant

### Authentication Flow

- Each request is authenticated by verifying the username and password provided in the `Authorization` HTTP header using **HTTP Basic Authentication**
- Credentials are validated against an htpasswd-format file
- If credentials are invalid or missing, the request is rejected with a `401 Unauthorized` status code
- If authentication succeeds, the request proceeds to the authorization phase

### Authorization Flow

- Authenticated users must be explicitly authorized to access requested tenant(s)
- Tenants are specified via the `X-Scope-OrgID` HTTP header as a pipe-separated list (e.g., `X-Scope-OrgID: tenant1|tenant2`)
- **All requested tenants must be in the user's authorized tenant list** for the request to succeed
- If any requested tenant is not authorized, the request is rejected with a `403 Forbidden` status code
- **Requests without tenant specification are rejected** to prevent accidental cross-tenant access

### Design Rationale

**Why NGINX + Perl over alternatives?**
- **Performance**: NGINX handles high-throughput traffic efficiently
- **Integration**: Native integration with existing Mimir Gateway
- **Flexibility**: Perl scripting allows complex authorization logic
- **Reliability**: Battle-tested components with extensive documentation
- **Operational Simplicity**: Minimal additional infrastructure required

## Implementation

### 1. Authentication

Basic authentication is already configurable via the Mimir Gateway Helm chart.

Create user credentials using the `htpasswd` command:

```sh
$ htpasswd -ci .htpasswd admin < adminpassword_file
Adding password for user admin
$ htpasswd -i .htpasswd foo < foopassword_file
Adding password for user foo
$ htpasswd -i .htpasswd bar < barpassword_file
Adding password for user bar
```

Create a Kubernetes secret:

```sh
$ kubectl create secret generic mimir-gateway-basic-auth-secret \
  --from-file .htpasswd
```

Enable basic authentication in the Helm chart `values.yaml`:

```yaml
mimir:
  gateway:
    nginx:
      basicAuth:
        enabled: true
        existingSecret: mimir-gateway-basic-auth-secret
```

### 2. Authorization

Create a user-to-tenants mapping file:

```sh
cat <<EOF > authorized_tenants.map
admin tenant1|tenant2|tenant3;
foo tenant1|tenant2;
bar tenant3;
EOF
```

Create a Kubernetes secret:

```sh
$ kubectl create secret generic mimir-gateway-authorized-tenants \
  --from-file authorized_tenants.map
```

Add the authorization logic to the Helm chart:

```yaml
mimir:
  gateway:
    nginx:
      config:
        # Enable perl module in nginx for tenant access control
        errorLogLevel: |-
          error;
          load_module modules/ngx_http_perl_module.so
        httpSnippet: |
          # Match basic auth user to its configured tenant IDs
          map $remote_user $authorized_tenants {
              include /etc/nginx/authorized-tenants/authorized_tenants.map;
          }

          # Restrict access to tenants based on X-Scope-OrgID header
          perl_set $tenant_access_authorized '
              sub {
                my $r = shift;

                # Health check endpoints bypass authorization
                return 1 if $r->variable("uri") eq "/";
                return 1 if $r->variable("uri") eq "/ready";

                # Split the pipe-separated strings into arrays
                my @authorized_tenants_array = split /\|/, $r->variable("authorized_tenants");
                my @requested_ids = split /\|/, $r->variable("http_x_scope_orgid");

                # Deny access if no tenants are requested
                return 0 if !@requested_ids;

                # Create a hash for fast lookups
                my %authorized_tenants_hash = map { $_ => 1 } @authorized_tenants_array;

                # Check if all requested tenants are in the authorized tenants
                foreach my $element (@requested_ids) {
                    unless (exists $authorized_tenants_hash{$element}) {
                        return 0;
                    }
                }

                return 1;
              }
          ';
        serverSnippet: |
          if ($tenant_access_authorized = 0) {
              return 403 'Forbidden';
          }
      image:
        # Use an NGINX image with Perl module enabled
        registry: docker.io
        repository: nginxinc/nginx-unprivileged
        tag: 1.28-alpine-perl
    extraVolumes:
    - name: mimir-gateway-authorized-tenants
      secret:
        secretName: mimir-gateway-authorized-tenants
    extraVolumeMounts:
    - name: mimir-gateway-authorized-tenants
      mountPath: /etc/nginx/authorized-tenants
      readOnly: true
```

### 3. Automatic Config Reload

To automatically reload NGINX when secrets change, use `inotifyd` to watch for file changes.

Create the watcher scripts:

```sh
# config-watcher.sh
#!/bin/sh

inotifyd /usr/bin/nginx-reload.sh \
  /etc/nginx/authorized-tenants/authorized_tenants.map:c \
  /etc/nginx/secrets/.htpasswd:c &
```

```sh
# nginx-reload.sh
#!/bin/sh

nginx -s reload
```

Create a ConfigMap:

```sh
$ kubectl create configmap mimir-gateway-reload \
  --from-file=config-watcher.sh \
  --from-file=nginx-reload.sh
```

Mount the ConfigMap and enable the watcher:

```yaml
mimir:
  gateway:
    extraVolumes:
    - name: mimir-gateway-reload
      configMap:
        name: mimir-gateway-reload
        defaultMode: 0755
    extraVolumeMounts:
    - name: mimir-gateway-reload
      mountPath: /docker-entrypoint.d/config-watcher.sh
      subPath: config-watcher.sh
    - name: mimir-gateway-reload
      mountPath: /usr/bin/nginx-reload.sh
      subPath: nginx-reload.sh
```

## Testing

Test the authentication and authorization:

```sh
# Port-forward the Mimir Gateway
$ kubectl -n mimir port-forward svc/mimir-gateway 8080

# Test with mimirtool
$ mimirtool rules list \
  --address=http://127.0.0.1:8080 \
  --user admin \
  --password strongpassword \
  --id tenant1

# Test with curl
$ curl -i -u admin:strongpassword \
  -H "X-Scope-OrgID: tenant1|tenant2" \
  http://localhost:8080/prometheus/api/v1/query \
  --data-urlencode "query=ALERTS"
```

## Caveats

- All access data is loaded in memory by NGINX, which might not scale well with a large number of users and tenants
- The config reload script runs in the background without a process manager - consider using [s6-overlay](https://github.com/just-containers/s6-overlay) for production
- The NGINX `load_module` directive for Perl is injected via the `errorLogLevel` setting, which might break in future Helm chart versions

## Conclusion

This approach provides a flexible and scalable way to manage multi-tenant access control in Mimir, leveraging existing NGINX capabilities without requiring additional infrastructure. By combining basic authentication with Perl-based authorization logic, you can ensure that only authorized users can access their permitted tenants while maintaining operational simplicity.

Et voilà! Doing a bit of Perl and NGINX was fun after all :)

---

Full implementation details and configuration files are available in the [pull request](https://github.com/giantswarm/observability-operator/pull/532).
