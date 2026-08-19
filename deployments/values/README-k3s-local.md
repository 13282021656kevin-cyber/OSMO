<!--
SPDX-FileCopyrightText: Copyright (c) 2026 NVIDIA CORPORATION. All rights reserved.
SPDX-License-Identifier: Apache-2.0
-->

# Local OSMO on single-node k3s

Deploys the OSMO control plane, UI, and a working compute pool on one k3s node,
CPU-only. Verified end to end with `cookbook/tutorials/hello_world.yaml` and
`serial_workflow.yaml` (three chained tasks passing data through S3).

The chart used is the `quick-start` umbrella chart from the **6.3.0 tag**, not
the charts on `main`. `main` is version 1.4.0 / appVersion 6.4.0 and unreleased:
it passes `--backend_token_directory` to the service binary, which the newest
published image (6.3.1) does not accept, so every Python service crash-loops.
Published images stop at 6.3.1, so chart 1.3.0 is the version-consistent choice.

```bash
mkdir -p /tmp/osmo630 && git archive 6.3.0 deployments/charts | tar -x -C /tmp/osmo630
git -C /tmp/osmo630 init -q && git -C /tmp/osmo630 apply -p1 \
  <repo>/deployments/values/k3s-local-quickstart-chart.patch
cp -r /tmp/osmo630/deployments/charts ~/osmo-k3s/
```

The `quick-start` umbrella resolves its dependencies from the `.tgz` files under
`charts/`, not from the sibling source directories, so edits to the `service`
subchart only take effect once it is repackaged:

```bash
cd ~/osmo-k3s/charts
helm package service -d /tmp/pkg && cp /tmp/pkg/service-1.3.0.tgz quick-start/charts/
```

## Why the chart itself is patched

`k3s-local-quickstart.yaml` holds everything a values file *can* express. What it
cannot:

- **Node selectors.** The chart pins pods to `node_group: service` / `data`, and
  hardcodes `node_group: compute` in the default compute pod template. One node
  cannot carry three values of one label key. Helm deep-merges maps and the
  templates render selectors with `range`, so neither `nodeSelector: {}` nor
  `node_group: null` deletes an inherited key — null renders as an empty-valued
  key. The entries have to come out of the chart.
- **The token bootstrap URL.** See below.
- **S3 addressing style.** The chart omits `addressing_style`, which defaults to
  virtual-hosted style: the SDK prepends the bucket as a subdomain and produces
  `http://osmo.localstack-s3.osmo:4566`, a hostname that does not resolve
  in-cluster. Every task output upload and downstream input download fails with
  `Connection was closed before we received a valid response`. A single-task
  workflow that writes no output still passes, which makes this easy to miss.
  The patch sets `addressing_style: path` on the three `workflow_*` credentials
  and the `osmo` data credential.

## Host prerequisites

```bash
kubectl create ns osmo && kubectl create ns osmo-test
kubectl create secret generic local-admin-password -n osmo --from-literal=password=<pw>
kubectl create cm mek-config -n osmo --from-file=mek.yaml
deployments/scripts/install-kai-scheduler.sh   # gang scheduling, required
echo "127.0.0.1 quick-start.osmo" | sudo tee -a /etc/hosts
```

Install k3s with `--disable=traefik` so it does not take host port 80; the
gateway needs it.

## Install

```bash
helm install osmo ~/osmo-k3s/charts/quick-start -n osmo \
  -f deployments/values/k3s-local-quickstart.yaml
```

Then `http://quick-start.osmo` serves both the UI and the API
(`/api/version` → 6.3.1).

## Host-level fixes this deployment needed

Each of these broke the cluster in a way that was not obvious from pod status.

**Clash/mihomo TUN hijacked cluster networking.** With `auto-route: true` and
`dns-hijack: [any:53]`, the TUN default route captured the service CIDR, so
CoreDNS upstream queries went through the proxy and `kubernetes.default.svc`
resolved to a fake-ip address. Symptom: every in-cluster connection times out
for no visible reason. Fixed by carving the cluster CIDRs out of the TUN route
with policy routing, made persistent in `/etc/systemd/system/k3s-tun-bypass.service`:

```bash
ip rule add to 10.43.0.0/16 lookup main priority 8000   # services
ip rule add to 10.42.0.0/16 lookup main priority 8001   # pods
```

**Kubelet disk-pressure evicted the whole namespace.** The default eviction
threshold is percentage-based (15% available). On a 290G disk holding ~196G of
unrelated user data, that trips with ~34G still free. Symptom: all pods go
Pending behind a `node.kubernetes.io/disk-pressure` taint. Fixed with absolute
thresholds in `/etc/rancher/k3s/kubelet-eviction.yaml`, wired in via
`--kubelet-arg=config=...`. The file needs `apiVersion` + `kind:
KubeletConfiguration` or k3s refuses to start.

**Workflow pods request `runtimeClassName: nvidia`.** The 6.3.1 image sets this
internally; it is not exposed in any chart value or config API. k3s pre-creates
a matching RuntimeClass but containerd has no such handler, so sandbox creation
fails with `no runtime for "nvidia" is configured`. The RuntimeClass is managed
by the k3s `runtimes` addon and would be reverted if edited, so the handler is
added to containerd instead, via the drop-in
`/var/lib/rancher/k3s/agent/etc/containerd/config-v3.toml.d/10-nvidia-runc.toml`
mapping `nvidia` → runc. A GPU node wants the NVIDIA container toolkit here
instead.

