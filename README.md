# uos-server-mobile-direct

If you're running the self-hosted UniFi OS Server and the UniFi Network mobile app keeps going through Ubiquiti's cloud relay - even when your phone is sitting on the same LAN as your controller - this repo explains why that happens and how to fix it with a reverse proxy.

Ubiquiti hardware consoles (UDM family, UCG-Max / UCG-Ultra / UCG-Fiber, Cloud Key Gen2 Plus) don't have this problem. (Gateway-only devices like the UXG-Fiber don't count here - they're adopted by a separate console rather than running the Network app themselves.) The mobile app direct-connects to them on-LAN, or across a site-to-site VPN, without touching the cloud. The self-hosted UOS Server behaves differently, and it's not immediately obvious why.

## How the mobile app actually finds your controller

The mechanism is pure DNS plus HTTPS, so it's platform-identical between iOS and Android (I verified on iOS with pcap on 2026-04-19; Android is expected to behave the same). Here's what happens when you tap a site in the app:

1. The app issues DNS queries (A and HTTPS/Type65 records) for `<console-id>.id.ui.direct`. That's a hostname in a public Ubiquiti-owned zone hosted on AWS Route53 (SOA `ns-*.awsdns-*.co.uk`).
2. If DNS returns an A record, the phone opens a TCP connection to that IP on port 443 and sends a TLS ClientHello with `SNI = <console-id>.id.ui.direct`.
3. After the handshake completes, the app sends an HTTP/2 probe: `GET /api/system`.
4. If the probe returns a valid response, the app commits to direct-connect for that session. If anything fails (NXDomain, unreachable IP, bad TLS, a 404 on `/api/system`), it falls back to cloud relay.

### Why hardware consoles just work

Hardware consoles auto-register their LAN IP into the `id.ui.direct` zone during cloud adoption. Any client anywhere can resolve the public DNS and get back an RFC1918 address like 192.168.1.1. If you're on that LAN (or reachable via site-to-site VPN), you direct-connect. If not, you don't, and the app falls back. Simple.

### Why self-hosted UOS Server doesn't

The self-hosted UniFi OS Server never registers itself in that zone. You can confirm with a quick dig from anywhere:

```bash
dig @1.1.1.1 <your-console-id>.id.ui.direct A +short
# (empty - NXDomain)
```

Without a DNS answer, the phone has no IP to connect to, and the app skips straight to cloud relay. There's also a port mismatch to be aware of: the self-hosted UOS Server serves its Network app on port 11443 by default, while the mobile app probes port 443. If your UOS Server is configured to listen on 443 directly (dedicated host, nothing else bound to 443), you don't need a reverse proxy at all - just the DNS entry pointing at the UOS Server's LAN IP. More commonly, UOS shares a host with other services that already occupy 443, which is why this repo focuses on the reverse-proxy path.

## The fix

Two pieces, both straightforward. If you're in the lucky case where UOS is already on port 443, you can stop after step 1.

### 1. Local DNS override

The only requirement is that when your phone resolves `<console-id>.id.ui.direct`, it gets back your reverse proxy's LAN IP:

```
<console-id>.id.ui.direct  ->  <reverse-proxy LAN IP>
```

Where that override lives does not matter, as long as it sits somewhere in the resolution path your phone uses. Any of these will work:

- Your phone's DHCP-assigned resolver (usually your router) with a built-in override. UniFi Network has this under Settings > Routing > DNS as host records.
- A LAN resolver you explicitly point your phone at: Pi-hole, AdGuard Home, Unbound, dnsmasq, BIND. Pick one.
- A layered setup where one resolver forwards the hostname (or the whole zone) to another resolver that holds the override. That's what I happen to use (Pi-hole delegating to UniFi Network DNS, where the A record lives) but it's not required. A single resolver with a single override does the job.

What won't work is leaving your phone on 1.1.1.1, 8.8.8.8, or any other public resolver. Those always return the authoritative NXDomain, and direct-connect never triggers.

### 2. Traefik HTTP reverse proxy

Bridge port 443 (what the app hits) to port 11443 (where UOS Server actually listens).

```yaml
http:
  routers:
    unifi-direct:
      rule: "Host(`<console-id>.id.ui.direct`)"
      entryPoints:
        - websecure
      service: unifi
      middlewares:
        - lan-restricted
      tls: {}

  services:
    unifi:
      loadBalancer:
        servers:
          - url: "https://<uos-server-lan-ip>:11443"
        serversTransport: unifi-skipverify

  serversTransports:
    unifi-skipverify:
      insecureSkipVerify: true

  middlewares:
    lan-restricted:
      ipAllowList:
        sourceRange:
          - 192.168.0.0/16
          - 10.0.0.0/8
          - 172.16.0.0/12
```

A few notes on what's going on here.

`tls: {}` uses Traefik's default self-signed cert. Let's Encrypt can't issue a cert for the `.ui.direct` zone (you don't own it), and as it turns out you don't need one - the iOS app accepted Traefik's self-signed cert without any cert install on the phone. Android should behave the same, but I haven't independently verified.

`insecureSkipVerify: true` on the upstream transport lets Traefik connect to the UOS Server without validating its cert, since UOS serves a self-signed `unifi.local` cert by default.

The `lan-restricted` middleware matters more than it looks. The hostname you're overriding is publicly resolvable - anyone who knows your console ID can look it up. Without a LAN allowlist, someone could point their own DNS at your reverse proxy and probe it from the internet. Limit it to LAN source ranges and you're covered.

### Finding your console ID

Launch the mobile app on your LAN while capturing DNS on a gateway the phone traverses:

```bash
tcpdump -i any -nn 'host <phone-ip> and port 53' 2>&1 | grep 'id.ui.direct'
```

You'll see a query like:

```
... A? <your-console-id>.id.ui.direct.
```

The hex string is your console ID. Plug it into the DNS override and the Traefik host rule.

## Other reverse proxies should work too

I haven't personally tested any of these, but the requirements are clear enough that they should all work:

- nginx with `server { listen 443 ssl; server_name <console-id>.id.ui.direct; ... }`, upstream to `https://<uos-ip>:11443` with `proxy_ssl_verify off`.
- Caddy with a site block for `<console-id>.id.ui.direct` that `reverse_proxy`s to `https://<uos-ip>:11443`, `tls_insecure_skip_verify` on the transport, and `tls internal` on the listener.
- HAProxy with an HTTP frontend keyed on host or SNI, backend pointing to UOS on 11443 with `verify none`.

Any implementation needs to do four things: accept TLS on port 443 for `SNI = <console-id>.id.ui.direct`, serve some cert (self-signed is fine), proxy HTTP/2 through to the UOS Server on port 11443 with upstream cert verification off, and restrict access to LAN sources.

## What won't work

Pointing DNS directly at the UOS Server's LAN IP with no reverse proxy, if UOS is running on its default port 11443. The app hits port 443 and nothing answers. If you've moved UOS to port 443 directly, though, the DNS entry alone is sufficient.

Layer-4 TLS passthrough (`stream { ssl_preread }` in nginx, TCP routers in Traefik). Not wrong - it would work - but it's unnecessary. The app doesn't validate the upstream cert's chain to whatever cert the UOS Server originally registered, so plain HTTPS termination at the reverse proxy is enough.

Pointing DNS at any reverse proxy that returns 404 (or anything else invalid) for `GET /api/system`. The probe has to succeed or the app gives up and goes back to cloud relay.

## License

MIT. See LICENSE.
