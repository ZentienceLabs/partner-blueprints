# AlchemiStudio

Partner blueprints for **AlchemiStudio** — an AI platform (agentic Copilot, AIOS
computer-use sandbox, and RAG) for SUSE AI Factory.

| Blueprint | What it deploys |
|---|---|
| [`alchemistudio-with-vllm`](blueprints/alchemistudio-with-vllm-0.1.0.yaml) | AlchemiStudio + in-cluster **vLLM** (Qwen2.5-3B-Instruct, open stack) as the Copilot model, plus a bundled CPU embedding server for RAG — fully self-hosted (no NGC / HuggingFace / cloud keys). |

## Prerequisites

- **SUSE AI Factory** (aif-operator) on a Rancher-managed RKE2 cluster.
- **An HTTP ClusterRepo for the Rancher partner charts** (`http-partner-charts`) — the
  `alchemistudio-platform` chart is pulled from it. Use the HTTP index, **not** Rancher's default
  git-backed `rancher-partner-charts`: the chart unpacks to more than Fleet's 1 MiB bundle
  limit, which only git-backed repos hit.

  ```bash
  kubectl apply -f - <<'EOF'
  apiVersion: catalog.cattle.io/v1
  kind: ClusterRepo
  metadata:
    name: http-partner-charts
  spec:
    url: https://partner-charts.rancher.io
  EOF
  ```

- **One free GPU** with ≥16 GB memory (e.g. NVIDIA T4) on a node with runtimeclass `nvidia`,
  tainted the usual way (`nvidia.com/gpu=present:NoSchedule`) so only GPU workloads land on it —
  vLLM tolerates that taint; the platform pods don't.
  vLLM claims a whole GPU (see step 1 below to check).
- **Kata is optional.** The blueprint sets `kata.enabled: false`, so the code sandbox and AIOS
  computer run as standard containers on any Linux node — a stock RKE2 cluster works. For
  production / multi-user installs, prepare a Kata node (KVM-capable, Kata runtime installed,
  labelled `kata=true` and tainted `pool=kata:NoSchedule`) and set `kata.enabled: true` to run
  user code in Kata micro-VMs.
