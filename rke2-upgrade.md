# RKE2 Version Upgrade Guide

This document provides a step-by-step guide on how to upgrade the Kubernetes version in RKE2 clusters. Upgrading your RKE2 cluster involves several critical steps that ensure the upgrade process is smooth and your cluster remains stable and compatible with the latest Kubernetes enhancements.

## Pre-upgrade Steps

### Check Rancher Version Compatibility

Before upgrading RKE2, ensure that the target RKE2 version is compatible with your Rancher version. Refer to the [Rancher Support Matrix](https://www.suse.com/suse-rancher/support-matrix/all-supported-versions) to verify compatibility.

This step is crucial to prevent any potential issues caused by version mismatches between Rancher and RKE2.

### Check for Deprecated API Versions

Before initiating the upgrade, it's crucial to identify any deprecated API versions that might affect your resources. Use `kubent` to scan your cluster for deprecated APIs:

- install kubent any linux jump server. [Documantation](https://github.com/doitintl/kube-no-trouble)

- Change active kubeconfig

- Run the `kubent` command:
```bash
kubent --target-version <target_kubernetes_version> 
```
Example:
```bash
kubent --target-version 1.30
```

### Pre-upgrade Application Status Check

Before starting the upgrade, take note of any applications that are not running properly in all namespaces within the cluster. Pay special attention to applications with frequent restarts or error states. This step is crucial for distinguishing issues caused by the upgrade from pre-existing issues.

To identify problematic applications, you can check the status of all pods and sort them by their status in the Rancher interface. Document these observations so you can accurately determine which applications might have been affected by the upgrade.

### Installation of System Upgrade Controller and CRDs

The system-upgrade-controller can be installed as a deployment into your cluster. The deployment requires a service-account, clusterRoleBinding, and a configmap. To install these components, run the following command:

```bash
kubectl apply -f https://github.com/rancher/system-upgrade-controller/releases/latest/download/system-upgrade-controller.yaml
kubectl apply -f https://github.com/rancher/system-upgrade-controller/releases/latest/download/crd.yaml
```

## Preparing Upgrade Plans

Create separate upgrade plans from verison 1.27 to verson 1.28 for your master and worker nodes. These plans will direct the System Upgrade Controller on how to proceed with the upgrade.
Visit for the latest installation instructions: [RKE2 Docs: Install the system-upgrade-controller](https://docs.rke2.io/upgrade/automated_upgrade#install-the-system-upgrade-controller)

**[Notes]**
- The plans must be created in the same namespace where the controller was deployed.
- The `concurrency` field indicates how many nodes can be upgraded at the same time.
- The server-plan targets server nodes by specifying a label selector that selects nodes with the node-role.kubernetes.io/control-plane label

###  Master Upgrade Plan (Nodes 1-by-1)
- The version you are upgrading to must be exactly 1 minor version ahead of your current version. For example, if your current version is v1.27.X, you can upgrade to v1.28.X
- Check the exact patch version to use for your target version from the [official release notes](https://docs.rke2.io/release-notes/v1.32.X)
- Make sure the specified patch version is stable and compatible with your cluster setup.
- Create a server-plan-for-<rke2-version>.yaml file with the following content:
### Notes on Version Formatting
- `<rke2-version>` refers to the target version you want to upgrade to, which can be found in the [RKE2 release notes](https://docs.rke2.io/release-notes/v1.32.X).
- `<rke2-version-label>` is the same version as `<rke2-version>`, but with the `+` character replaced by a `-` character. This modification is necessary because `+` is not accepted in labels or names.

#### Example:
- **Original version (`<rke2-version>`)**: `v1.28.14+rke2r1`
- **Modified version (`<rke2-version-label>`)**: `v1.28.14-rke2r1`

This adjustment ensures compatibility with Kubernetes naming conventions for labels and metadata.
```yaml
apiVersion: upgrade.cattle.io/v1
kind: Plan
metadata:
  name: server-plan-for-<rke2-version-label>
  namespace: system-upgrade
  labels:
    rke2-upgrade: server
spec:
  concurrency: 1 # for master nodes
  nodeSelector:
    matchExpressions:
      - {key: node-role.kubernetes.io/control-plane, operator: In, values: ["true"]}
      - {key: rke2-upgrade, operator: Exists}
      - {key: rke2-upgrade, operator: NotIn, values: ["disabled", "false"]}
      - {key: rke2-upgrade-to, operator: In, values: ["<rke2-version-label>"]}
  tolerations:
  - key: "CriticalAddonsOnly"
    operator: "Equal"
    value: "true"
    effect: "NoExecute"
  serviceAccountName: system-upgrade
  cordon: true
  drain:
    force: true
  upgrade:
    image: rancher/rke2-upgrade
  version: "<rke2-version>"
```

After the version upgrade of all master nodes is completed, you can move on to worker nodes.

###  Worker Upgrade Plan
Create an agent-plan-for-<rke2-version>.yaml file with the following content:

```yaml
apiVersion: upgrade.cattle.io/v1
kind: Plan
metadata:
  name: agent-plan-for-<rke2-version-label>
  namespace: system-upgrade
  labels:
    rke2-upgrade: agent
spec:
  concurrency: 1
  nodeSelector:
    matchExpressions:
      - {key: rke2-upgrade, operator: Exists}
      - {key: rke2-upgrade-to, operator: In, values: ["<rke2-version-label>"]}
  serviceAccountName: system-upgrade
  cordon: true
  drain:
    force: true
  upgrade:
    image: rancher/rke2-upgrade
  version: "<rke2-version>"
```

## Labelling Nodes

List all node names by their roles:  
- `Master Nodes`: Displays control-plane nodes.  
- `Worker Nodes`: Displays nodes without control-plane or master roles.
Reveal all node names
```bash
echo -e "Master Nodes:\n$(kubectl get nodes -l 'node-role.kubernetes.io/control-plane' -o custom-columns=NAME:.metadata.name --no-headers | tr '\n' ' ')\n" && \
echo -e "Worker Nodes:\n$(kubectl get nodes --selector='!node-role.kubernetes.io/control-plane,!node-role.kubernetes.io/master' -o custom-columns=NAME:.metadata.name --no-headers | tr '\n' ' ')"
```

### Labeling Master Nodes

**Note:** If you want to proceed cautiously, you can label the nodes one by one. Alternatively, if you label all master nodes at once, the upgrade will process them sequentially based on the concurrency value defined in the plan.

```bash
kubectl label nodes <master01> <master02> <master03>    rke2-upgrade=true rke2-upgrade-to=<rke2-version-label> --overwrite
```

### Labeling Worker Nodes

**Note:** Similar to the master nodes, you can label worker nodes individually for more control. If you label all worker nodes simultaneously, the upgrade will process them sequentially according to the concurrency value specified in the plan.

```bash
kubectl label nodes <worker01> <worker02> <worker03>    rke2-upgrade=true rke2-upgrade-to=<rke2-version-label> --overwrite
```

## Applying the Upgrade Plans

### Apply master nodes plan
```bash
kubectl apply -f server-plan-for-<rke2-version-label>
kubectl -n system-upgrade get plans,jobs
```
Watch the system upgrade controller and job logs.

### Apply worker nodes plan
```bash
kubectl apply -f agent-plan-for-<rke2-version-label>
kubectl -n system-upgrade get plans,jobs
```
Watch the system upgrade controller and job logs.

**Note:** When you apply the plan, a job will be created for each node. Each job will spawn a pod with **two containers**:

**Init container:** Responsible for draining the node.
**Main container:** Performs the upgrade process.
**Important:** If your cluster has a **Pod Disruption Budget (PDB)** in place, the init container may fail to complete the drain operation. Manual intervention might be required to temporarily adjust or remove the PDB for the affected pods.

## After Upgrade

### Additional Notes for Further Upgrades

- If you want to upgrade to another version, repeat the process starting from the Create Plans step.
- Prepare new server and worker plans for the target version.
- Apply the plans and label the nodes you want to upgrade.
- Monitor the logs of the upgrade jobs to ensure the process completes successfully.

### Final Validation

Once all steps are completed, review your cluster's workloads to ensure everything is functioning as expected. Compare the state of the workloads before and after the upgrade to identify any discrepancies or issues.

### Delete labels

After the upgrade, you no longer need the above labels.

```bash 
kubectl label nodes <master01> <master02> <master03> <worker01> <worker02> <worker03>  "<label_key_1>"-  "label_key_2"- "rke2-upgrade"- "rke2-upgrade-to<rke2-version-label>"-
```

### Remove Upgrade Controller

```bash
kubectl delete -f https://github.com/rancher/system-upgrade-controller/releases/latest/download/system-upgrade-controller.yaml
kubectl delete -f https://github.com/rancher/system-upgrade-controller/releases/latest/download/crd.yaml
```