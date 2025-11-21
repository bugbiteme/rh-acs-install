# rh-acs-install
Installation of Red Hat Advanced Cluster Security

## Overview
This repository contains Kubernetes manifests for installing Red Hat Advanced Cluster Security (ACS) on OpenShift clusters.

## Prerequisites
- Access to an OpenShift cluster
- `oc` CLI tool installed
- Cluster admin permissions
- Access to the `redhat-operators` catalog source

## Installation Steps

### Step 1: Install the ACS Operator
First, install the Red Hat Advanced Cluster Security operator on your central cluster:

```bash
oc apply -k k8/operators/
```

Alternatively, you can apply the file directly:

```bash
oc apply -f k8/operators/acs.yaml
```

This will:
- Create the `rhacs-operator` namespace
- Create an OperatorGroup for the namespace
- Subscribe to the `rhacs-operator` from the Red Hat operators catalog

### Step 2: Wait for Operator Installation
Wait for the operator to be fully installed and ready. You can check the status with:

```bash
oc get pods -n rhacs-operator
```

Wait until the operator pod is in `Running` state and ready (typically takes 1-2 minutes).

### Step 3: Create Central Resources
Once the operator is ready, create the Central component:

```bash
oc apply -k k8/central/
```

Alternatively, you can apply the files individually:

```bash
oc apply -f k8/central/namespace.yaml
oc apply -f k8/central/central.yaml
```

This will:
- Create the `stackrox` namespace
- Create a Central custom resource instance
- Trigger the operator to deploy the Central component

### Step 4: Verify Installation
Monitor the Central deployment:

```bash
oc get pods -n stackrox
```
```bash
watch oc get pvc,deploy,svc,route
```

The Central component will take several minutes to fully deploy. Once all pods are running, you can access the Central UI through the route that was created.

To check the status of the Central instance via the CLI

```bash
oc get central stackrox-central-services \ 
  -o jsonpath-as-json='{.status.conditions}'
```

Output:
```json
[
    [
        {
            "lastTransitionTime": "2025-11-21T18:20:34Z",
            "message": "StackRox Central Services has been installed.\n\n\n\n\nStackRox Kubernetes Security Platform collects and transmits anonymous usage and\nsystem configuration information. If you want to OPT OUT from this, use\n--set central.telemetry.enabled=false.\n\n\nThank you for using StackRox!\n",
            "reason": "InstallSuccessful",
            "status": "True",
            "type": "Deployed"
        },
...
```

### Step 4: Log in to the RHACS portal as the admin user

Retrieve the admin user's password from the central-htpasswd secret

```bash
oc extract secret/central-htpasswd \
  --keys password --to -
```

Example output:
```bash
# password
UIooqQXXrcxX0mzDuYUPCo6uX
```
Retrieve the URL of the RHACS portal from the central route

```bash
oc get route central -n stackrox -o jsonpath='https://{.spec.host}'
```

Example Output:
```
https://central-stackrox.apps.cluster-8cj8w.8cj8w.sandbox3260.opentlc.com
```

log in with `admin` and the extracted password

***Note*** If you see the error `The database is currently not available. If this problem persists, please contact support.`
You might not have enough cluster resources to launch the `central-db` deployment. If testing you may be able to scale down the following resources, otherwise provision mode nodes

```bash
oc patch hpa scanner -n stackrox \
  --type=merge \
  -p '{"spec":{"minReplicas":1,"maxReplicas":1}}'

oc patch hpa scanner-v4-indexer -n stackrox \
  --type=merge \
  -p '{"spec":{"minReplicas":1,"maxReplicas":1}}'

oc patch hpa scanner-v4-matcher -n stackrox \
  --type=merge \
  -p '{"spec":{"minReplicas":1,"maxReplicas":1}}'

# Classic scanner: 2 → 1
oc scale deploy scanner --replicas=1

# v4 indexer: 2 → 1
oc scale deploy scanner-v4-indexer --replicas=1

# v4 matcher: 2 → 1
oc scale deploy scanner-v4-matcher --replicas=1

# Optional: if still tight, you can even scale scanner dbs a bit
# but usually just replica cuts are enough.
```

## Directory Structure
- `k8/operators/kustomization.yaml` - Kustomize configuration for operator resources
- `k8/operators/acs.yaml` - Operator installation manifest
- `k8/central/kustomization.yaml` - Kustomize configuration for Central resources
- `k8/central/namespace.yaml` - StackRox namespace definition
- `k8/central/central.yaml` - Central component deployment manifest
