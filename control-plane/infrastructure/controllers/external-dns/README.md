# ExternalDNS

This component deploys two ExternalDNS instances from one shared HelmRelease base.

| HelmRelease | Provider | Gateway filter | Managed view |
|---|---|---|---|
| `external-dns-internal` | RFC2136 | `gateway.sbrtech.xyz/exposure=internal` | Technitium internal DNS |
| `external-dns-public` | Cloudflare | `gateway.sbrtech.xyz/exposure=public` | Public Cloudflare DNS |

The internal instance manages both `svc.int.sbrtech.xyz` and the Technitium internal view of `sbrtech.xyz`. The RFC2136 provider selects the most-specific configured zone for each endpoint. Both zones intentionally share the existing TSIG identity and TXT owner ID because they use the same Technitium backend and operational lifecycle.

The public instance publishes application records beneath `sbrtech.xyz`. The public Gateway must carry:

```yaml
external-dns.alpha.kubernetes.io/target: lab-edge.sbrtech.xyz
```

The UDM Pro dynamic-DNS integration owns the public `lab-edge.sbrtech.xyz` A record. ExternalDNS owns application CNAMEs that target it.

## Prerequisites

- Label the internal Gateway with `gateway.sbrtech.xyz/exposure=internal`.
- Label the future public Gateway with `gateway.sbrtech.xyz/exposure=public`.
- Configure the Technitium internal `sbrtech.xyz` Forwarder zone to accept authenticated AXFR and RFC2136 updates from the internal controller.
- Route internal `sbrtech.xyz` resolver traffic to Technitium.
- Create `Secret/external-dns-rfc2136` and `Secret/external-dns-cloudflare` through the corresponding ExternalSecrets.
- Give the Cloudflare API token permission to manage DNS records in `sbrtech.xyz`.

The root Kustomization is reconciled by the existing `infrastructure-external-dns` Flux Kustomization. The base directory is not reconciled directly.
