Title
----
* Author(s): Zhiyan Foo
* Approver: a11r
* Status: Draft
* Implemented in: TBD
* Last updated: 2024-11-06
* Discussion at: TBD (filled after thread exists)

## Abstract

Propose extending gRPC's xDS client to honour `host_rewrite_literal` in route configuration for
authority header rewriting when `trusted_xds_server` is set.

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
* [A81: `auto_host_rewrite` in gRPC xDS](https://github.com/grpc/proposal/blob/master/A81-xds-routes.md)
* [A86: xDS-Based HTTP CONNECT](https://github.com/grpc/proposal/pull/455)

## Proposal

Implement `host_rewrite_literal` feature in gRPC xDS client, enabling explicit authority header
rewrites. This should be conditioned on `trusted_xds_server`.

### Temporary environment variable protection

Feature guarded by `GRPC_XDS_EXPERIMENTAL_AUTHORIY_LITERAL_REWRITE`, disabled by default.

## Rationale

Alternative approaches considered.
- Exclusively use the existing `auto_host_rewrite` feature. This would require a CDS cluster per
  upstream target, which mean that connections would not be re-used between gRPC xDS client and
  the proxy.
- Using the HTTP CONNECT protocol has the same drawback in that connections to the proxy that
  target different upstream services won't reuse the same connection.
   - in the case where instead of a single proxy in between the target upstream service there
     are two proxies (e.g. client -> egress proxy -> ingress proxy), this would not only prevent
     connection reuse between the initial client to the proxy, but also between the intermediary
     proxies (e.g. between the egress and ingress proxy).

Even if connection pooling was not a major concern, this addition will bring gRPC xDS client
functionality closer to parity with Envoy's capabilities, simplifying configurations for users
migrating to or using both systems concurrently.


## Implementation

We can contribute an implementation of this to golang. From our initial drafts, the changes
required to support this are minimal.