- **A way to reach the gateway:** a LoadBalancer provider (cloud provider, MetalLB, …) for the
  `apisix-gateway` Service — or, without one, the cluster's ingress controller in front of it
  (see [Reaching the app without a LoadBalancer](#reaching-the-app-without-a-loadbalancer)).
- **Storage — Longhorn by default, one storage provider for everything.** The StorageClass
  names are blueprint component values; without Longhorn, set all three to your own classes,
  or the pods stay `Pending` on unbound PVCs.

  | Used for | Access mode | Blueprint value | Default |
  |---|---|---|---|
  | Postgres, Redis, SeaweedFS, etcd, OpenBao | ReadWriteOnce | `global.storageClass` | `longhorn` |
  | Shared AIOS / file data volume (mounted by several pods) | **ReadWriteMany** | `aios.dataPlane.storageClass` | `longhorn` |
  | vLLM model cache | ReadWriteOnce | vllm `storageClass` | `longhorn` |

  Longhorn RWX needs an NFS client on the nodes; any RWX-capable class (e.g. NFS) works. If
  the GPU node isn't a Longhorn node, use a node-local class such as `local-path` for vLLM.
- **cert-manager** for `tls.mode=acme` (Let's Encrypt), or your own cert with `tls.mode=byo`.
  Let's Encrypt's HTTP-01 challenge is answered by the cluster's ingress controller, set by
  `tls.acme.ingressClass`: `traefik` by default (RKE2's default), or `nginx` for
  rke2-ingress-nginx.
- **Egress** to `dp.apps.rancher.io`, `ghcr.io`, `huggingface.co` and `alchemiacr.azurecr.io`
  (mirror these for an air-gapped install).

## Install — step by step

1. **Check for a free GPU.**

   ```bash
   for node in $(kubectl get nodes -o jsonpath='{.items[*].metadata.name}'); do
     allocatable=$(kubectl get node "$node" -o jsonpath='{.status.allocatable.nvidia\.com/gpu}')
     if [ -n "$allocatable" ] && [ "$allocatable" -gt 0 ]; then
       allocated=$(kubectl get pods -A --field-selector spec.nodeName="$node",status.phase!=Failed,status.phase!=Succeeded \
         -o jsonpath='{range .items[*].spec.containers[*]}{.resources.requests.nvidia\.com/gpu}{"\n"}{end}' \
         | grep -v '^$' | awk '{s+=$1} END {print s+0}')
       echo "Node: $node | Total: $allocatable | Allocated: $allocated | Free: $((allocatable - allocated))"
     fi
   done
   ```

   You need `Free >= 1` on at least one node.

2. **Create the `http-partner-charts` ClusterRepo** and **check the StorageClass names** in
   the blueprint match your cluster (see Prerequisites).

3. **Choose how the app is reached** — in the blueprint's values, or at install time (values
   given in the install wizard are applied to the components and override the blueprint's
   defaults). Set **one** of:
   - `global.domain: ai.example.com` — real domain (production). You'll add DNS records in step 6.
   - `gateway.loadBalancerIP: <reserved static IP>` — no domain; the app is served at
     `<ip>.nip.io` with no DNS setup. Prefer [sslip.io](https://sslip.io)? Also set
     `global.wildcardDns: sslip.io`.

   If both are blank the install stops with a clear message. No LoadBalancer provider? See
   [Reaching the app without a LoadBalancer](#reaching-the-app-without-a-loadbalancer).

4. **Install** from the AI Factory UI: *Blueprints → AlchemiStudio + vLLM → Install →
   Fleet Bundle → target cluster → namespace → Install.* Everything installs into the
   namespace you pick here; the commands below use it as `$NS`:

   ```bash
   NS=<the namespace you picked>
   ```

5. **Wait for the APISIX gateway** to get an external IP — this IP fronts
   every AlchemiStudio hostname:

   ```bash
   kubectl -n "$NS" get svc apisix-gateway -w   # wait until EXTERNAL-IP is set, then Ctrl+C
   ```

6. **Real domain only:** create DNS A-records for `alchemi-app`, `alchemi-auth`, `alchemi-ops`
   and `alchemi-zenith` `.<your-domain>` pointing at that IP. The TLS certificate is issued
   automatically once DNS resolves.

7. **Get the URLs and the first login:**

   ```bash
   kubectl -n "$NS" get configmap alchemi-endpoints -o jsonpath='{.data.APP_URL}{"\n"}'
   kubectl -n "$NS" get configmap alchemi-endpoints -o jsonpath='{.data.ADMIN_EMAIL}{"\n"}'
   kubectl -n "$NS" get secret alchemi-secrets -o jsonpath='{.data.FIRST_USER_PASSWORD}' | base64 -d; echo
   ```

   Keycloak asks for a new password on first login. vLLM needs a few minutes on first install
   to download the model weights.

## What this blueprint deploys

Both components install into **the one namespace you pick in the install wizard** — the
blueprint pins no namespace and the platform chart creates none of its own. So you can run
**several AlchemiStudio instances on one cluster**, one per namespace, each with its own
domain (or static IP). Cluster-wide pieces are named per instance, and the External Secrets
controller is scoped to its own namespace.

| Component | What runs there |
|---|---|
| App tier | web, ai, worker, workflow, zenith, compute-api, embeddings (TEI), AIOS workers |
| Data | PostgreSQL (pgvector), Redis, SeaweedFS object store |
| Login | Keycloak (OIDC) |
| Observability | OpenObserve (logs, metrics, traces) |
| Code sandboxes | code sandboxes and the AIOS computer (in Kata micro-VMs when `kata.enabled: true`) |
| Gateway | APISIX + etcd — `apisix-gateway` is the external LoadBalancer |
| Secrets | OpenBao + External Secrets Operator (scoped to this namespace) |
| Model serving | vLLM engine + router (GPU), from the `vllm` component |

**Optional — one namespace per tier:** set `global.namespaceMode: split` in the platform
component's values. The app tier (and vLLM) stay in the wizard namespace; every other tier
gets its own namespace named `<global.namespacePrefix>-<tier>` — `alchemistudio-data`,
`-auth`, `-observability`, `-gateway`, `-secrets`, `-sandbox` by default — created and removed
by the chart. For a second instance in split mode, set a different `global.namespacePrefix`.

## Reaching the app without a LoadBalancer

By default `apisix-gateway` is a LoadBalancer Service; without a LoadBalancer provider it
stays `<pending>`. Instead, put the cluster's ingress controller in front of APISIX. APISIX
still terminates TLS, so it needs the certificate (`tls.mode: acme` or `byo`) and the
ingress must pass HTTPS through unchanged.

1. In the platform component's values, set `gateway.serviceType: ClusterIP` and, as the
   hostname source, `global.domain: <ingress-ip>.sslip.io` (don't use `loadBalancerIP` +
   `wildcardDns` here — that path assumes APISIX owns the IP).
2. Route the AlchemiStudio hostnames to `apisix-gateway`. Tested example for **Traefik**
   (RKE2's default): TLS passthrough on 443 by SNI, plain HTTP on 80. Replace `$NS` with the
   namespace `apisix-gateway` runs in (the install namespace; `<prefix>-gateway` in split
   mode) and `<DOMAIN_REGEX_ESCAPED>` with your domain, dots escaped (e.g.
   `1\.2\.3\.4\.sslip\.io`):

   ```yaml
   apiVersion: traefik.io/v1alpha1
   kind: IngressRouteTCP
   metadata:
     name: apisix-gateway-tls
     namespace: $NS
     annotations:
       kubernetes.io/ingress.class: traefik   # required on RKE2
   spec:
     entryPoints: [websecure]
     routes:
       - match: HostSNIRegexp(`^.+\.<DOMAIN_REGEX_ESCAPED>$`)
         services:
           - name: apisix-gateway
             port: 443
     tls:
       passthrough: true
   ---
   apiVersion: traefik.io/v1alpha1
   kind: IngressRoute
   metadata:
     name: apisix-gateway-http
     namespace: $NS
     annotations:
       kubernetes.io/ingress.class: traefik
   spec:
     entryPoints: [web]
     routes:
       - match: HostRegexp(`^.+\.<DOMAIN_REGEX_ESCAPED>$`)
         kind: Rule
         services:
           - name: apisix-gateway
             port: 80
   ```

Notes:
- Match only your domain — don't use ``HostSNI(`*`)``, which also captures Rancher's traffic.
- APISIX sees the Traefik pod IPs as the client address (relevant for its rate limiting)
  unless PROXY protocol is enabled between Traefik and APISIX.
- With `tls.mode: acme`, there can be a gap of up to ~2 minutes after the certificate is
  issued until APISIX loads it.
- **Traefik only, without APISIX**, isn't supported yet — see the roadmap note below.

## Notes for reviewers

- **Gateway:** the chart deploys **Apache APISIX** into the install namespace as
  AlchemiStudio's own application router: it maps each AlchemiStudio hostname to its service,
  carries a wildcard route for apps users build inside AlchemiStudio, and handles CORS, rate
  limiting, HTTPS redirects and long timeouts for AI requests. Doing this with plain `Ingress`
  objects would need controller-specific annotations (ingress-nginx vs Traefik). APISIX
  doesn't touch the cluster's ingress controller.
  - **Reaching it from outside:** a LoadBalancer Service by default, or the cluster's ingress
    in front of it — see [Reaching the app without a LoadBalancer](#reaching-the-app-without-a-loadbalancer).
  - **Roadmap:** we plan to make APISIX optional in a future release, so the platform can run
    directly behind the cluster's ingress controller or an existing gateway.
- **Chart source:** the `alchemistudio-platform` component uses the `http-partner-charts`
  ClusterRepo (`https://partner-charts.rancher.io`) — the same certified chart published via
  rancher/partner-charts, served over HTTP because the git-backed repo exceeds Fleet's 1 MiB
  bundle limit for this chart. AI Factory versions that include
  [SUSE/aif#348](https://github.com/SUSE/aif/pull/348) (compressed bundles) can use the default
  git-backed `rancher-partner-charts` repo as well. The `vllm` component uses
  `application-collection` (the current `vllm` chart; moving to the new `vllm-stack` chart is
  planned for a later blueprint version).
- **Images / credentials:** AlchemiStudio is a commercial product; its images are pulled from
  our registry with a **scoped, read-only (pull-only)** trial token, the same model as our
  published Rancher partner chart.
