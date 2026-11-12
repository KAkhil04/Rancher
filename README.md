# RKE2 Production Migration Plan: Ingress NGINX → Traefik

> **Scope:** Migration of the bundled `rke2-ingress-nginx` controller to `rke2-traefik` on a live production RKE2 cluster, following the official SUSE RKE2 migration path (4-phase, zero/low-downtime).
>
> **Environment profile (this runbook is tailored to):**
> - RKE2 ≥ June releases (v1.34.9+ / v1.35.6+ / v1.36.2+) → provider key `kubernetesIngressNGINX`
> - **Airgapped** — image tarballs / private registry
> - **Rancher-provisioned or imported** cluster
> - Traffic entry topology: **TBD by discovery** (hostPort / external LB / DNS)
>
> **Sources:**
> - [RKE2: Ingress NGINX to Traefik Migration Guide](https://docs.rke2.io/reference/ingress_migration)
> - [Traefik: Kubernetes Ingress NGINX routing & annotation support](https://doc.traefik.io/traefik/reference/routing-configuration/kubernetes/ingress-nginx/)
> - [Traefik: NGINX→Traefik migration guide](https://doc.traefik.io/traefik/migrate/nginx-to-traefik/)
> - [traefik/ingress-nginx-migration discovery tool](https://github.com/traefik/ingress-nginx-migration)
> - [SUSE Prime KB](https://support.scc.suse.com/s/kb/Migrating-from-Ingress-NGINX-to-Traefik-in-a-standalone-RKE2-cluster) *(login required — fetch if you have a SUSE account)*
> - [RKE2 networking services / ingress controller docs](https://docs.rke2.io/networking/networking_services)

---

## 0. Context & Hard Facts

### Why migrate
- `ingress-nginx` upstream went **EOL March 2026** — no patches, no fixes.
- **RKE2 v1.36**: Traefik is the default for new clusters; existing clusters keep nginx until switched.
- **RKE2 v1.37**: standalone `ingress-nginx` removed. Only valid `ingress-controller` values: `ingress-nginx + traefik` (dual) or `traefik`. **Phase 1 steps 1–2 minimum must be complete before upgrading to v1.37 or the cluster fails to start.**
- SUSE Rancher Prime LTS customers: nginx support through Dec 2027.

### Version gates that control this plan

| Condition | Consequence |
|---|---|
| v1.32 ≥ .11, v1.33 ≥ .7, v1.34 ≥ .3, any v1.35+ | Migration path supported at all |
| ≥ v1.34.9 / v1.35.6 / v1.36.2 / v1.33.13 | Provider key **`kubernetesIngressNGINX`** (capital NGINX) — older builds use `kubernetesIngressNginx`. Wrong key = provider silently never starts. |
| Traefik ≥ **v3.6.2** | Required for the `kubernetesIngressNGINX` provider to exist at all |
| v1.36+ | Traefik images bundled in `rke2-images-core` tarball; separate traefik tarball removed |
| ≤ v1.35.x (airgap) | Must stage `rke2-images-traefik.linux-<arch>.tar.zst` |

### The bridge-class design (why the plan works)
RKE2's approach uses a transitional IngressClass `rke2-ingress-nginx-migration` instead of pointing Traefik at the `nginx` class, because:
1. Two controllers owning the same Ingress race over `status.loadBalancer.ingress[]` (ExternalDNS/GitOps churn).
2. The `nginx` IngressClass is **deleted automatically** when nginx is uninstalled — Traefik-served copies would lose their class mid-migration.

---

## 1. Assumptions

Confirm each before starting; a "no" changes the plan:

- [ ] RKE2 version on a supported line (see version gates)
- [ ] Ingress-nginx is the RKE2 bundled chart (`rke2-ingress-nginx` in `kube-system`), not a custom helm install
- [ ] This cluster does **not** host the Rancher server itself (no `rancher` Ingress in `cattle-system` — else SUSE support path)
- [ ] Cluster-admin RBAC available (debug pods for discovery, HelmChartConfig, restarts)
- [ ] Maintenance window exists for Phase 3 + owner for go/no-go and rollback calls
- [ ] `kubectl`, `jq`, and `ingress-nginx-migration` binary available on the operator machine

---

## 2. Phase 0 — Discovery & Inventory

**Script:** `rke2-ingress-discovery.sh` (in this directory). Run:

```bash
./rke2-ingress-discovery.sh [cluster-name]   # arg optional if exactly one cluster in fleet-default
```

Produces `discovery-report-<ts>.txt` + backup artifacts (`ingress-backup`, `nginx-cm-backup`, `nginx-hcc-backup`, `cluster-spec-backup`, `report-<ts>.json`).

### What it collects (and the decision each feeds)

| Section | Collects | Feeds |
|---|---|---|
| A | k8s/rke2 version, node arch/OS/count, helmchart versions | Provider key name, airgap tarball, v1.37 constraint |
| A2 | Per-node via `kubectl debug node`: `rke2 --version`, config.yaml keys, registries.yaml, node-port-range, `agent/images` contents, crictl traefik images, `ss` listeners 80/443/8000/8443, SELinux, firewalld | Airgap staging scope, port collisions, CIS/hardening |
| B | Rancher local-cluster check, fleet cluster, `machineGlobalConfig`, GitOps-owned ingresses | Provisioned vs imported path, source-vs-patch changes |
| C | nginx ConfigMap data, tcp/udp-services, HelmChartConfig, DS args/ports/tolerations/resources, webhooks, IngressClasses | Provider-option translation, L4 gap, sizing, dual defaults |
| C2 | Any existing `rke2-traefik` HelmChartConfig | Pre-existing config conflicts |
| D | Class map (classless?), counts, **unique annotation names**, hosts, TLS, cert-manager shim users | Pinning scope, audit surface, duplicate strip-list |
| D2 | `ingress-nginx-migration` JSON report | Authoritative unsupported-annotation verdict |
| D3 | pathType distribution, hostless/wildcard ingresses, names >53 chars, `*-traefik` collisions | strictValidatePathType, open-redirect risk, naming |
| D4 | Snippet annotation **bodies** | Directive-by-directive audit vs supported subset |
| D5 | Key annotation **values** (body-size, timeouts, auth-url, CIDRs, limits, backend-protocol…) | Provider option sizing + per-ingress test cases |
| D6 | Ingress backends with zero endpoints | Pre-existing breakage (fix before, not during) |
| E/E2 | certs, issuers, ACME solver types, **solver ingress class** | HTTP-01 orphan risk after nginx removal |
| F | svc types, LoadBalancers, ExternalDNS, occupied hostPorts/NodePorts | Cutover method, temp-port availability |
| F2 | NetworkPolicies, service mesh | Silent traffic-drop risk |
| F3 | `kubectl top nodes` | DaemonSet capacity headroom |
| G | ServiceMonitors / PrometheusRules on nginx metrics | Observability re-implementation |
| H | Config backups, cluster-spec backup | Rollback artifacts |

### Manual items the script cannot reach
- External LB: pool membership, health-check path (`/healthz`→`/ping`), weighted/shadow pools, PROXY protocol
- DNS: records pointing at ingress IPs, TTLs, ability to lower TTL
- Maintenance window, rollback owner, comms plan
- Airgap: download + checksum-verify + stage tarballs

### GO/NO-GO (from discovery)
- [ ] rke2 build supported; correct provider key selected
- [ ] Traefik ≥ v3.6.2 in the release's image set
- [ ] `tcp-services`/`udp-services` ConfigMaps **empty** (else separate L4 workstream)
- [ ] `unsupportedIngressCount == 0`, or every offender has a documented decision
- [ ] All Ingresses classed `nginx` (or pinning list ready)
- [ ] Snippet bodies use only supported directives
- [ ] 8000/8443 free (hostPorts, NodePorts, `ss`, firewall)
- [ ] No Rancher local ingress; no blocking NetPol; backends healthy
- [ ] cert-manager solver-class flip planned
- [ ] Backups verified; airgap images staged on **every** node (server + agent)

---

## 3. Annotation Compatibility Reference

The `kubernetesIngressNGINX` provider translates ~85+ common annotations. **It is not complete.** Audit output = unique annotation names (D) + values (D5) + snippet bodies (D4) + tool report (D2).

### 3.1 Supported — but with behavioral differences (test cases required)

| Annotation | Difference |
|---|---|
| `auth-url` | ForwardAuth, not sub-request. With `rewrite-target`/`rewrite` on same Ingress, `X-Forwarded-Uri` = **rewritten** path. Limited NGINX-var interpolation only. |
| `auth-signin` | Auto-appends `rd=`; routes **without** a `Host` matcher → open-redirect risk. Scope Host rules. |
| `auth-snippet` / `server-snippet` / `configuration-snippet` | Only directive subset: `add_header`, `proxy_set_header`, `proxy_method`, `more_set_headers`, `more_set_input_headers`, `set`, `if`, `return`; `location` (server-snippet) and `rewrite` (configuration-snippet) also supported. Anything else = manual rewrite. |
| `auth-tls-verify-client`, `auth-tls-secret` | Rejection at TLS handshake, not HTTP 400. `auth-tls-verify-depth`: no depth cap (Go accepts any valid chain). |
| `affinity-mode` | `persistent` only (no `balanced`). `affinity-canary-behavior`: `sticky` only. |
| `load-balance` | **Silently ignored** — round_robin only; no EWMA/IP-hash. |
| `backend-protocol` | FCGI/AUTO_HTTP unsupported. `proxy-http-version`: only `"1.1"` (`"1.0"` warns). |
| `custom-headers` | Inert unless provider `globalAllowedResponseHeaders` is set. |
| `default-backend` | Same-namespace only unless `allowCrossNamespaceResources`. |
| `ssl-redirect` / `force-ssl-redirect` | If enabled **globally**, cannot opt out per route. |
| `ssl-passthrough` | SNI/default-backend handling differs — verify presented cert. |
| `from-to-www-redirect` | No wildcard hosts. `server-alias` ignored if conflicting with an existing Host rule. |
| `permanent-redirect` | defaults 301; `temporal-redirect` defaults 302 & takes precedence; `*-code` = valid 3xx only. |
| `limit-rps/rpm/connections/burst-multiplier` | Leaky-bucket+queue → token-bucket; over-limit returns **429 not 503**; `limit-connections` per client IP, ≤0 ignored; burst-multiplier <1 → 5. |
| `proxy-next-upstream*` | Retry may hit the **same** server; `error`≡`timeout`; tries=0 → capped at server count. |
| `proxy-request-buffering` | NGINX default ON, Traefik default **OFF** → set `proxyRequestBuffering: true` to preserve behavior. |
| `proxy-buffers-number` | Computes buffer size as `size × number` (different semantics). |
| `enable-access-log` | Only effective if access logs enabled at install level; then per-Ingress true/false toggles. |
| `enable-cors` | Partial; pre-flight vs regular header handling differs; `cors-allow-origin` wildcard = `protocol://*.domain` single-label only. |
| `whitelist/allowlist-source-range` | Client-IP source depends on `ipAllowListStrategy` — configure if behind a proxy using XFF. |

### 3.2 Unsupported annotations — hard blockers

```
auth-tls-error-page · auth-tls-match-cn · auth-cache-key · auth-cache-duration
auth-keepalive · auth-keepalive-share-vars · auth-keepalive-requests · auth-keepalive-timeout
auth-proxy-set-headers · auth-always-set-cookie · auth-request-redirect · auth-signin-redirect-param
disable-proxy-intercept-errors · limit-rate · limit-rate-after · limit-whitelist
global-rate-limit · global-rate-limit-window · global-rate-limit-key · global-rate-limit-ignored-cidrs
preserve-trailing-slash (Traefik always preserves) · proxy-cookie-domain · proxy-cookie-path
proxy-busy-buffers-size · proxy-redirect-from · proxy-redirect-to
proxy-ssl-ciphers · proxy-ssl-verify-depth · proxy-ssl-protocols
enable-rewrite-log · satisfy · session-cookie-conditional-samesite-none
session-cookie-change-on-failure · upstream-hash-by-subset · upstream-hash-by-subset-size
ssl-ciphers · ssl-prefer-server-ciphers · http2-push-preload · connection-proxy-header
enable-opentracing · opentracing-trust-incoming-span · enable-opentelemetry · opentelemetry-trust-incoming-span
enable-modsecurity · enable-owasp-core-rules · modsecurity-transaction-id · modsecurity-snippet
mirror-request-body · mirror-target · mirror-host · denylist-source-range · stream-snippet
```

Per finding: rewrite as Traefik Middleware/CRD, replace natively (`connection-proxy-header`→`ipAllowListStrategy`/trustedIPs; `enable-opentelemetry`→static tracing config), or accept loss with sign-off. ModSecurity has no OSS equivalent (Traefik Hub WAF only).

### 3.3 Global-config-only gaps (not fixable per-Ingress)

- **Trailing slashes**: Traefik preserves; NGINX strips unless configured → backend behavior may change.
- No leader election; no EWMA/IP-hash; no auth session caching (more load on auth endpoints).
- CORS: NGINX sends all headers unconditionally; Traefik differs pre-flight vs regular.

---

## 4. ConfigMap → Provider Options Mapping

Export current values (discovery §C), then translate. **Units differ:** `16k`/`1m` → bytes; `30s` → integer seconds; `on/off` → bool.

| nginx ConfigMap key | `providers.kubernetesIngressNGINX.*` option | Notes |
|---|---|---|
| `proxy-connect-timeout` | `proxyConnectTimeout` | int seconds |
| `proxy-send-timeout` | `proxySendTimeout` | int seconds |
| `proxy-read-timeout` | `proxyReadTimeout` | int seconds |
| `proxy-request-buffering` | `proxyRequestBuffering` | bool; nginx default on, Traefik off |
| `client-body-buffer-size` | `clientBodyBufferSize` | `16k`→16384 |
| `proxy-buffering` | `proxyBuffering` | bool |
| `proxy-body-size` | `proxyBodySize` | `50m`→52428800 |
| `proxy-buffer-size` | `proxyBufferSize` | bytes |
| `proxy-buffers-number` | `proxyBuffersNumber` | integer (size×number semantics!) |
| `proxy-next-upstream` | `proxyNextUpstream` | space-separated conditions, e.g. `"error timeout http_502"` |
| `proxy-next-upstream-timeout` | `proxyNextUpstreamTimeout` | int seconds |
| `proxy-next-upstream-tries` | `proxyNextUpstreamTries` | integer |
| `custom-http-errors` | `customHTTPErrors` + `defaultBackendService` | error-page backend service |
| `global-allowed-response-headers` | `globalAllowedResponseHeaders` | **required for `custom-headers` to work** |
| `allow-cross-namespace-resources` | `allowCrossNamespaceResources` | cross-ns secret/backend refs |
| `strict-validate-path-type` | `strictValidatePathType` | Traefik v3.7 default true |
| `ssl-redirect`/`force-ssl-redirect` (global) | entryPoint redirect + `httpEntryPoint`/`httpsEntryPoint` | no per-route opt-out |
| `ssl-protocols`/`ssl-ciphers` | TLS options (entryPoint or `traefik.ingress.kubernetes.io/router.tls.options`) | |
| `hsts`, `hsts-max-age`, `hsts-include-subdomains`, `hsts-preload` | Headers middleware (`stsSeconds`, `stsIncludeSubdomains`, `stsPreload`, `forceSTSHeader`) on entryPoint | |
| `use-proxy-protocol` | entryPoint `proxyProtocol` | also check `ipAllowListStrategy` |
| `access-log-path` | `accessLog.filePath` | |
| `log-format-upstream` | `accessLog.format` (`common`/`genericCLF`/`json`) | no custom NGINX log templates |

**No equivalent** (translate intent or accept gap): `worker-processes`, `worker-cpu-affinity`, lua shared-dict, `main-snippet`, `http-snippet`, `server-snippet`, `location-snippet`, `stream-snippet` (CM-level), custom log formats.

---

## 5. Phase 1 — Dual Ingress Setup

**Pre-step:** stage airgap images on **every** node (server + agent):

```
# v1.34.x / v1.35.x:
rke2-images-traefik.linux-<arch>.tar.zst  →  /var/lib/rancher/rke2/agent/images/
# v1.36.x: already inside rke2-images-core — verify crictl images | grep traefik
```

Or push Traefik images to `system-default-registry`/private mirror.

### Step 1 — Pin all existing Ingresses to `nginx`

Prevents both controllers racing to claim classless Ingresses when Traefik appears.

```bash
kubectl get ingress --all-namespaces -o custom-columns='NAMESPACE:.metadata.namespace,NAME:.metadata.name' --no-headers | while read NS NAME; do
    kubectl patch ingress "$NAME" -n "$NS" --type=merge -p '{"spec": {"ingressClassName": "nginx"}}'
done
```

- **GitOps/Fleet/Helm-owned Ingresses:** change `ingressClassName` at source instead — patch gets reverted.
- Verify every row shows `nginx`:

```bash
kubectl get ingress --all-namespaces -o custom-columns='NAMESPACE:.metadata.namespace,NAME:.metadata.name,ICLASS:.spec.ingressClassName'
```

### Step 2 — Enable both controllers

- **Rancher-provisioned:** edit cluster YAML → `spec.rkeConfig.machineGlobalConfig`:

```yaml
ingress-controller:
  - ingress-nginx
  - traefik
```

- **Imported/standalone:** same in `/etc/rancher/rke2/config.yaml` on **all** server nodes.

### Step 3 — Traefik HelmChartConfig (temp ports + provider)

```yaml
# rke2-traefik-config.yaml
apiVersion: helm.cattle.io/v1
kind: HelmChartConfig
metadata:
  name: rke2-traefik
  namespace: kube-system
spec:
  valuesContent: |-
    ports:
      web:
        hostPort: 8000
      websecure:
        hostPort: 8443
    providers:
      kubernetesIngressNGINX:
        enabled: true
        ingressClass: "rke2-ingress-nginx-migration"
        controllerClass: "rke2.cattle.io/ingress-nginx-migration"
        # ↓ translated ConfigMap equivalents go here (section 4)
        # proxyRequestBuffering: true
        # proxyBodySize: 52428800
        # proxyReadTimeout: 120
        # globalAllowedResponseHeaders: ...
    # metrics:
    #   prometheus:
    #     enabled: true
```

- **Rancher-provisioned:** deliver via `spec.rkeConfig.additionalManifest` (survives CP-node reprovisioning) — or place the file in `/var/lib/rancher/rke2/server/manifests/` on **every** server node.
- Also fold in: resources sized to nginx footprint, tolerations/nodeSelector mirroring the nginx DS, metrics/ServiceMonitor for parallel observability.

### Step 4 — Rollout

- Provisioned: save cluster YAML → Rancher reconciles; watch node plans.
- Imported: `sudo systemctl restart rke2-server` on each CP node **sequentially** — verify node Ready + etcd health between each.

### Step 5 — Verify

```bash
kubectl get daemonset -n kube-system                      # both rke2-ingress-nginx-controller AND rke2-traefik
kubectl get ingressclass                                  # nginx, traefik, rke2-ingress-nginx-migration
kubectl -n kube-system logs ds/rke2-traefik | grep "Starting provider \*ingressnginx.Provider"
kubectl -n kube-system get ds rke2-traefik -o jsonpath='{.spec.template.spec.containers[*].ports}'   # 8000/8443
```

- All existing routes still serve on 80/443 via nginx — run smoke suite **now**.
- Check **dual defaults**: if `traefik` IngressClass is also marked default, remove `ingressclass.kubernetes.io/is-default-class` from it or confirm 100% explicit classing.
- **Rollback (P1):** set `ingress-controller` back to `ingress-nginx` only; delete the HelmChartConfig + `rke2-traefik` HelmChart resources; restart.

---

## 6. Phase 2 — Parallel Validation

Goal: prove Traefik serves identical responses on temp ports before any production traffic moves.

### Step 1 — Duplicate & reclassify

For every nginx-class Ingress, create `<name>-traefik` with `ingressClassName: rke2-ingress-nginx-migration`. Use the official jq duplication script **with these extra field strips**:

```jq
del(
  .metadata.resourceVersion, .metadata.uid, .metadata.creationTimestamp,
  .metadata.annotations["kubectl.kubernetes.io/last-applied-configuration"],
  .metadata.annotations["field.cattle.io/publicEndpoints"],
  .metadata.annotations["cert-manager.io/cluster-issuer"],
  .metadata.annotations["cert-manager.io/issuer"],
  .metadata.annotations["acme.cert-manager.io/http01-edit-in-place"],
  .status, .metadata.managedFields
)
```

**Before running:**
- **cert-manager ingress-shim:** stripping issuer annotations is mandatory — otherwise the duplicate mints a second Certificate fighting for the same `spec.tls.secretName`.
- GitOps-owned Ingresses: commit `-traefik` copies to source instead of hand-applying.
- Names >53 chars: suffix exceeds 63-char limit → use a shorter deterministic suffix and track the mapping.
- Start with a **canary subset** (plain HTTP, TLS, rewrite-target, auth, large body, websocket) before bulk duplication.

### Step 2 — Test both paths

```bash
curl -sI  http://<NodeIP>/path      -H "Host: app.example.com"    # nginx  (80/443)
curl -sI  http://<NodeIP>:8000/path -H "Host: app.example.com"    # traefik
curl -sI https://<NodeIP>:8443/path --resolve app.example.com:8443:<NodeIP>
```

If firewalls can't open 8000/8443: `kubectl -n kube-system port-forward svc/rke2-traefik 8000:80 8443:443`.

**Per-Ingress test matrix:** status codes · headers (custom/proxy headers) · TLS cert presented · redirects (301 vs 302 semantics) · large POST/upload · websockets · gRPC backends · external auth (`X-Forwarded-Uri` + `rd=` behavior) · session-affinity stickiness · regex/`ImplementationSpecific` paths · rate-limit behavior (expect 429 not 503) · trailing-slash sensitivity.

Watch Traefik logs for provider warnings/dropped annotations; diff nginx vs Traefik access logs for the same request set.

### Step 3 — (optional) external LB parallel backend

Add Traefik (`<NodeIP>:8000`, health check **`/ping`**) as a shadow/weighted backend for live-traffic validation. Weighted progression per the Traefik guide: 100/0 → 90/10 → 50/50 → 10/90 → 0/100.

**Rollback (P2):** delete `-traefik` Ingresses. Zero production impact.

> **Do not proceed until every prod-critical Ingress passes.** This phase can run for days — it's the risk-reduction phase.

---

## 7. Phase 3 — Final Switchover

Take an etcd snapshot first: `rke2 etcd-snapshot save --name pre-phase3`. Split the phase for minimal downtime.

### 3a — Remove nginx (Traefik still on 8000/8443)

```yaml
ingress-controller:
  - traefik
```

Roll CP nodes. nginx chart teardown is **slow** (webhook/CR cleanup). Traffic handling during the window, per discovered topology:

| Topology | Action |
|---|---|
| External LB → nodes | **First** repoint LB to `<NodeIP>:8000/8443` + health check `/ping`, then restart — near-zero downtime |
| hostPort + DNS-to-nodes | Maintenance window; nothing answers 80/443 between nginx removal and Traefik port flip — minimize by doing 3a+3b back-to-back |
| ServiceLB/MetalLB VIP | Cutover via service/selector or VIP reassignment |

Verify gone: `kubectl -n kube-system get ds rke2-ingress-nginx-controller` → empty; `kubectl get validatingwebhookconfigurations | grep nginx` → empty; `kubectl get ingressclass nginx` → gone (expected — bridge class carries the copies).

**cert-manager (from discovery §E2):** if solver class is `nginx`, set `--acme-http01-solver-class=traefik` (or solver `ingress.class: traefik`) **now**, before HTTP-01 renewals hit orphans.

### 3b — Traefik to standard ports

Remove port overrides from the HelmChartConfig (keep the provider block), roll CP nodes again:

```yaml
spec:
  valuesContent: |-
    providers:
      kubernetesIngressNGINX:
        enabled: true
        ingressClass: "rke2-ingress-nginx-migration"
        controllerClass: "rke2.cattle.io/ingress-nginx-migration"
        # + translated options
```

Verify DaemonSet shows hostPort 80/443; `-traefik` Ingresses now serve production on standard ports.

### 3c — Post-switch verification

- Full smoke suite on 80/443 for **every** migrated route
- LB health checks green on `/ping`; `/healthz` checks removed
- `traefik_*` metrics flowing; dashboards/alerts live
- No nginx remnants: `kubectl -n kube-system get all,cm,secret | grep -i nginx`; webhooks list
- **DNS-based cutovers:** keep nginx rollback-ready **24–48h** — some ISPs ignore TTL.

**Rollback (P3, pre-v1.37 only):** re-add `ingress-nginx` to `ingress-controller` + restore old HelmChartConfig + restart — original `nginx`-class Ingresses still exist and resume serving. After v1.37, etcd snapshot is the fallback.

---

## 8. Phase 4 — Cleanup

```bash
# Delete legacy nginx-class Ingresses (fixes the doc's awk bug that exits after first match)
kubectl get ingress -A -o custom-columns='NS:.metadata.namespace,NAME:.metadata.name,ICLASS:.spec.ingressClassName' --no-headers \
  | awk '$3=="nginx" {print $1, $2}' | while read NS NAME; do
    kubectl delete ingress "$NAME" -n "$NS"
done
```

Then:
- Delete at source for GitOps-owned objects.
- Optionally rename `-traefik` copies back to canonical names (cosmetic, later).
- Confirm final HelmChartConfig contents; remove stale temp-port manifest files.
- Close firewall for 8000/8443 if opened.
- Keep `kubernetesIngressNGINX` provider enabled (harmless — it only reads the bridge class); disable later in a separate window once no stragglers.
- Update runbooks, dashboards, LB configs, documentation.
- **Never** set `ingress-controller: ingress-nginx` again — invalid on v1.37+.

---

## 9. Scenario / Risk Matrix

| Scenario | Handling |
|---|---|
| Unsupported annotation found | Pause that Ingress → Traefik middleware/CRD rewrite, native equivalent, or signed-off loss |
| Snippet directives outside supported subset | Same as above — no arbitrary NGINX config possible |
| `tcp/udp-services` in use | **Separate workstream**: `IngressRouteTCP`/`IngressRouteUDP` + extra `ports:` on chart, or independent L4 LB. Migration class won't carry them — silent break |
| ModSecurity/OWASP in use | No OSS equivalent; Traefik Hub WAF, external WAF, or accept gap |
| cert-manager ingress-shim | Strip issuer annotations on duplicates (done in §6.1); flip solver class to `traefik` (§7.1) |
| HTTP-01 ACME during coexistence | Traefik on temp ports isn't public → challenges fail until switchover; `tls.secretName` unaffected; DNS-01 unaffected |
| GitOps/Fleet owns Ingresses | All class changes at source; commit `-traefik` copies; expect drift warnings |
| Two default IngressClasses | Explicit classing (P1S1) is the fix; optionally strip `is-default-class` from `traefik` |
| 8000/8443 occupied | Choose alternates, substitute everywhere, sync firewall/NSG |
| nginx uninstall hangs | Known (webhook teardown) — wait, don't force-delete; this is why 3a/3b are split |
| ExternalDNS present | It reacts to `status.loadBalancer` writer changes — the bridge class prevents dual writers; verify its source filters don't emit records for `-traefik` copies |
| Rancher local cluster (rancher Ingress exists) | SUSE support path — do not proceed with this runbook |
| Traefik image missing on some node | Pod ImagePullBackOff → partial node-level outage; pre-verify `crictl images` everywhere |
| Rancher reprovisions a CP node | Manually-placed manifests are lost → use `additionalManifest` (§5.3) |
| Mid-migration v1.37 upgrade | Need Phase 1 steps 1–2 done minimum, or cluster won't start |
| Wrong provider key name for build | Provider never starts — check for `Starting provider *ingressnginx.Provider` in logs (§5.5) |
| NetPol selects nginx pods | Traefik has different labels → traffic silently dropped (discovery §F2) |
| auth-url + rewrite-target combo | `X-Forwarded-Uri` shows rewritten path — auth service may break (§3.1) |
| Trailing-slash-sensitive backends | Traefik preserves; NGINX strips — explicit test case |
| Ingress names >53 chars | `-traefik` suffix overflows 63-char limit — use short suffix + mapping table |

---

## 10. Post-Migration Considerations

- Migrate critical Ingresses gradually to native `traefik` class / `IngressRoute` or Gateway API — the bridge class is a compatibility shim, not a destination.
- Re-enable/adjust `publishService` behavior if anything consumed nginx-written ingress status.
- Traefik-native: access logs format, metrics dashboards, tracing config (replaces `enable-opentelemetry` gap).
- Provider options now live in the HelmChartConfig — version-control it (it's part of the contract with the cluster).
- SUSE Prime LTS remains an option if any straggler workload can't migrate by the v1.37 window.

---

## 11. Rollback Summary

| Phase | Rollback |
|---|---|
| 0 | n/a — read-only |
| 1 | `ingress-controller: [ingress-nginx]`; delete traefik HelmChartConfig + chart; restart |
| 2 | Delete `-traefik` Ingresses |
| 3a/3b | Re-add `ingress-nginx` + restore old HelmChartConfig + restart (**only pre-v1.37**); else etcd restore |
| 4 | Re-apply from `ingress-backup-<ts>.yaml` (requires nginx re-enabled first) |

## 12. Change-Record Checklist (sign-offs)

- [ ] Discovery report reviewed; all GO/NO-GO boxes checked
- [ ] Annotation decisions documented per offender
- [ ] ConfigMap translation reviewed by app owners
- [ ] etcd snapshot confirmed before Phase 1 and Phase 3
- [ ] Phase 2 test matrix executed and signed
- [ ] LB/DNS changes logged with before/after
- [ ] Rollback owner identified per phase
- [ ] Post-migration verification sign-off
