# rhoai-distributed

GitOps-friendly manifests for **distributed** OpenShift AI workloads: JobSet (Kubeflow Trainer v2 prerequisite), default Kueue queues, and an optional Data Science Pipelines server backed by the MinIO deployment from [rhoai-deploy](https://github.com/redhat-ai-americas/rhoai-deploy).

This repo is consumed by [rhoai-demo-foundations](https://github.com/redhat-ai-americas/rhoai-demo-foundations) (app-of-apps). You can also apply individual Applications under `gitops/applications/` for testing.

## Layout

| Path | Purpose |
|------|---------|
| `platform/jobset-operator/subscription/` | Namespace, OperatorGroup, Subscription |
| `platform/jobset-operator/instance/` | `JobSetOperator` operand (sync manually after the operator CSV is ready) |
| `platform/kueue/cluster/` | `ClusterQueue` named `default` |
| `platform/kueue/local-demo/` | `LocalQueue` `default` in namespace `demo` (sync after `demo` exists) |

| `platform/pipelines/` | `DataSciencePipelinesApplication` in `demo` using MinIO bucket `pipelines` (`minio-dspa-connection` Secret managed by ESO via rhoai-deploy `access`) |
| `gitops/applications/` | Standalone Argo CD `Application` manifests |

## Prerequisites

- OpenShift with [rhoai-deploy](https://github.com/redhat-ai-americas/rhoai-deploy) (or equivalent): RHOAI operator, `DataScienceCluster` with `aipipelines`, `trainingoperator`, **`trainer`**, and **Ray** **Managed** in Git (first sync); Argo CD then ignores live `default-dsc` **spec** drift so you can customize on-cluster.
- Argo CD with `ServerSideApply=true` on apps that touch the DSC.
- **JobSet Operator** (OpenShift documentation): cert-manager Operator for Red Hat OpenShift is required before the JobSet Operator installs successfully.

## MinIO and pipelines credentials

`Secret/minio-dspa-connection` in namespace `demo` is managed automatically by the **External Secrets Operator** via the `access` Application in [rhoai-deploy](https://github.com/redhat-ai-americas/rhoai-deploy). It reads `MINIO_ROOT_USER` / `MINIO_ROOT_PASSWORD` from `minio/minio-secret` and maps them to `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` with a 5-minute refresh interval.

**Prerequisites before syncing `pipelines-server`:**

1. `minio-secret` exists in `minio` (created before MinIO sync — see [rhoai-deploy README](https://github.com/redhat-ai-americas/rhoai-deploy/blob/main/README.md)).
2. The `minio` Application has been synced so the `pipelines` bucket exists.
3. The `access` Application has been synced so ESO is running and `minio-dspa-connection` has been reconciled in `demo`.

In [rhoai-demo-foundations](https://github.com/redhat-ai-americas/rhoai-demo-foundations) these are handled by sync waves: minio (wave 4) → access (wave 5) → pipelines-server (wave 12, manual sync).

After those conditions are met, **Sync** the `pipelines-server` Application in Argo CD. The `pipelines-server` GitOps Application sets **`ignoreDifferences` on `DataSciencePipelinesApplication` `dspa` `/spec`**, so you can change or remove the pipelines server on the cluster without drift-driven churn; **Sync** the Application to re-apply `platform/pipelines/dspa.yaml`.

## Ray and the DataScienceCluster

The base DSC in rhoai-deploy enables **Ray** and **Trainer v2** (`trainer`) in Git. The `rhoai-instance` Application uses **`ignoreDifferences` on `default-dsc` `/spec`**, so live edits (including turning Ray or Trainer off) are not continuously reconciled away; **Sync** `rhoai-instance` to reset from Git.

## Kubeflow Trainer

With **`trainer`** and `trainingoperator` **Managed** on the DSC and JobSet installed, use cluster-scoped training runtimes (for example `oc get clustertrainingruntime`). See Red Hat OpenShift AI documentation for *Running Kubeflow Trainer v2-based distributed training workloads*.

## Troubleshooting

- If `JobSetOperator` fails to apply, confirm the API with `oc api-resources | grep -i jobset` and create the operator instance from the OperatorHub UI if your OpenShift version uses a different API shape.
- If Kueue resources fail, confirm the Red Hat Kueue Operator is installed (pulled in via rhoai-deploy dependencies).

## License

Apache License 2.0