## Where to see per-task metrics

Node placement and the phase-by-phase timing breakdown come from `/api/task`:

```bash
curl -H "x-osmo-user: testuser" \
  "http://quick-start.osmo/api/task?workflow_id=<id>"
```

Each task returns `node`, `duration`, `cpu`, `memory`, `gpu`, `storage`, plus
per-task `logs`. The UI renders the same data at
`http://quick-start.osmo/workflows/<id>`, and `/occupancy` shows cluster-wide
node utilisation.

The logger service persists a finer breakdown to PostgreSQL that the API
summarises away — scheduling, initializing, execution, input download, and
output upload are timestamped separately in `tasks`:

```sql
SELECT name, node_name,
       EXTRACT(epoch FROM (end_time - start_time))                             AS exec_s,
       EXTRACT(epoch FROM (input_download_end_time - input_download_start_time)) AS dl_s,
       EXTRACT(epoch FROM (output_upload_end_time - output_upload_start_time))   AS ul_s
FROM tasks WHERE workflow_id = '<id>' ORDER BY name;
```

`task_io` holds per-transfer byte counts and file counts, but stays empty here:
the logger only records rows for tasks with declared `inputs`/`outputs` data
sections, not the implicit output directory these tutorials use.

### Prometheus and Grafana

Installed as a separate release, since the OSMO charts ship PodMonitors but no
Prometheus to honour them:

```bash
helm upgrade --install monitoring \
  prometheus-community/kube-prometheus-stack -n monitoring --create-namespace \
  --version 88.5.0 -f deployments/values/k3s-local-monitoring.yaml
```

Grafana is at `http://<node-ip>:3000` (admin / `osmo-admin`, a LoadBalancer
served by k3s servicelb). Prometheus has no ingress; reach it with
`kubectl port-forward -n monitoring svc/monitoring-kube-prometheus-prometheus 9090:9090`.

`podMonitor.enabled: true` on both the `service` and `backend-operator`
subcharts turns on scraping of container port 9464. Six OSMO targets report up:
service, worker, agent, delayed-job-monitor, backend-listener, backend-worker.

Load the three shipped dashboards from `docs/deployment_guide/dashboards/` as
ConfigMaps — the stack's sidecar imports anything labelled `grafana_dashboard`:

```bash
kubectl create configmap osmo-dash-workflow-resources-usage -n monitoring \
  --from-file=workflow_resources_usage.json --dry-run=client -o yaml | \
  kubectl label -f - --local -o yaml grafana_dashboard=1 | kubectl apply -f -
```

`observability.grafanaUrl` in the values file is POSTed onto the `default`
backend config, which is what fills in the per-task `grafana_url` returned by
`/api/task`. `generate_grafana_url()` appends `var-namespace`, `var-uuid`,
`from` and `to`, so it must point at a dashboard declaring those variables —
uid `workflow_resources` does. `dashboard_url` stays empty: no Kubernetes
Dashboard is installed.

Two caveats specific to this deployment:

- `osmo_tasks_count` is an observable gauge over a 5-minute task window, so it
  only has series while a workflow is running or just finished. An empty result
  between runs is normal.
- The dashboards filter on a `cluster` label that nothing sets on a single
  cluster (`externalLabels` only apply to federation and remote-write, not to
  locally stored series). Their queries still resolve because an empty matcher
  `cluster=""` also matches series where the label is absent — leave the
  `cluster` variable blank.

Three chart fixes were needed for scraping to work at all, all in the patch
file. The agent and logger deployments each had a stray second `ports:` key that
silently overrode the metrics port (YAML keeps the last duplicate) — still
present on `main`. The agent needs `METRICS_OTEL_ENABLE` via env rather than
argv: `agent_service.py`
loads both `WorkflowServiceConfig` (which carries `MetricsCreatorConfig`) and
`BackendServiceConfig` (which does not) from the same argv, so a `--metrics_*`
flag makes the latter abort on an unrecognized argument. The logger is dropped
from the PodMonitor selector entirely: `LoggerServiceConfig` has no
`MetricsCreatorConfig` and `logger.py` never builds a `MetricCreator`, so nothing
can listen on 9464 in 6.3.1.

There is no Loki in this deployment; task logs come from `/api/task` and
`kubectl logs`.

Note the "multi-machine" view is single-node here: every task reports
`node_name: lwk-ms-7e24`. The metrics are per-task and node-labelled, so they
would spread across nodes unchanged on a real multi-node backend.

## Security note

`gateway.envoy.defaultIdentity` grants every request `testuser` /
`osmo-admin` with oauth2Proxy and authz both disabled — the gateway **trusts
client-supplied `x-osmo-*` headers**. Only safe because it is bound to a
workstation. Do not expose this gateway to an untrusted network.

## Known unclean edge

The `osmo-backend-operator-token` job hardcodes `BASE_URL="http://osmo-service"`,
but the service runs with `--ssl_self_signed`, so its port 80 speaks TLS. Under
`set -e` the plain-HTTP curl exits 52 and the script dies before printing a
diagnostic, leaving backend-listener/worker stuck in `Init` forever. The patch
uses `https://osmo-service:80` (the port must be explicit or curl defaults to
443, where nothing listens), `curl -sk` for the self-signed cert, and `|| true`
on the token test so a stale secret falls through to the recreate path. This
looks like an upstream bug worth reporting.
