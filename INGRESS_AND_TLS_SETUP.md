# Ingress and TLS Setup Guide

This guide covers ingress and TLS prerequisites for Corrino Lens Helm chart installation.

## Overview

The Helm chart automatically installs as dependencies:
- **ingress-nginx** (v4.13.2) - Kubernetes Ingress Controller in `lens` namespace
- **cert-manager** (v1.13.2) - TLS certificate management in `lens` namespace  
- **Let's Encrypt ClusterIssuer** - Production ACME certificates (cluster-wide)
- **IngressClass** `lens-nginx` (cluster-wide)

**Architecture:**
```
Internet → [LoadBalancer] → [ingress-nginx] → [Ingress] → [Apps]
```

**Important:** Deleting the `lens` namespace is safe and removes all namespace-scoped resources. Only delete cluster-wide resources (IngressClass, ClusterIssuer, CRDs) if certain no other applications use them.

---

## Pre-Install Check

Check for existing resources before installation:

```bash
# Check if lens namespace exists
kubectl get ns lens 2>&1

# Check for existing pods in lens namespace
kubectl get pods -n lens 2>&1

# Check cluster-wide resources
kubectl get ingressclass lens-nginx 2>&1
kubectl get clusterissuer letsencrypt-prod 2>&1
kubectl get crd | grep cert-manager
```

**If any resources exist from a previous installation, proceed to cleanup.**

---

## Clean Up if Exists

### Option 1: Safe Cleanup (Recommended)

Removes only the `lens` namespace, preserving cluster-wide resources:

```bash
# Uninstall Helm release
helm uninstall lens -n lens

# Delete namespace (removes all namespace-scoped resources)
kubectl delete namespace lens

# Wait for deletion to complete
kubectl wait --for=delete namespace/lens --timeout=120s 2>/dev/null || echo "Namespace deleted"
```

**Verification:**
```bash
kubectl get namespace lens 2>&1 | grep "NotFound" && echo "✅ Ready for install" || echo "❌ Namespace still exists"
```

### Option 2: Complete Cleanup (Fresh Cluster Only)

⚠️ **Use only if no other applications use cert-manager or ingress-nginx**

```bash
# 1. Uninstall Helm release and namespace
helm uninstall lens -n lens
kubectl delete namespace lens
kubectl wait --for=delete namespace/lens --timeout=120s 2>/dev/null

# 2. Verify no other apps are using these resources
kubectl get certificate --all-namespaces
kubectl get ingress --all-namespaces

# 3. Delete cluster-wide resources (if safe)
kubectl delete clusterissuer letsencrypt-prod --ignore-not-found
kubectl delete ingressclass lens-nginx --ignore-not-found

# 4. Delete cert-manager CRDs (affects entire cluster!)
kubectl get crd | grep cert-manager | awk '{print $1}' | xargs kubectl delete crd --ignore-not-found

# 5. Delete webhook configurations
kubectl delete validatingwebhookconfiguration -l app.kubernetes.io/instance=lens --ignore-not-found
kubectl delete mutatingwebhookconfiguration -l app.kubernetes.io/instance=lens --ignore-not-found
```

**Verification:**
```bash
kubectl get namespace lens 2>&1 | grep "NotFound"
kubectl get ingressclass lens-nginx 2>&1 | grep "NotFound"
kubectl get crd | grep cert-manager || echo "No cert-manager CRDs"
```

---

## Post-Install Check

After helm installation, verify all components are running:

```bash
# 1. Verify namespace and pods
kubectl get namespace lens
kubectl get pods -n lens

# 2. Check ingress-nginx and cert-manager are running
kubectl get pods -n lens | grep -E 'ingress|cert-manager'

# 3. Get LoadBalancer external IP (may take 1-2 minutes)
kubectl get svc -n lens -l app.kubernetes.io/component=controller

# 4. Check cluster-wide resources
kubectl get ingressclass lens-nginx
kubectl get clusterissuer letsencrypt-prod

# 5. View all ingress endpoints
kubectl get ingress -n lens

# 6. Check TLS certificates (should show READY=True after 2-5 minutes)
kubectl get certificate -n lens
```

**Expected output:**
- All pods in `Running` state
- LoadBalancer service has an `EXTERNAL-IP`
- Certificates show `READY=True`
- Ingress resources show correct hosts

**If certificates not ready:**
```bash
# Check certificate details and challenges
kubectl describe certificate -n lens
kubectl get challenge -n lens
kubectl logs -n lens -l app=cert-manager --tail=50
```

---

## Complete Uninstall

### Standard Uninstall (Preserves Cluster-Wide Resources)

```bash
helm uninstall lens -n lens
kubectl delete namespace lens
```

### Full Uninstall (Removes Everything)

⚠️ **Only use on dedicated test clusters**

```bash
# 1. Uninstall Helm release
helm uninstall lens -n lens

# 2. Delete namespace
kubectl delete namespace lens

# 3. Wait for namespace deletion
kubectl wait --for=delete namespace/lens --timeout=120s 2>/dev/null

# 4. Delete cluster-wide resources
kubectl delete clusterissuer letsencrypt-prod --ignore-not-found
kubectl delete ingressclass lens-nginx --ignore-not-found

# 5. Delete cert-manager CRDs
kubectl get crd | grep cert-manager | awk '{print $1}' | xargs kubectl delete crd --ignore-not-found

# 6. Delete webhook configurations  
kubectl delete validatingwebhookconfiguration -l app.kubernetes.io/instance=lens --ignore-not-found
kubectl delete mutatingwebhookconfiguration -l app.kubernetes.io/instance=lens --ignore-not-found
```

**Verify complete removal:**
```bash
kubectl get all -n lens 2>&1 | grep "NotFound" && echo "✅ Complete uninstall successful"
```

---

## Troubleshooting

### Namespace Stuck in "Terminating"
```bash
# Check what's blocking deletion
kubectl get all -n lens
kubectl get certificate -n lens

# Force remove finalizers (use with caution)
kubectl get namespace lens -o json | jq '.spec.finalizers = []' | kubectl replace --raw /api/v1/namespaces/lens/finalize -f -
```

### Certificates Not Getting Issued
```bash
# Check status and challenges
kubectl describe certificate -n lens
kubectl get challenge -n lens

# Check cert-manager logs
kubectl logs -n lens -l app=cert-manager -f

# Common causes:
# - DNS not pointing to LoadBalancer IP
# - Firewall blocking port 80/443 from internet
# - Let's Encrypt rate limits exceeded
```

### IngressClass or ClusterIssuer Already Exists
```bash
# Check if from previous lens installation
kubectl get ingressclass lens-nginx -o yaml | grep app.kubernetes.io/instance

# Delete if from previous lens install
kubectl delete ingressclass lens-nginx
kubectl delete clusterissuer letsencrypt-prod
```

### Check if Other Apps Use Resources
```bash
# List all ingress and certificates across cluster
kubectl get ingress --all-namespaces
kubectl get certificate --all-namespaces

# If only lens namespace appears, safe to delete cluster-wide resources
```

---

## Additional Resources

- **ingress-nginx**: https://kubernetes.github.io/ingress-nginx/
- **cert-manager**: https://cert-manager.io/docs/
- **Let's Encrypt**: https://letsencrypt.org/docs/
- **Report Issues**: https://github.com/oci-ai-incubations/corrino-lens-devops
