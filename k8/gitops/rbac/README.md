# ArgoCD RBAC Configuration for OpenShift

This directory contains the ServiceAccount, ClusterRole, and ClusterRoleBinding required for ArgoCD to install operators and CRDs on OpenShift clusters.

## Resources

- **serviceaccount.yaml**: ServiceAccount for ArgoCD Application Controller
- **clusterrole.yaml**: ClusterRole with permissions for:
  - Namespace management
  - Operator Lifecycle Manager (OLM) resources (OperatorGroups, Subscriptions, InstallPlans, CSVs)
  - Custom Resource Definitions (CRDs)
  - ACS Custom Resources (Central, SecuredCluster, etc.)
  - Compliance operator resources
  - Standard Kubernetes resources (Pods, Services, Deployments, etc.)
  - OpenShift Routes
  - RBAC resources
- **clusterrolebinding.yaml**: Binds the ServiceAccount to the ClusterRole

## Configuration Steps

After deploying these RBAC resources, you need to configure ArgoCD to use the service account.

### Option 1: Using ArgoCD Operator (Recommended for OpenShift)

If you're using the ArgoCD Operator, patch the ArgoCD CR to use the service account:

```bash
oc patch argocd argocd -n argocd --type=merge -p '
spec:
  controller:
    serviceAccount: argocd-application-controller
'
```

### Option 2: Manual Patch (If not using ArgoCD Operator)

Patch the ArgoCD Application Controller deployment directly:

```bash
oc patch deployment argocd-application-controller -n argocd --type=json -p='
[
  {
    "op": "replace",
    "path": "/spec/template/spec/serviceAccountName",
    "value": "argocd-application-controller"
  }
]'
```

### Option 3: Using ArgoCD CLI

If you have access to the ArgoCD CLI:

```bash
argocd account update-password --account argocd-application-controller
```

## Verification

After configuration, verify the service account is being used:

```bash
# Check the deployment
oc get deployment argocd-application-controller -n argocd -o jsonpath='{.spec.template.spec.serviceAccountName}'

# Should output: argocd-application-controller

# Check the pod
oc get pod -n argocd -l app.kubernetes.io/name=argocd,app.kubernetes.io/component=application-controller -o jsonpath='{.items[0].spec.serviceAccountName}'

# Should output: argocd-application-controller
```

## Deployment Order

The RBAC resources are automatically deployed first (sync-wave: 0) before operators and central applications, ensuring permissions are in place before they're needed.

