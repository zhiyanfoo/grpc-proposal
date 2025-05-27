Title
----
* Author(s): Zhiyan Foo
* Approver: a11r
* Status: Draft
* Implemented in: TBD
* Last updated: 2025-04-23
* Discussion at: TBD (filled after thread exists)

## Abstract

Implement the 
[`host_rewrite_literal`](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/route/v3/route_components.proto#envoy-v3-api-field-config-route-v3-routeaction-host-rewrite-literal) 
route_action feature in gRPC xDS client, enabling explicit authority header rewrites. This will
be conditioned on `trusted_xds_server` as described in [gRFC A81][A81].


## Background

gRPC xDS currently supports `auto_host_rewrite` as per [gRFC A81][A81] but lacks `host_rewrite_literal`
support, useful when a single cluster targets a reverse proxy routing based on the authority
header.
```
            +------------------+
            |  gRPC xDS Client |  <- (RDS configuration sets `host_rewrite_literal` to
            +------------------+      target one of Svc A/B/C)
                     |
                     |    <- (gRPC xDS client reuses the same CDS target for all connections to
                     |        the proxy)
                 +---------+
                 |  Proxy  |
                 +---------+
                      |   <- (Proxy routing logic based on Authority header)
      .---------------+--------------.
     /                |               \
+--------+       +--------+       +--------+
| Svc A  |       | Svc B  |       | Svc C  |
+--------+       +--------+       +--------+
```

### Related Proposals:
* [gRFC A29: xDS-Based Security for gRPC Clients and Servers][A29]
* [gRFC A81: xDS Authority Rewriting][A81]
* [gRFC A86: xDS-Based HTTP CONNECT][A86]


## Proposal

This proposal has the following parts:
- xDS resource unmarshalling: We will retrieve the `host_rewrite_literal` value from rds. The
  protobuf option for authority rewriting is already set as a one-of, so there's no issue of
  deciding precedence of the different authority writing options.

  If the `xds_trusted_server` attribute is set to false, the `host_rewrite_literal` field will
  be ignored.
- xDS ConfigSelector: We will propagate the `host_rewrite_literal` configuration to the
  configSeletor where it would be stored in the rpc context. 
- During the stream creation, and the RPC call attributes are being set, if there is a non-empty
  value for the `host_rewrite_literal`, it will take precedence over other options for the RPC
  Call Host attribute.


### Temporary environment variable protection

Feature guarded by `GRPC_XDS_EXPERIMENTAL_AUTHORIY_LITERAL_REWRITE`, disabled by default.

## Rationale

Alternative approaches considered.
- Exclusively use the existing [`auto_host_rewrite`][route_action] feature. This would require a
  CDS cluster per upstream target, which means that there would be one connection per service
  behind the proxy between each gRPC clients and the proxy.
- Using the HTTP CONNECT protocol has the same drawback in that connections to the proxy that
  target different upstream services won't reuse the same connection.
   - in the case where instead of a single proxy in between the client and target upstream
     service there are two proxies (e.g. client -> egress proxy -> ingress proxy -> svc), this
     would not only prevent connection reuse between the initial client to the proxy, but also
     between the intermediary proxies (e.g. between the egress and ingress proxy).

Even if connection pooling was not a major concern, this addition will bring gRPC xDS client
functionality closer to parity with Envoy's capabilities, simplifying configurations for users
migrating to or using both systems concurrently.

## Implementation

Go implementation[insert here]().

[A29]: A29-xds-tls-security.md
[A81]: A81-xds-authority-rewriting.md
[A86]: https://github.com/grpc/proposal/pull/455
[route_action]: https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/route/v3/route_components.proto
