# RPCEdge Relay Client

Rust client and shared protocol types for RPCEdge transaction relay.

This repository is intentionally separate from the private Polaris monorepo.
The intended upstream is:

```text
https://github.com/rpc-edge/rpcedge-relay-client.git
```

Push access is required before Polaris can pin this client by Git revision.

## Current Status

Public Rust client and shared protocol types:

- `rpcedge-relay-protocol`: public method, route, route-set, HTTP envelope, and
  QUIC frame types.
- `rpcedge-relay-client`: async HTTP client for route-aware
  `POST /v1/submit`, compatibility JSON-RPC `POST /v1/sendTransaction`, raw
  HTTP `POST /v1/transactions`, and route-aware QUIC single-transaction
  submission.

Public launch scope is single-transaction submission:

- `sendTransaction`
- `sendTransactionFast`

`sendBundle` is represented in the protocol for Polaris internal use only. The
server must deny it for public standard and premium keys.

## HTTP Usage

```rust
use rpcedge_relay_client::{RelayClient, RelayClientConfig, RouteSet};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let client = RelayClient::new(RelayClientConfig::new(
        "https://relay.rpcedge.com",
        "00000000-0000-4000-8000-000000000000",
    ))?;

    let response = client
        .send_transaction_base64("BASE64_TRANSACTION", RouteSet::server_default())
        .await?;

    println!("accepted={} signature={}", response.accepted, response.signature);
    Ok(())
}
```

The client sends both `Authorization: Bearer <api-key>` and `x-api-key:
<api-key>` headers. The endpoint may be either a base URL such as
`https://relay.rpcedge.com` or the full submit URL
`https://relay.rpcedge.com/v1/submit`.

## Route Sets

```rust
use rpcedge_relay_protocol::{RelayRoute, RouteSet};

let route_set = RouteSet::default_plus([RelayRoute::JitoBundle]);
```

The server resolves route sets after authentication and policy lookup. A client
request for an unauthorized route must fail explicitly; it must not silently
fall back to server defaults.

## Transports

- HTTP JSON envelope: `POST /v1/submit`
- Raw HTTP compatibility: `POST /v1/transactions`
- JSON-RPC HTTP compatibility: `POST /v1/sendTransaction`
- QUIC: persistent QUIC connection with one bidirectional stream per
  transaction. The stream begins with `api-key: <key>\n`, followed by the
  framed `QuicSubmitHeader` plus raw transaction payload. The header carries
  `request_id` and `route_set`, so QUIC route selection is now equivalent to
  HTTP `/v1/submit`. The server keeps the legacy
  `api-key: <key>\n<raw_tx>` fallback for old clients, but legacy raw QUIC
  always uses server defaults.

QUIC verifies the server name against the public WebPKI root set by default.
There is no certificate-verification bypass. During a private-CA migration,
use `with_additional_root_certificate` or
`with_additional_root_certificates_pem` to overlap the old and new trust roots.

Route-aware QUIC example:

```rust
use rpcedge_relay_client::{QuicRelayClient, QuicRelayClientConfig};
use rpcedge_relay_protocol::{RelayRoute, RouteSet};

# async fn example(raw_tx: Vec<u8>) -> Result<(), Box<dyn std::error::Error>> {
let endpoint = tokio::net::lookup_host(("relay.rpcedge.com", 4433))
    .await?
    .next()
    .ok_or("relay.rpcedge.com did not resolve")?;
let client = QuicRelayClient::connect(QuicRelayClientConfig::new(
    endpoint,
    "00000000-0000-4000-8000-000000000000",
))
.await?;

let response = client
    .send_transaction_raw_bytes_with_route_set(raw_tx, RouteSet::only([RelayRoute::TpuQuic]))
    .await?;

println!("accepted={} signature={}", response.accepted, response.signature);
# Ok(())
# }
```

See the Polaris architecture docs for the rollout plan.
