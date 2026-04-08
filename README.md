# rhoai-distributed

GitOps-friendly manifests for **distributed** OpenShift AI workloads: JobSet (Kubeflow Trainer v2 prerequisite), default Kueue queues, optional Ray on the `DataScienceCluster`, and an optional Data Science Pipelines server backed by the MinIO deployment from [rhoai-deploy](https://github.com/redhat-ai-americas/rhoai-deploy).

This repo is consumed by [rhoai-demo-foundations](https://github.com/redhat-ai-americas/rhoai-demo-foundations) (app-of-apps). You can also apply individual Applications under `gitops/applications/` for testing.

## Layout

| Path | Purpose |
|------|---------|
| `platform/jobset-operator/subscription/` | Namespace, OperatorGroup, Subscription |
| `platform/jobset-operator/instance/` | `JobSetOperator` operand (sync manually after the operator CSV is ready) |
| `platform/kueue/cluster/` | `ClusterQueue` named `default` |
| `platform/kueue/local-demo/` | `LocalQueue` `default` in namespace `demo` (sync after `demo` exists) |
| `platform/ray/` | DSC patch: `ray: Managed` (manual sync; base DSC still lists `ray: Removed` in Git) |
| `platform/pipelines/` | `DataSciencePipelinesApplication` in `demo` using MinIO bucket `pipelines` |
| `gitops/applications/` | Standalone Argo CD `Application` manifests |

## Prerequisites

- OpenShift with [rhoai-deploy](https://github.com/redhat-ai-americas/rhoai-deploy) (or equivalent): RHOAI operator, `DataScienceCluster` with `aipipelines` and `trainingoperator` **Managed**.
- Argo CD with `ServerSideApply=true` on apps that touch the DSC.
- **JobSet Operator** (OpenShift documentation): cert-manager Operator for Red Hat OpenShift is required before the JobSet Operator installs successfully.

## MinIO and pipelines credentials

Pipelines object storage uses the **same root credentials** as the MinIO server. Create `minio-secret` in namespace `minio` exactly as described in the **rhoai-deploy** README (`MINIO_ROOT_USER` / `MINIO_ROOT_PASSWORD`), sync MinIO, then expose those credentials to namespace `demo` for the DSPA:

```bash
S3U=$(oc get secret minio-secret -n minio -o jsonpath='{.data.MINIO_ROOT_USER}' | base64 -d)
S3P=$(oc get secret minio-secret -n minio -o jsonpath='{.data.MINIO_ROOT_PASSWORD}' | base64 -d)
oc create secret generic minio-dspa-connection -n demo \
  --from-literal=AWS_ACCESS_KEY_ID="${S3U}" \
  --from-literal=AWS_SECRET_ACCESS_KEY="${S3P}" \
  --dry-run=client -o yaml | oc apply -f -
```

After `minio-create-buckets` has created the `pipelines` bucket, manually sync the `pipelines-server` Application.

## Ray and the DataScienceCluster

The base DSC in rhoai-deploy keeps `ray: Removed` as the checked-in default. The `platform/ray` patch sets `ray: Managed`. Argo CD must **ignore** drift on `/spec/components/ray` for the Application that syncs `platform/rhoai-operator/instance` (see rhoai-deploy `gitops/platform/rhoai-platform.yaml` and rhoai-demo-foundations `rhoai-instance` Application).

## Kubeflow Trainer

With `trainingoperator: Managed` on the DSC and JobSet installed, use cluster-scoped training runtimes (for example `oc get clustertrainingruntime`). See Red Hat OpenShift AI documentation for *Running Kubeflow Trainer v2-based distributed training workloads*.

## Troubleshooting

- If `JobSetOperator` fails to apply, confirm the API with `oc api-resources | grep -i jobset` and create the operator instance from the OperatorHub UI if your OpenShift version uses a different API shape.
- If Kueue resources fail, confirm the Red Hat Kueue Operator is installed (pulled in via rhoai-deploy dependencies).

## License

Apache License 2.0
