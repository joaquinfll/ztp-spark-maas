# ztp-spark-maas

GitOps repository for Models-as-a-Service on single-node OpenShift 4.22 on an NVIDIA DGX Spark (`spark-sno.home.arpa`), synced by the default OpenShift GitOps (Argo CD) instance.

## Bootstrap

The agent-based installer (`openshift/` manifests in the cluster install directory) sets up everything this repo needs:

- `50-gitops-operator.yaml`: OpenShift GitOps operator, channel `gitops-1.22`.
- `60-gitops-argocd-cluster-admin.yaml`: binds `cluster-admin` to the Argo CD application controller (`openshift-gitops-argocd-application-controller`). Anyone who can push to this repo effectively owns the cluster; replace it with a scoped ClusterRole if this stops being a lab.
- `99-spark-sno-root-application.yaml`: a Job that waits for the Argo CD CRD and creates the root Application `spark-sno` (`clusters/spark-sno`, `main`).

## Layout

```
clusters/spark-sno/          app of apps, one Application per component
components/
  lvms/operator|instance     LVM Storage on /dev/nvme0n1p5, default StorageClass lvms-vg1
  metallb/operator|instance  MetalLB, pool maas-pool = 192.168.6.21 (node IP + 1), L2
  gpu/operators|instances    NFD + NVIDIA GPU Operator (v26.7), NodeFeatureDiscovery, ClusterPolicy
  maas-operators/            cert-manager, Leader Worker Set, Connectivity Link, RHOAI (stable-3.x)
  maas-platform/             Kuadrant/Authorino, user workload monitoring, Gateway, passthrough Route, fixups Job
  maas-db/                   PostgreSQL + Job that generates the DB secrets in-cluster
  rhoai-config/              DSCI (monitoring metrics/traces), DSC (MaaS enabled), HardwareProfile, dashboard flags
  models/                    llm namespace, agentic-coders Group, Qwen3.6-35B-A3B NVFP4, MaaS tiers
  observability/operators    Cluster Observability, OpenTelemetry, Tempo, Loki operators
  observability/config       Versity S3 + LokiStack (MaaS usage logs), gateway telemetry and usage logging
  openshift-mcp-server/      Red Hat OpenShift MCP server Helm values + dashboard registration
```

| Wave | Applications |
|------|--------------|
| 0 | `lvms-operator`, `metallb-operator`, `gpu-operators`, `maas-operators`, `observability-operators` |
| 1 | `lvms-instance`, `metallb-instance`, `gpu-instances` |
| 2 | `maas-platform` |
| 3 | `maas-db` |
| 4 | `rhoai-config` |
| 5 | `models`, `observability` |
| 6 | `openshift-mcp-server` |

Waves only order creation. Every child Application has automated prune and self-heal, `SkipDryRunOnMissingResource=true`, and retries 10 times (30s to 5m backoff, about 40 minutes). If a sync still fails after that, Argo CD waits for a new commit or a manual sync. A first install takes 30-60 minutes (operators, GPU driver build, model pull and load).

## Notes

- **Jobs:** `maas-db/secrets-job.yaml` and `maas-platform/fixups-job.yaml` cover steps with no declarative equivalent (DB password, Authorino TLS env, Kuadrant restart). They are idempotent and use the pinned 4.22.18 aarch64 `oc` image. Jobs are immutable: delete the Job in the cluster before changing its spec.
- **Observability:** RHOAI dashboards (Perses), metrics and traces in `redhat-ods-monitoring`; MaaS usage logs in LokiStack `usage`, stored on a Versity S3 gateway (`observability-s3`, 50Gi LVMS volume, credentials generated in-cluster). `captureUser` is off in `maas-telemetry.yaml`; the MaaS UI Usage tab needs it on.
- **Never pruned:** `redhat-ods-applications`, `redhat-ai-gateway-infra` and `llm` carry `Prune=false`.
- **OpenShift MCP server:** Red Hat chart `redhat-openshift-mcp-server` 0.3.0 (charts.openshift.io), release `redhat-openshift-mcp-server` in `llm`. Read-only (`read_only`, `disable_destructive`, Secrets denied) and bound to `cluster-reader`. In-cluster only at `http://redhat-openshift-mcp-server.llm.svc.cluster.local:8080/mcp`, listed on the dashboard through the `gen-ai-aa-mcp-servers` ConfigMap. Argo CD renders the chart, so don't manage the release with `helm`.

## Changing Things

- **Agentic coders:** add users to `components/models/group-agentic-coders.yaml`; they mint new API keys afterwards.
- **Rate limits:** `components/models/qwen36-35b-nvfp4/maas/maas-subscription-*.yaml`.
- **Another model:** add a directory under `components/models/` and list it in its `kustomization.yaml`.
