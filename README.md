# iron-proxy snap

A [snap](https://snapcraft.io) that builds and packages
[iron-proxy](https://github.com/paradigmxyz/iron-proxy) from source and runs it
as a system service. iron-proxy is a MITM egress proxy with a built-in DNS
server: it enforces default-deny egress, swaps proxy tokens for real secrets at
the boundary, and writes a per-request audit trail.

See [docs.iron.sh](https://docs.iron.sh) for the product documentation.

## Install

```sh
sudo snap install iron-proxy
```

The snap is strictly confined. It uses the `network` and `network-bind`
interfaces, which are auto-connected on install, so the proxy can listen on
privileged ports and make outbound connections. The service starts
automatically and keeps running (`restart-condition: on-failure`).

## Configuration

The configuration file is `$SNAP_COMMON/iron-proxy.yaml`, i.e.
`/var/snap/iron-proxy/common/iron-proxy.yaml`. It is created from a default
template on first start and is never overwritten on refresh, so your edits
persist. After editing it, apply the change:

```sh
sudo snap restart iron-proxy
```

The default configuration ships iron-proxy in its intended default-deny mode:
every outbound request is rejected (403) until you add `domains` or `cidrs` to
the `allowlist` transform. Set `warn: true` on that transform to log what would
be blocked without enforcing it while you build out the allowlist.

On first start the snap also generates a certificate authority at
`$SNAP_COMMON/ca.crt` / `$SNAP_COMMON/ca.key`, which iron-proxy uses to sign the
leaf certificates it mints for TLS interception (MITM). Clients proxied through
iron-proxy must trust `ca.crt`:

```sh
sudo cp /var/snap/iron-proxy/common/ca.crt /usr/local/share/ca-certificates/iron-proxy.crt
sudo update-ca-certificates
```

### Ports

| Listener | Default | Purpose |
|---|---|---|
| `proxy.http_listen` | `:80` | HTTP traffic routed to the proxy |
| `proxy.https_listen` | `:443` | HTTPS traffic routed to the proxy (MITM) |
| `proxy.tunnel_listen` | `:8080` | Explicit HTTP/CONNECT/SOCKS5 proxy (`HTTP_PROXY`, `ALL_PROXY`, ...) |
| `dns.listen` | `:53` | Built-in DNS server (disabled by default) |

DNS interception is disabled by default because the built-in server answers
every lookup with `dns.proxy_ip`, which must be the address clients use to reach
this host. Set `dns.enabled: true` and `dns.proxy_ip` to use it.
