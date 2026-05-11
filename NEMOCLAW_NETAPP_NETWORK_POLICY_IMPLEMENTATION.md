# NVIDIA NeMoCLAW + NetApp ONTAP NetworkPolicy Implementation Guide

This guide explains how to implement the custom Kubernetes `NetworkPolicy` in an **existing NVIDIA NeMoCLAW deployment** so NeMoCLAW can reach NetApp ONTAP APIs for monitoring and configuration workflows while keeping network access restricted.

---

## Prerequisites

1. A running Kubernetes cluster with your NeMoCLAW workload deployed.
2. `kubectl` access with permissions to read/apply resources in the target namespace.
3. The NeMoCLAW namespace name (examples below use `nvidia-nemoclaw`).
4. NetApp ONTAP management endpoint details:
   - Management LIF IP/CIDR ranges
   - Required protocols/ports (typically HTTPS `443`)
5. Existing label used by NeMoCLAW pods (manifest defaults to `app.kubernetes.io/name: nemoclaw`).

---

## Step 1: Review the policy manifest

Open the provided policy:

```bash
cat nvidia-nemoclaw-netapp-ontap-networkpolicy.yaml
```

What it does:
- Selects NeMoCLAW pods by label.
- Enforces **egress-only** restrictions.
- Allows DNS to `kube-dns` (TCP/UDP 53).
- Allows ONTAP access to the configured `ipBlock` on selected ports.

---

## Step 2: Confirm NeMoCLAW pod labels

Verify pod labels in your namespace:

```bash
kubectl get pods -n nvidia-nemoclaw --show-labels
```

If your pods do **not** use `app.kubernetes.io/name=nemoclaw`, update `spec.podSelector.matchLabels` in the manifest to match your real label set.

---

## Step 3: Customize namespace (if needed)

If your NeMoCLAW deployment is in a different namespace, edit:

```yaml
metadata:
  namespace: nvidia-nemoclaw
```

Set this to your actual namespace.

---

## Step 4: Customize ONTAP management network ranges

Edit the `ipBlock` section:

```yaml
- ipBlock:
    cidr: 10.20.30.0/24
    except:
      - 10.20.30.128/25
```

Replace:
- `cidr` with your real ONTAP management LIF subnet/range.
- `except` with any ranges that should be explicitly denied.

> If you only have a single ONTAP management IP, use `/32` (example: `192.0.2.25/32`).

---

## Step 5: Keep only required ports

The manifest includes a secure default plus optional ports:

- Required for most API workflows: `TCP 443`
- Optional legacy API: `TCP 80`
- Optional automation over CLI: `TCP 22`
- Optional SNMP monitoring: `UDP 161`, `UDP 162`

Remove any optional ports your environment does not require.

---

## Step 6: Validate manifest locally

Run a client-side schema validation:

```bash
kubectl apply --dry-run=client -f nvidia-nemoclaw-netapp-ontap-networkpolicy.yaml
```

---

## Step 7: Apply the policy

Deploy it to the cluster:

```bash
kubectl apply -f nvidia-nemoclaw-netapp-ontap-networkpolicy.yaml
```

Confirm creation:

```bash
kubectl get networkpolicy -n nvidia-nemoclaw
kubectl describe networkpolicy nemoclaw-netapp-ontap-access -n nvidia-nemoclaw
```

---

## Step 8: Verify NeMoCLAW connectivity to ONTAP

1. Check NeMoCLAW application logs for successful ONTAP API connections.
2. Run an in-pod connectivity test (if permitted):

```bash
kubectl exec -n nvidia-nemoclaw <nemoclaw-pod> -- sh -c 'nc -vz <ontap-mgmt-ip-or-fqdn> 443'
```

3. Confirm DNS resolution still works:

```bash
kubectl exec -n nvidia-nemoclaw <nemoclaw-pod> -- nslookup <ontap-mgmt-fqdn>
```

---

## Step 9: Troubleshoot (if traffic is blocked)

If NeMoCLAW cannot reach ONTAP after rollout:

1. Confirm the pod labels match the policy selector.
2. Confirm ONTAP target IP/FQDN resolves into allowed CIDR ranges.
3. Confirm required ports are present in the policy.
4. Check for additional NetworkPolicies in the namespace that may further restrict egress.
5. Check CNI/plugin support for NetworkPolicy egress enforcement.

---

## Step 10: Rollback procedure

If you need to revert quickly:

```bash
kubectl delete -f nvidia-nemoclaw-netapp-ontap-networkpolicy.yaml
```

Or delete by name:

```bash
kubectl delete networkpolicy nemoclaw-netapp-ontap-access -n nvidia-nemoclaw
```

---

## Operational recommendations

- Apply first in non-production and validate ONTAP workflows.
- Prefer least privilege: keep only `TCP 443` unless extra ports are proven necessary.
- Version-control policy changes and peer-review CIDR/port updates.
- Re-validate the policy whenever ONTAP management endpoints change.
