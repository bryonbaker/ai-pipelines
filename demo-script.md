# Remote Pipeline GitOps Demo

This demo walks through manually deploying a pipeline definition and triggering a run,
simulating what ACM would do via ManifestWork.

All commands run against the `pipeline-demo` namespace.

## Prerequisites

- Logged into the OpenShift cluster (`oc whoami` succeeds)
- The DSPA `pipelines-definition` is deployed and healthy in `pipeline-demo`

## Known Behavior

When PipelineVersions are created via `oc apply` (rather than the KFP upload API),
the DSPA controller sets the ownerReference and `pipeline-id` label but does NOT write
`status.conditions`. The PipelineVersion is fully functional — it just lacks a READY
status on the CR. The run-trigger Job checks for the `pipeline-id` label instead.

The DSPA has `podToPodTLS: true`, so all in-cluster API calls use HTTPS with the CA
from the `dsp-trusted-ca-pipelines-definition` ConfigMap.

## Step 1: Deploy the Pipeline Definition

Apply the RBAC, Pipeline, and PipelineVersion (order matters — Pipeline before PipelineVersion):

```bash
oc apply -f pipeline/rbac.yaml
oc apply -f pipeline/remote-pipeline.yaml
oc apply -f pipeline/remote-pipeline-pv.yaml
```

Verify the controller has reconciled the PipelineVersion (look for the `pipeline-id` label):

```bash
oc get pipelineversions.pipelines.kubeflow.org remote-pipeline -n pipeline-demo \
  -o jsonpath='{.metadata.labels.pipelines\.kubeflow\.org/pipeline-id}' ; echo
```

Expected output: the UID of the Pipeline (e.g. `03194203-8650-40d8-a1a5-5688867b37b2`).
If empty, wait a few seconds and retry — the controller needs a moment to reconcile.

Verify both resources exist:

```bash
oc get pipelines.pipelines.kubeflow.org,pipelineversions.pipelines.kubeflow.org \
  -n pipeline-demo | grep remote-pipeline
```

## Step 2: Trigger a Run

Apply the run-trigger Job:

```bash
oc apply -f runs/run-001.yaml
```

Watch the Job complete:

```bash
oc wait --for=condition=complete job/remote-pipeline-run-001 \
  -n pipeline-demo --timeout=180s
```

Check the Job logs to confirm the run was created:

```bash
oc logs job/remote-pipeline-run-001 -n pipeline-demo
```

You should see the Pipeline UID, PipelineVersion UID, and the JSON response from the
KFP API confirming the run was created.

## Step 3: Verify the Run

Watch the Argo Workflow execute:

```bash
oc get workflows.argoproj.io -n pipeline-demo --watch
```

The workflow progresses through: data-prep → train-model → evaluate-model + register-model.
All four steps sleep for 10 seconds each, so the full run takes about 2-3 minutes.

## Cleanup

Delete everything in reverse order so the demo can be run again from scratch.

Delete the run-trigger Job:

```bash
oc delete job remote-pipeline-run-001 -n pipeline-demo --ignore-not-found
```

Delete any Argo Workflows created by the run:

```bash
oc delete workflows.argoproj.io -n pipeline-demo -l pipelines.kubeflow.org/v2_component=true
```

Delete the PipelineVersion (must go before the Pipeline — it has an ownerReference):

```bash
oc delete pipelineversions.pipelines.kubeflow.org remote-pipeline -n pipeline-demo --ignore-not-found
```

Delete the Pipeline:

```bash
oc delete pipelines.pipelines.kubeflow.org remote-pipeline -n pipeline-demo --ignore-not-found
```

Delete the RBAC resources:

```bash
oc delete rolebinding kfp-run-trigger -n pipeline-demo --ignore-not-found
oc delete role kfp-run-trigger -n pipeline-demo --ignore-not-found
oc delete sa kfp-run-trigger -n pipeline-demo --ignore-not-found
```

Verify everything is gone:

```bash
oc get pipelines.pipelines.kubeflow.org,pipelineversions.pipelines.kubeflow.org \
  -n pipeline-demo | grep remote-pipeline
oc get job -n pipeline-demo | grep remote-pipeline
```

Both commands should return no results. The demo is now ready to run again from Step 1.
