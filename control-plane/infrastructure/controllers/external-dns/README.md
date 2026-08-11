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
- Give the Cloudflare API token `Zone:Read` and `DNS:Edit` permissions scoped to `sbrtech.xyz`. Credential scoping is the security boundary; the deployment intentionally does not commit the Cloudflare zone ID.

The root Kustomization is reconciled by the existing `infrastructure-external-dns` Flux Kustomization. The base directory is not reconciled directly.

## Cloudflare request-budget controls

Cloudflare applies a global client API limit across account activity, so the public overlay deliberately reduces periodic and event-driven API traffic:

| Control | Value | Purpose |
|---|---|---|
| Replicas | `1` | Prevent duplicate controllers from polling the same zone |
| Periodic interval | `5m` | Reduce steady-state reconciliation frequency |
| Event reconciliation | Enabled | Publish intended Gateway/HTTPRoute changes promptly |
| Minimum event interval | `1m` | Coalesce bursts of Gateway API status and route events |
| Provider record cache | `15m` | Reuse Cloudflare record inventories between reconciliations |
| Records per page | `5000` | Normally list the entire zone in one request |
| Batch size | `200` | Group record creates, updates, and deletes into batch requests |
| Batch interval | `1s` | Space multiple batch requests if a change exceeds one batch |

ExternalDNS invalidates its provider cache after applying DNS changes. Kubernetes-driven changes therefore reach Cloudflare immediately after the event debounce. The main trade-off is that manual Cloudflare-side drift may take up to approximately 15 minutes to be observed and repaired.

The UDM Pro DDNS process owns only `lab-edge.sbrtech.xyz`; public ExternalDNS owns the application CNAMEs. This prevents WAN address churn from triggering an ExternalDNS reconciliation.

## Monitoring

Monitor the public controller separately from the internal RFC2136 instance.

### Logs

Alert on repeated occurrences of:

- HTTP `429` responses;
- `rate limit`;
- `exceeded available rate limit retries`;
- persistent Cloudflare list or apply failures.

For an immediate inspection:

```bash
kubectl logs -n external-dns deployment/external-dns-public --since=30m \
  | grep -Ei '429|rate limit|exceeded available rate limit retries|error'
```

A single transient retry is not necessarily an incident. Repeated messages across reconciliation intervals indicate that the controller or other Cloudflare account activity is consuming the shared API budget.

### Provider-cache metrics

ExternalDNS exposes these provider-cache counters:

```text
external_dns_provider_cache_records_calls
external_dns_provider_cache_apply_changes_calls
```

`external_dns_provider_cache_records_calls` has a `from_cache` label. In normal steady state, cache hits should outnumber provider reads:

```promql
sum(rate(external_dns_provider_cache_records_calls{from_cache="true"}[15m]))
```

compared with:

```promql
sum(rate(external_dns_provider_cache_records_calls{from_cache="false"}[15m]))
```

Investigate when provider reads remain continuously high without corresponding HTTPRoute or Gateway changes. `external_dns_provider_cache_apply_changes_calls` should normally increase only when a public application record is created, changed, or removed.

### Post-rollout checks

1. Confirm `HelmRelease/external-dns-public` and its Deployment are Ready.
2. Inspect the rendered container arguments for the provider cache, page size, event debounce, and batch settings.
3. Confirm the Cloudflare zone contains no record while no HTTPRoute references the public Gateway.
4. Add a disposable public-parent route and verify one application CNAME is created for `lab-edge.sbrtech.xyz`.
5. Confirm repeated no-change reconciliations produce cache hits rather than repeated provider reads.
6. Remove the disposable route and verify `policy: sync` removes only the ExternalDNS-owned application record.
