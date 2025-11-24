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

This will:
- Create the `rhacs-operator` namespace
- Create an OperatorGroup for the namespace
- Subscribe to the `rhacs-operator` from the Red Hat operators catalog
- Subscribe to the  `openshift-compliance` operator (and create respective namespace).

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

This will:
- Create the `stackrox` namespace
- Create a Central custom resource instance
- Trigger the operator to deploy the Central component

### Step 4: Verify Installation
Monitor the Central deployment:

```bash
oc get pods -n stackrox
```

Or watch all resources:
```bash
watch oc get pvc,deploy,svc,route -n stackrox
```

`scanner-db` will take the longest to initialize (~5+ minutes)

The Central component will take several minutes to fully deploy. Once all pods are running, you can access the Central UI through the route that was created.

To check the status of the Central instance via the CLI

```bash
oc get central stackrox-central-services -o jsonpath-as-json='{.status.conditions}' -n stackrox
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

### Step 5: Log in to the RHACS portal as the admin user

Retrieve the admin user's password from the central-htpasswd secret

```bash
oc -n stackrox extract secret/central-htpasswd \
  --keys password --to -
```

Example output:
```bash
# password
UIooqQXXrcxX0mzDuYUPCo6uX
```
Retrieve the URL of the RHACS portal from the central route

```bash
oc -n stackrox get route central -o jsonpath='https://{.spec.host}'
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

## Securing Clusters

After Central is deployed and accessible, you can secure additional clusters. This process involves:
1. Creating a cluster init bundle from the Central UI
2. Applying the init bundle to the cluster you want to secure
3. Deploying a SecuredCluster custom resource

### Step 1: Create Cluster Init Bundle

From the ACS console, navigate to `Platform Configuration` → `Clusters`, then click the `Init bundles` button.

From the Cluster init bundles page, click `Create bundle`.

Provide a name (e.g., `install-import`), select `OpenShift`, and click `Download`.

This will download a file to your local system called `install-import-Operator-secrets-cluster-init-bundle.yaml`.

**Important:** Keep this file secure as it contains authentication credentials for connecting clusters to Central.

### Step 2: Secure the Central Cluster (Self-Management)

To have Central manage itself as a secured cluster, apply the init bundle and SecuredCluster resource on the Central cluster:

```bash
# Apply the init bundle secrets
oc -n stackrox apply -f install-import-Operator-secrets-cluster-init-bundle.yaml

# Deploy the SecuredCluster resource for the central cluster
oc apply -k k8/secured-cluster/base/
```

Expected output when applying the init bundle:
```
secret/collector-tls created
secret/sensor-tls created
secret/admission-control-tls created
```

Eventually, the cluster will show up in ACS as `Healthy` and `Up to date with Central`.

### Step 3: Secure Additional Clusters

To secure additional clusters (managed clusters), you need to install the ACS operator and deploy a SecuredCluster resource on each cluster.

**Note:** This process can be automated with ACM (Advanced Cluster Management) policies, which is out of scope for this guide.

#### On Each Managed Cluster:

1. **Log in to the managed cluster:**
   ```bash
   oc login -u admin -p <password> https://api.<domain-of-managed-cluster>.com:6443
   ```

2. **Install the RHACS Operator:**
   ```bash
   oc apply -k k8/operators/
   ```
   
   Wait for the operator to be ready:
   ```bash
   oc get pods -n rhacs-operator
   ```

3. **Create the `stackrox` namespace:**
   ```bash
   oc apply -f k8/central/namespace.yaml
   ```

4. **Apply the init bundle (downloaded in Step 1):**
   ```bash
   oc apply -f install-import-Operator-secrets-cluster-init-bundle.yaml -n stackrox
   ```

5. **Deploy the SecuredCluster resource:**
   
   For the first managed cluster, edit the overlay file:
   ```bash
   # Edit the managed-cluster.yaml file
   vi k8/secured-cluster/overlays/secured-cluster01/managed-cluster.yaml
   ```
   
   Update the `centralEndpoint` with your Central cluster's endpoint:
   - Use the same hostname as the Central UI route
   - Remove the `https://` prefix
   - Add `:443` port
   
   Example:
   ```yaml
   apiVersion: platform.stackrox.io/v1alpha1
   kind: SecuredCluster
   metadata:
     name: stackrox-secured-cluster-services
     namespace: stackrox
   spec:
     clusterName: secured-cluster01
     # Update with your central cluster endpoint (route hostname + :443)
     centralEndpoint: central-stackrox.apps.cluster-8cj8w.8cj8w.sandbox3260.opentlc.com:443
   ```
   
   Then apply it:
   ```bash
   oc apply -f k8/secured-cluster/overlays/secured-cluster01/managed-cluster.yaml
   ```
   
   For additional clusters, you can:
   - Use the `secured-cluster02` overlay directory
   - Create a new overlay following the same pattern
   - Or directly apply a customized SecuredCluster YAML

6. **Monitor the deployment:**
   ```bash
   oc get pods -n stackrox
   oc get pvc,deploy,svc,route -n stackrox
   ```
   
   **Note:** The full deployment might be resource-intensive for Single Node OpenShift (SNO) clusters. Monitor resource usage and scale down components if needed (see resource scaling section above).


## Directory Structure

```
k8/
├── operators/
│   ├── kustomization.yaml          # Kustomize config for operator resources
│   ├── acs.yaml                     # ACS operator installation manifest
│   └── compliance-operator.yaml     # Compliance operator namespace and subscription
├── central/
│   ├── kustomization.yaml           # Kustomize config for Central resources
│   ├── namespace.yaml                # StackRox namespace definition
│   └── central.yaml                  # Central component deployment manifest
└── secured-cluster/
    ├── base/                         # Base SecuredCluster configuration
    │   ├── kustomization.yaml
    │   └── secured-cluster.yaml      # Base SecuredCluster CR (clusterName: central-cluster)
    └── overlays/                      # Overlays for different managed clusters
        ├── secured-cluster01/
        │   └── managed-cluster.yaml  # SecuredCluster for first managed cluster
        └── secured-cluster02/        # Overlay directory for second managed cluster
```

### Usage Notes

- **Base configuration**: Use `k8/secured-cluster/base/` to deploy a SecuredCluster on the Central cluster itself
- **Overlays**: Use overlay directories for managed clusters with different cluster names and central endpoints
- **Kustomize**: All directories support `oc apply -k` for applying resources
