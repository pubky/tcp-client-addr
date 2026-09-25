# TCP client address

This crate identifies the client address for an accepted TCP connection. It
does not serve HTTP or apply rate limits.

- `IdentityMode::Direct` uses the TCP peer address and ignores HTTP headers.
- `IdentityMode::ProxyProtocol` reads a PROXY v1 or v2 preface from a trusted
  immediate peer and uses its source address. A missing or invalid preface is
  an error; it never falls back to the peer address.

Both modes return the original `TcpStream` at the first application byte,
along with the client and immediate peer addresses. No application bytes are
buffered or consumed.

## Why use PROXY protocol?

An HTTP forwarding header is easy for a client to forge unless every proxy
that accepts it strips or replaces untrusted values. For both HTTP and HTTPS,
the application must establish whether it is handling HTTP/1.1 or HTTP/2 and
parse the request headers before it can read the forwarded address. HTTPS also
requires a TLS handshake first.
A proxy passing through end-to-end TLS cannot add or change HTTP headers.

PROXY protocol carries the client address before application data. With TLS
pass-through, it also arrives before the backend TLS handshake. The application
can identify the client as soon as it accepts the connection. This is safe
only when the listener accepts PROXY prefaces from trusted immediate peers: a
trusted peer can claim any source address.

If the application terminates TLS, accept TCP, call `identify`, perform the
TLS handshake on the returned stream, then serve the application protocol.

## Example

```rust,no_run
use tcp_client_addr::{IdentityMode, ProxyProtocol};
use ipnet::IpNet;
use tokio::net::TcpListener;

# async fn example() -> Result<(), Box<dyn std::error::Error>> {
let listener = TcpListener::bind("127.0.0.1:8080").await?;
let trusted_proxy: IpNet = "127.0.0.1/32".parse()?;
let mode = IdentityMode::ProxyProtocol(ProxyProtocol::new([trusted_proxy])?);

let (stream, _) = listener.accept().await?;
let (stream, addr) = mode.identify(stream).await?;
println!("client: {}, immediate peer: {}", addr.client(), addr.peer());
println!("rate-limit IP: {}", addr.client_ip());
// Pass `stream` to the TLS or application server.
drop(stream);
# Ok(())
# }
```

With Nginx on the same host, add one of these `stream` blocks to its
configuration. For TLS pass-through, the application terminates TLS:

```nginx
stream {
    server {
        listen 443;
        proxy_pass 127.0.0.1:8080;
        proxy_protocol on;
    }
}
```

Nginx passes TLS through and sends a PROXY v1 preface before the TLS bytes.
The Rust listener above reads that preface, then passes the stream to its TLS
server.

To terminate TLS at Nginx instead, enable the [stream SSL module][nginx-ssl]:

```nginx
stream {
    server {
        listen 443 ssl;
        ssl_certificate /etc/nginx/certs/server.crt;
        ssl_certificate_key /etc/nginx/certs/server.key;
        proxy_pass 127.0.0.1:8080;
        proxy_protocol on;
    }
}
```

Nginx sends the PROXY preface followed by decrypted application bytes. The
Rust listener calls `identify`, then serves plaintext HTTP or another
application protocol without a second TLS handshake.

If Nginx runs on another host, point `proxy_pass` at the application's private
address and trust only Nginx's source IP.

`ClientAddr::client()` preserves the received address. For per-IP decisions,
use `ClientAddr::client_ip()`. If a server needs a socket address, use
`ClientAddr::normalized_client()`. The latter two methods convert IPv4-mapped
IPv6 addresses to IPv4 and preserve native IPv6 addresses.

## Proxy listener requirements

- Restrict the listener to trusted proxies with a firewall or private network.
  A trusted CIDR authorizes every peer in it to claim any client address. Even
  a loopback CIDR trusts every local process that can connect.
- In a proxy chain, trust the immediate peer that sends the PROXY preface. The
  first proxy must establish the real client address, not copy a client-supplied
  forwarding header.
- Keep one backend TCP connection per client. The identity applies to the
  whole connection, including every HTTP request or HTTP/2 stream on it.
  Nginx `stream` proxying does this by default. Do not share an upstream
  connection between clients.

Proxy mode rejects v1 `UNKNOWN` and v2 `LOCAL` because they provide no TCP
client address. Configure proxy health checks accordingly. PROXY v1 has a
107-byte limit; `ProxyProtocol::with_max_header_bytes` can impose a lower
limit and also sets the v2 limit. V2 bytes after the address block are consumed
but ignored, including TLVs and optional CRC32C checksums.

A complete preface must arrive within one second by default. The timer starts
when `identify` begins reading. Use `ProxyProtocol::with_header_timeout` to
change it. Also limit concurrent connection setup tasks and, if needed, apply
a deadline from connection acceptance.

See the [PROXY protocol specification][proxy-spec].

[proxy-spec]: https://www.haproxy.org/download/2.9/doc/proxy-protocol.txt
[nginx-ssl]: https://nginx.org/en/docs/stream/ngx_stream_ssl_module.html

## Tests

Run `cargo test`. The optional Nginx integration test checks the forwarded
address and untouched application bytes:

```bash
cargo test --test nginx_proxy -- --ignored
```

It requires Docker, a local `nginx:alpine` image, and Docker host networking.
The test starts and removes its own container.
