# ☸️ CKA Practical Cheat Sheet — Commands, Troubleshooting & Exam Tips

---

## 📑 Table of Contents

1. [Setup — First 60 Seconds](#1-setup--first-60-seconds)
2. [Kubernetes Context](#2-kubernetes-context)
3. [Create Resources Fast](#3-create-resources-fast)
4. [RBAC](#4-rbac)
5. [Deployments](#5-deployments)
6. [Node Operations](#6-node-operations)
7. [etcd Backup and Restore](#7-etcd-backup-and-restore)
8. [kubeadm Upgrades](#8-kubeadm-upgrades)
9. [Storage — PV and PVC](#9-storage--pv-and-pvc)
10. [Services](#10-services)
11. [NetworkPolicies](#11-networkpolicies)
12. [Troubleshooting](#12-troubleshooting)
13. [Static Pods](#13-static-pods)
14. [DNS and Connectivity](#14-dns-and-connectivity)
15. [Quick Debug](#15-quick-debug)
16. [kubectl explain](#16-kubectl-explain)
17. [Time Savers](#17-time-savers)
18. [Patch Resources](#18-patch-resources)
19. [Flannel and Calico](#19-flannel-and-calico)
20. [IPv4 Packet Forwarding](#20-ipv4-packet-forwarding)
21. [Multi-container Pods](#21-multi-container-pods)
22. [Helm and CRDs](#22-helm-and-crds)
23. [Resource Requests, Limits & Troubleshooting](#23-resource-requests-limits--troubleshooting)
24. [Official Documentation](#24-official-documentation)
25. [Final CKA Tips](#25-final-cka-tips)

---

## 1. Setup — First 60 Seconds

Before starting Kubernetes tasks, configure useful terminal shortcuts.

### Essential aliases

```bash
alias k='kubectl'
alias kn='kubectl config set-context --current --namespace'

source <(kubectl completion bash)
complete -o default -F __start_kubectl k
```

Useful aliases:

- `k` — Shorter alternative to `kubectl`.
- `kn` — Quickly change the active namespace.
- Autocompletion — Reduce typing errors.

Example:

```bash
k run nginx --image=nginx --dry-run=client -o yaml > pod.yaml
```

**💡 CKA Tip:** Small terminal optimizations can save time across multiple tasks. Test your aliases before relying on them.

---

## 2. Kubernetes Context

Always check which cluster and namespace you are working in.

```bash
# List contexts
k config get-contexts

# Switch context
k config use-context <context-name>

# Verify context
k config current-context

# Change namespace
k config set-context --current --namespace=<namespace>

# Using the alias
kn <namespace>
```

### SSH to the correct server

If the task provides a server to connect to, follow that instruction before working on the cluster.

```bash
ssh <user>@<server>
```

After connecting, verify the environment and use the specified Kubernetes context.

**💡 CKA Tip:** Always check the server specified in the task instructions. Different tasks may require working on different servers. A technically correct solution in the wrong environment will not solve the intended task.

Documentation: [Configure Access to Multiple Clusters](https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/)

---

## 3. Create Resources Fast

Use imperative commands to generate manifests instead of writing every YAML file manually.

### Pod

```bash
k run pod1 --image=nginx:1.27 --dry-run=client -o yaml > pod.yaml
```

### Deployment

```bash
k create deployment dep1 \
  --image=nginx:1.27 \
  --replicas=3 \
  --dry-run=client -o yaml > deployment.yaml
```

### Service

```bash
k expose deployment dep1 \
  --port=80 \
  --target-port=80 \
  --dry-run=client -o yaml > service.yaml
```

### Job

```bash
k create job job1 \
  --image=busybox:1.36 \
  --dry-run=client -o yaml -- sh -c "echo done" > job.yaml
```

### CronJob

```bash
k create cronjob cron1 \
  --image=busybox:1.36 \
  --schedule="*/5 * * * *" \
  --dry-run=client -o yaml -- date > cronjob.yaml
```

### ConfigMap

```bash
k create cm my-cm --from-literal=KEY=value
```

### Secret

```bash
k create secret generic my-sec \
  --from-literal=PASS=example
```

### ServiceAccount

```bash
k create sa my-sa -n <namespace>
```

**💡 CKA Tip:** Generate YAML, modify the required fields, apply it, and verify the result.

```bash
k apply -f deployment.yaml
k get deployments
k get pods -o wide
```

---

## 4. RBAC

Role-Based Access Control determines which actions users and ServiceAccounts can perform.

### Create Role

```bash
k create role pod-reader \
  --verb=get,list,watch \
  --resource=pods \
  -n development
```

### Create RoleBinding

```bash
k create rolebinding read-pods \
  --role=pod-reader \
  --serviceaccount=development:app-sa \
  -n development
```

### Create ClusterRole

```bash
k create clusterrole node-reader \
  --verb=get,list \
  --resource=nodes
```

### Create ClusterRoleBinding

```bash
k create clusterrolebinding read-nodes \
  --clusterrole=node-reader \
  --serviceaccount=development:app-sa
```

### Verify permissions

```bash
k auth can-i get pods \
  -n development \
  --as=system:serviceaccount:development:app-sa
```

**Remember:**

- `Role` — Namespace-scoped permissions.
- `ClusterRole` — Can define cluster-scoped or namespaced permissions.
- `RoleBinding` — Grants permissions within a namespace.
- `ClusterRoleBinding` — Grants permissions cluster-wide.

Documentation: [Kubernetes RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)

---

## 5. Deployments

### Rolling updates

```bash
k set image deployment/<name> \
  <container>=<image>
```

### Monitor rollout

```bash
k rollout status deployment/<name>
```

### View history

```bash
k rollout history deployment/<name>
```

### Rollback

```bash
k rollout undo deployment/<name>
```

### Rollback to a revision

```bash
k rollout undo deployment/<name> --to-revision=2
```

### Scale

```bash
k scale deployment/<name> --replicas=5
```

### Restart

```bash
k rollout restart deployment/<name>
```

### Verify

```bash
k get deployment <name>
k describe deployment <name>
k get pods -o wide
```

**💡 CKA Tip:** Don't trust a successful command alone. Check rollout status and Pod readiness.

---

## 6. Node Operations

### Cordon a node

```bash
k cordon <node>
```

Prevents new Pods from being scheduled on the node.

### Drain a node

```bash
k drain <node> \
  --ignore-daemonsets \
  --delete-emptydir-data
```

Evicts eligible workloads before maintenance. Be aware that PodDisruptionBudgets can block eviction.

### Uncordon

```bash
k uncordon <node>
```

### Add a taint

```bash
k taint nodes <node> key=value:NoSchedule
```

### Remove a taint

```bash
k taint nodes <node> key=value:NoSchedule-
```

### Label a node

```bash
k label nodes <node> disk=ssd
```

### Inspect node health

```bash
k get nodes -o wide
k describe node <node>
```

Documentation: [Safely Drain a Node](https://kubernetes.io/docs/tasks/administer-cluster/safely-drain-node/)

---

## 7. etcd Backup and Restore

etcd stores Kubernetes cluster state.

These examples assume a kubeadm cluster with local etcd.

### Inspect etcd configuration

```bash
cat /etc/kubernetes/manifests/etcd.yaml
```

Look for certificate paths, endpoints, and the data directory.

### Backup

```bash
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
  --key=/etc/kubernetes/pki/etcd/healthcheck-client.key \
  snapshot save /tmp/backup.db
```

### Verify snapshot

```bash
etcdutl snapshot status /tmp/backup.db \
  --write-out=table
```

### Restore snapshot

```bash
etcdutl snapshot restore /tmp/backup.db \
  --data-dir=/var/lib/etcd-restored
```

The restore command creates a restored data directory. It does not automatically switch the running cluster to that directory.

For a kubeadm static Pod setup, the etcd manifest must be configured to use the restored data appropriately. Follow the appropriate recovery procedure for your cluster topology.

**💡 CKA Tip:** Learn where to find certificate paths in the etcd manifest rather than memorizing every path.

Documentation: [Operating etcd Clusters](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/)

---

## 8. kubeadm Upgrades

Always use the Kubernetes version specified in the task.

### Check current versions

```bash
kubectl get nodes
kubeadm version
kubelet --version
```

### Upgrade kubeadm

For Debian/Ubuntu systems:

```bash
sudo apt-mark unhold kubeadm
sudo apt-get update
sudo apt-get install -y kubeadm=<package-version>
sudo apt-mark hold kubeadm
```

### Check upgrade plan

```bash
sudo kubeadm upgrade plan
```

### Upgrade first control plane node

```bash
sudo kubeadm upgrade apply <target-version>
```

### Upgrade kubelet and kubectl

```bash
sudo apt-mark unhold kubelet kubectl

sudo apt-get install -y \
  kubelet=<package-version> \
  kubectl=<package-version>

sudo apt-mark hold kubelet kubectl

sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

### Worker nodes

For worker nodes and additional control plane nodes, the kubeadm step is:

```bash
sudo kubeadm upgrade node
```

Follow the appropriate drain, package upgrade, restart, and uncordon sequence for each node.

**💡 CKA Tip:** Remember the difference between `kubeadm upgrade apply` and `kubeadm upgrade node`.

Documentation: [Upgrading kubeadm Clusters](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/)

---

## 9. Storage — PV and PVC

Important concepts:

- PersistentVolume (PV) — Cluster-scoped resource.
- PersistentVolumeClaim (PVC) — Namespace-scoped resource.
- `storageClassName` — Storage class selection.
- `accessModes` — How storage can be accessed.
- `volumeName` — Bind a PVC to a specific PV.

### Access modes

| Mode | Meaning |
|---|---|
| RWO | ReadWriteOnce — read/write by one node |
| ROX | ReadOnlyMany — read-only by many nodes |
| RWX | ReadWriteMany — read/write by many nodes |
| RWOP | ReadWriteOncePod — read/write by one Pod |

### Important: PV and PVC Binding Requirements

For a PVC to bind to a PV, the following parameters must be compatible:

1. **Storage capacity:** The PV capacity must be equal to or greater than the PVC's requested storage.
2. **Access modes:** The PV must support the access modes requested by the PVC.
3. **Storage class:** The `storageClassName` must match. An empty string (`""`) explicitly requests a PV without a storage class.
4. **Volume mode:** Both must have compatible `volumeMode` settings (`Filesystem` or `Block`).
5. **Volume availability:** The PV must be available or appropriately reserved for the claim.

**💡 CKA Tip:** Setting `volumeName` alone is not enough. Even when you specify a particular PV, Kubernetes still checks the binding requirements. If the PV and PVC parameters don't match, the PVC can remain `Pending`.

### Example PersistentVolume

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: ""
  hostPath:
    path: /mnt/data
```

### PVC binding to a specific PV

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: ""
  volumeName: my-pv
```

In this example:

- The PV provides `2Gi`, while the PVC requests `1Gi` — compatible.
- Both use `ReadWriteOnce` — compatible.
- Both use `storageClassName: ""` — compatible.
- The PVC specifies `volumeName: my-pv` — requests the intended PV.

The `hostPath` PV is for single-node learning environments, not general-purpose production storage.

### Verify

```bash
k get pv
k get pvc
k describe pvc my-pvc
```

Expected status after successful binding:

```text
NAME     STATUS   VOLUME
my-pvc   Bound    my-pv
```

### Mount PVC into a Pod

```yaml
volumes:
  - name: data
    persistentVolumeClaim:
      claimName: my-pvc
```

Container:

```yaml
volumeMounts:
  - name: data
    mountPath: /data
```

### Reclaim policies

- `Retain` — Preserve the underlying storage asset for manual recovery.
- `Delete` — Delete the storage asset according to the storage provisioner's behavior.

**💡 CKA Tip:** If a PVC remains Pending, check `storageClassName`, `accessModes`, capacity, `volumeMode`, `volumeName`, and events.

Documentation: [Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)

---

## 10. Services

| Service Type | Purpose |
|---|---|
| ClusterIP | Internal cluster access |
| NodePort | Exposes a port on nodes |
| LoadBalancer | Uses an external load balancer where supported |

### Create a NodePort Service

```bash
k expose deployment web \
  --port=80 \
  --target-port=8080 \
  --type=NodePort
```

Remember:

- `port` — Service port.
- `targetPort` — Backend Pod port.
- `nodePort` — Port exposed on nodes.

The default NodePort range is `30000-32767`, although clusters can configure a different range.

### Check Service endpoints

```bash
k get svc
k get endpoints
k get endpointslices
```

**💡 CKA Tip:** If a Service has no ready backends, check its selector and the readiness of the matching Pods.

---

## 11. NetworkPolicies

NetworkPolicies control allowed traffic to and from selected Pods when supported by the network plugin.

### Default deny ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
spec:
  podSelector: {}
  policyTypes:
    - Ingress
```

### DNS egress

When an egress policy isolates Pods, you may need to explicitly allow DNS.

Example permitting DNS traffic to any destination:

```yaml
egress:
  - ports:
      - protocol: UDP
        port: 53
      - protocol: TCP
        port: 53
```

This is a broad rule. In production, prefer restricting DNS access to the appropriate DNS Pods or addresses.

### AND vs OR

- `namespaceSelector` and `podSelector` within the same peer item are combined with AND.
- Separate peer items are alternatives (OR).

### Inspect policies

```bash
k get netpol -A
k describe netpol <name> -n <namespace>
```

**💡 CKA Tip:** A NetworkPolicy only works if your CNI supports enforcement. Calico supports Kubernetes NetworkPolicies; standard Flannel installations do not enforce them by themselves.

Documentation: [Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)

---

## 12. Troubleshooting

This is one of the most important areas to practice.

### Step 1 — Check node status

```bash
k get nodes -o wide
```

If the API server is unavailable, continue directly on the affected node.

### Step 2 — Check kubelet

```bash
sudo systemctl status kubelet
```

If kubelet is stopped, investigate the cause and start it when appropriate:

```bash
sudo systemctl start kubelet
```

Enable startup at boot:

```bash
sudo systemctl enable kubelet
```

### Step 3 — Check kubelet logs

```bash
sudo journalctl -u kubelet -n 30 --no-pager
```

For live logs:

```bash
sudo journalctl -u kubelet -f
```

Look for certificate errors, configuration issues, container runtime failures, or missing files.

### Step 4 — Check API server readiness

```bash
curl -k 'https://127.0.0.1:6443/readyz?verbose'
```

This checks API server readiness, including etcd-related readiness checks.

For detailed connection information:

```bash
curl -kv https://127.0.0.1:6443/readyz
```

These commands assume you're on the control plane node and the API server listens locally on port 6443.

`-k` disables TLS certificate verification for debugging. Keep certificate verification enabled for normal secure access.

**Important:** `/readyz` does not check the health of every Kubernetes control plane component.

### Step 5 — When kubectl does not work

Use the container runtime interface:

```bash
sudo crictl ps -a
```

Inspect container logs:

```bash
sudo crictl logs <container-id>
```

Check containerd:

```bash
sudo systemctl status containerd
```

If needed, configure `crictl` to use the correct runtime endpoint.

### Step 6 — Inspect control plane manifests

```bash
ls /etc/kubernetes/manifests/
```

Typical kubeadm static Pod manifests:

```text
etcd.yaml
kube-apiserver.yaml
kube-controller-manager.yaml
kube-scheduler.yaml
```

Inspect the component you're troubleshooting:

```bash
sudo cat /etc/kubernetes/manifests/kube-apiserver.yaml
```

Check configuration, certificate paths, and startup arguments.

### Step 7 — Troubleshoot application Pods

```bash
k get pods -A -o wide
k describe pod <pod> -n <namespace>
k logs <pod> -n <namespace>
k logs <pod> -n <namespace> --previous
```

### Step 8 — Check events

```bash
k get events -A \
  --sort-by=.metadata.creationTimestamp
```

### Step 9 — Check networking

```bash
k get svc -A
k get endpointslices -A
k get netpol -A
```

**💡 CKA Tip:** Troubleshooting should follow evidence. If the API server is down, use node-level tools. If only one Pod is failing, start with Pod events, configuration, and logs.

---

## 13. Static Pods

Static Pods are managed directly by kubelet.

### Find the manifest directory

```bash
grep staticPodPath /var/lib/kubelet/config.yaml
```

For typical kubeadm clusters:

```text
/etc/kubernetes/manifests/
```

### Inspect static Pod manifests

```bash
ls /etc/kubernetes/manifests/
```

### Troubleshoot static Pods

```bash
sudo crictl ps -a
sudo journalctl -u kubelet -n 50
```

**💡 CKA Tip:** Editing a static Pod manifest causes kubelet to reconcile the Pod. Be careful: incorrect configuration can disrupt control plane components.

Documentation: [Create Static Pods](https://kubernetes.io/docs/tasks/configure-pod-container/static-pod/)

---

## 14. DNS and Connectivity

### Service DNS format

```text
<service>.<namespace>.svc.cluster.local
```

The suffix depends on the configured cluster domain.

### Test DNS

```bash
kubectl run dnstest \
  --image=busybox:1.36 \
  --rm -it \
  --restart=Never \
  -- nslookup kubernetes.default.svc.cluster.local
```

### Test a specific Service

```bash
kubectl run dnstest \
  --image=busybox:1.36 \
  --rm -it \
  --restart=Never \
  -- nslookup my-service.my-namespace.svc.cluster.local
```

### Test HTTP connectivity

```bash
kubectl run curltest \
  --image=curlimages/curl \
  --rm -it \
  --restart=Never \
  -- curl -v http://my-service.my-namespace.svc.cluster.local:80
```

### Check CoreDNS

```bash
k get pods -n kube-system -l k8s-app=kube-dns
k get svc kube-dns -n kube-system
k get endpointslice -n kube-system
```

**💡 CKA Tip:** DNS resolution and application connectivity are different. A Service can resolve successfully while its backends are unavailable.

Documentation: [Debugging DNS Resolution](https://kubernetes.io/docs/tasks/administer-cluster/dns-debugging-resolution/)

---

## 15. Quick Debug

Useful commands when investigating Kubernetes problems.

```bash
# All Pods and their nodes
k get pods -A -o wide

# Node resource usage
k top nodes

# Pod resource usage
k top pods -A

# Recent cluster events
k get events -A \
  --sort-by=.metadata.creationTimestamp

# Node conditions
k describe node <node> | grep -A5 Conditions

# Allocated resources
k describe node <node> | grep -A10 "Allocated resources"
```

### Search directly within long output

```bash
k -n project-c13 describe pod <pod-name> \
  | less -p Requests
```

This opens the Pod description and jumps to the first occurrence of `Requests`.

Other examples:

```bash
k describe node <node> | less -p "Allocated resources"

k describe pod <pod> | less -p Events
```

**💡 CKA Tip:** `less -p` is useful when you need to locate specific information quickly.

---

## 16. kubectl explain

Use `kubectl explain` when you don't remember a YAML field.

```bash
k explain pod

k explain pod.spec

k explain pod.spec.containers

k explain pod.spec.containers.env

k explain deployment.spec.strategy

k explain --recursive pod | less
```

### Useful examples

```bash
k explain rolebinding.roleRef

k explain deployment.spec.strategy.rollingUpdate

k explain pvc.spec.resources

k explain pod.spec.containers.livenessProbe
```

**💡 CKA Tip:** `kubectl explain` helps you inspect the Kubernetes API schema without opening a browser.

---

## 17. Time Savers

### View resource YAML

```bash
k get deployment <name> -o yaml
```

### Edit a resource

```bash
k edit deployment <name>
```

**💡 CKA Tip:** Learn the commands you use frequently until they become natural.

---

## 18. Patch Resources

Patching is useful when you only need to change a small part of an existing resource.

### Patch from file

```bash
kubectl patch deployment patch-demo \
  --patch-file patch-file.json
```

### Patch a Deployment image

```bash
kubectl patch deployment patch-demo \
  --type=strategic \
  -p '{"spec":{"template":{"spec":{"containers":[{"name":"app","image":"nginx:1.28"}]}}}}'
```

Replace `app` with the actual container name.

For image updates, this is usually simpler:

```bash
k set image deployment/patch-demo app=nginx:1.28
```

### PriorityClass example

Create a PriorityClass:

```bash
k create priorityclass high-priority \
  --value=1000 \
  --description="High priority workload"
```

Patch a Deployment:

```bash
k patch deployment busybox-logger \
  -n priority \
  -p '{"spec":{"template":{"spec":{"priorityClassName":"high-priority"}}}}'
```

Verify:

```bash
k get deployment busybox-logger \
  -n priority \
  -o jsonpath='{.spec.template.spec.priorityClassName}{"\n"}'
```

**💡 CKA Tip:** Patching can be faster than rewriting a full manifest, but always verify the resulting configuration.

Documentation: [Update API Objects Using kubectl patch](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/update-api-object-kubectl-patch/)

---

## 19. Flannel and Calico

### Flannel — Check Pod CIDR before installation

One useful lesson from Kubernetes networking practice is to check the Pod CIDR before installing a CNI.

### Check kubeadm configuration

```bash
kubectl -n kube-system get cm kubeadm-config -o yaml
```

Look for:

```yaml
networking:
  podSubnet: 10.244.0.0/16
```

Alternatively, on a kubeadm control plane node:

```bash
sudo grep -E 'cluster-cidr|allocate-node-cidrs' \
  /etc/kubernetes/manifests/kube-controller-manager.yaml
```

You may find:

```text
--cluster-cidr=10.244.0.0/16
```

### Download the Flannel manifest

Instead of immediately applying a remote manifest, download and inspect it.

```bash
curl -L \
  https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml \
  -o kube-flannel.yml
```

### Inspect the network configuration

```bash
grep -A8 '"net-conf.json"' kube-flannel.yml
```

A typical configuration contains:

```json
{
  "Network": "10.244.0.0/16",
  "Backend": {
    "Type": "vxlan"
  }
}
```

The exact fields depend on the Flannel version.

If your cluster uses a different Pod CIDR, update the manifest's network configuration to match.

```bash
vim kube-flannel.yml
```

Then apply:

```bash
kubectl apply -f kube-flannel.yml
```

### Verify

```bash
kubectl get pods -A -o wide
kubectl get nodes
```

**💡 CKA Tip:** Don't blindly apply a CNI manifest. Check the cluster Pod CIDR, CNI configuration, and node networking requirements first.

Flannel: [Official GitHub Repository](https://github.com/flannel-io/flannel)

### Calico

Calico provides Pod networking and Kubernetes NetworkPolicy enforcement.

Useful checks:

```bash
kubectl get pods -A | grep -i calico

kubectl get nodes -o wide

kubectl describe node <node>
```

If installed using the Tigera operator:

```bash
kubectl get installation.operator.tigera.io -A
```

Depending on the installation method, inspect the Calico IP pool configuration and confirm it matches your intended Pod networking.

**💡 CKA Tip:** Calico installation differs between operator-based and manifest-based deployments. Use the correct documentation for your setup.

Calico: [Official Documentation](https://docs.tigera.io/calico/latest/about/)

---

## 20. IPv4 Packet Forwarding

One of the official Kubernetes documentation sections that helped me with networking configuration was the container runtime setup guide.

### Check IPv4 forwarding

```bash
sysctl net.ipv4.ip_forward
```

Expected:

```text
net.ipv4.ip_forward = 1
```

### Enable forwarding persistently

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
EOF
```

Apply:

```bash
sudo sysctl --system
```

Verify:

```bash
sysctl net.ipv4.ip_forward
```

**💡 CKA Tip:** If Pod networking is broken, IPv4 forwarding is one useful setting to verify. Also check the CNI, routing, firewall rules, and Pod CIDR.

Documentation: [Kubernetes Container Runtimes](https://kubernetes.io/docs/setup/production-environment/container-runtimes/)

---

## 21. Multi-container Pods

A useful Kubernetes exercise is creating two containers that share a volume.

One container writes timestamps to a file, while another reads the file continuously.

### Complete example

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: shared-logs
spec:
  containers:
    - name: writer
      image: busybox:1.36
      command:
        - sh
        - -c
        - "while true; do date >> /var/log/shared/dates.log; sleep 1; done"
      volumeMounts:
        - name: shared-data
          mountPath: /var/log/shared

    - name: reader
      image: busybox:1.36
      command:
        - sh
        - -c
        - "while true; do cat /var/log/shared/dates.log 2>/dev/null; sleep 2; done"
      volumeMounts:
        - name: shared-data
          mountPath: /var/log/shared

  volumes:
    - name: shared-data
      emptyDir: {}
```

### Apply

```bash
kubectl apply -f shared-logs.yaml
```

### Verify

```bash
kubectl get pod shared-logs

kubectl logs shared-logs -c reader
```

**💡 CKA Tip:** Understand the relationship between `command`, `args`, `volumeMounts`, and `volumes`.

Documentation: [Communicate Between Containers in the Same Pod](https://kubernetes.io/docs/tasks/access-application-cluster/communicate-containers-same-pod-shared-volume/)

---

## 22. Helm and CRDs

Helm is useful for deploying and managing Kubernetes applications.

### Install a chart

```bash
helm install my-release ./my-chart
```

### List releases

```bash
helm list -A
```

### Upgrade a release

```bash
helm upgrade my-release ./my-chart
```

### Inspect chart values

```bash
helm show values ./my-chart
```

### Check installed CRDs

```bash
kubectl get crds
```

### Important: When CRDs Are Already Installed

Sometimes, during CKA practice or exam tasks, the cluster already contains the required **CustomResourceDefinitions (CRDs)**.

If a Helm chart also tries to install those CRDs, you may need to disable their installation, depending on how the chart manages them.

There are two approaches.

#### Option 1 — Disable CRD installation through chart values

Some charts expose a setting such as `crds.enabled`.

```bash
helm install my-release ./my-chart \
  --set crds.enabled=false
```

Other charts may use:

```bash
helm install my-release ./my-chart \
  --set crds.install=false
```

**Important:** These are chart-specific values. They only work if the chart supports them.

Check first:

```bash
helm show values ./my-chart
```

Look for the relevant `crds` configuration.

#### Option 2 — Skip CRDs managed by Helm

```bash
helm install my-release ./my-chart --skip-crds
```

This tells Helm not to install CRDs located in the chart's `crds/` directory.

For an upgrade:

```bash
helm upgrade my-release ./my-chart --skip-crds
```

### Why is this useful?

Imagine a task where:

- The cluster already has the required CRDs installed.
- You need to install a Helm chart.
- The chart includes CRDs or exposes a setting to install them.
- The task requires you to reuse the existing CRDs rather than reinstall them.

In this situation, check how the chart handles CRDs and choose the appropriate option.

### Verify

```bash
kubectl get crds

helm show crds ./my-chart

helm show values ./my-chart
```

**💡 CKA Tip:** Remember both `--skip-crds` and chart-specific settings such as `--set crds.enabled=false` or `--set crds.install=false`.

They are not interchangeable:

- `--skip-crds` skips CRDs in Helm's special `crds/` directory.
- `--set crds.enabled=false` works only when the chart defines and uses that value.
- `--set crds.install=false` is another chart-specific possibility.
- None of these options automatically fixes every existing-resource ownership conflict.

**Always inspect the chart values and CRD configuration before choosing an option.**

Documentation: [Helm Install](https://helm.sh/docs/helm/helm_install/)

---

## 23. Resource Requests, Limits & Troubleshooting

One important Kubernetes troubleshooting topic is understanding **CPU and memory requests, limits, and node capacity**.

A Deployment may not become fully ready because its Pods cannot be scheduled, containers are repeatedly restarting, or resource limits are too restrictive.

### Understanding Requests vs Limits

| Parameter | Purpose |
|---|---|
| CPU Request | CPU reserved for scheduling decisions |
| Memory Request | Memory reserved for scheduling decisions |
| CPU Limit | Maximum CPU usage allowed before throttling |
| Memory Limit | Maximum memory usage before the container may be OOMKilled |

Example:

```yaml
resources:
  requests:
    memory: "128Mi"
    cpu: "100m"
  limits:
    memory: "256Mi"
    cpu: "500m"
```

**Important:**

- `100m` CPU = 0.1 CPU core.
- `500m` CPU = 0.5 CPU core.
- `128Mi` = 128 mebibytes of memory.
- The scheduler uses **requests**, not current CPU/memory usage, when determining whether a Pod fits on a node.
- Memory limits can cause OOM kills; CPU limits generally cause throttling rather than termination.

### 1. OOMKilled — Why Does It Happen?

`OOMKilled` means a process was terminated because of an out-of-memory condition.

A common cause is a container exceeding its configured memory limit.

Example:

```text
NAME                     READY   STATUS             RESTARTS
backend-7d4c8f           0/1     CrashLoopBackOff   5
```

The container's previous termination may show:

```text
Last State: Terminated
Reason: OOMKilled
Exit Code: 137
```

#### Troubleshoot OOMKilled

**Step 1 — Describe the Pod**

```bash
k describe pod <pod-name> -n <namespace>
```

Look for:

```text
Last State:     Terminated
Reason:         OOMKilled
Exit Code:      137
```

**Step 2 — Check container resource limits**

```bash
k get pod <pod-name> -n <namespace> -o yaml
```

Or:

```bash
k describe pod <pod-name> -n <namespace> \
  | less -p Limits
```

**Step 3 — Check current memory usage**

```bash
k top pod <pod-name> -n <namespace> --containers
```

This requires metrics to be available and shows current usage, not necessarily usage at the time of the crash.

**Step 4 — Check previous container logs**

```bash
k logs <pod-name> -n <namespace> --previous
```

For multiple containers:

```bash
k logs <pod-name> -n <namespace> \
  -c <container-name> --previous
```

**Step 5 — Fix the underlying problem**

If the application legitimately needs more memory and the node has enough capacity, adjust the memory limit.

For example:

```yaml
resources:
  requests:
    memory: "256Mi"
    cpu: "100m"
  limits:
    memory: "512Mi"
    cpu: "500m"
```

Apply the updated Deployment manifest:

```bash
k apply -f deployment.yaml
```

Verify:

```bash
k rollout status deployment/<deployment-name> \
  -n <namespace>
```

**💡 CKA Tip:** Don't automatically increase memory limits whenever you see `OOMKilled`. Check for memory leaks, excessive application usage, and node-level memory pressure first.

---

### 2. Deployment Not Fully Ready

Sometimes you may see:

```text
NAME       READY   UP-TO-DATE   AVAILABLE
backend    1/3     3            1
```

This means only one of the three desired replicas is currently ready.

#### Troubleshooting workflow

**Step 1 — Check the Deployment**

```bash
k get deployment <deployment-name> -n <namespace>
```

**Step 2 — Describe the Deployment**

```bash
k describe deployment <deployment-name> \
  -n <namespace>
```

Look for conditions and events such as:

```text
Available=False
MinimumReplicasUnavailable
```

These indicate insufficient availability, but do not identify the root cause by themselves.

**Step 3 — Check the Pods**

```bash
k get pods -n <namespace> -o wide
```

Look for:

- `Pending`
- `CrashLoopBackOff`
- `OOMKilled` in the previous container state
- `ImagePullBackOff`
- `Running` but `0/1` Ready

**Step 4 — Describe the affected Pod**

```bash
k describe pod <pod-name> -n <namespace>
```

Check the **Events** section.

```bash
k describe pod <pod-name> -n <namespace> \
  | less -p Events
```

The events often explain why a Pod is not ready or cannot be scheduled.

---

### 3. Pending Pods — Insufficient CPU or Memory

A Pod may remain `Pending` because the scheduler cannot find a node with enough allocatable resources.

Example event:

```text
Warning  FailedScheduling

0/2 nodes are available:
2 Insufficient cpu.
```

Or:

```text
0/2 nodes are available:
2 Insufficient memory.
```

This means the requested resources cannot fit on eligible nodes according to scheduler calculations.

#### Step 1 — Describe the Pending Pod

```bash
k describe pod <pod-name> -n <namespace>
```

Look at:

```text
Requests:
  cpu:      2000m
  memory:   2Gi
```

#### Step 2 — Check node resources

```bash
k describe node <node-name>
```

Or jump directly to allocated resources:

```bash
k describe node <node-name> \
  | less -p "Allocated resources"
```

Example:

```text
Allocated resources:
  Resource     Requests       Limits
  cpu          1800m (90%)    3000m (150%)
  memory       3500Mi (85%)   5000Mi (122%)
```

**Important:** These values represent resource requests and limits configured on Pods, not actual resource consumption.

Limits can total more than 100% because Kubernetes permits resource overcommitment.

#### Step 3 — Check node capacity and allocatable resources

```bash
k get nodes
```

```bash
k describe node <node-name> \
  | less -p Allocatable
```

Or:

```bash
k get node <node-name> \
  -o jsonpath='{.status.allocatable}'
```

**Capacity** represents the node's resources.

**Allocatable** represents resources available for Pods after Kubernetes and system reservations.

#### Step 4 — Check actual usage

```bash
k top nodes
```

```bash
k top pods -A
```

Remember that `kubectl top` displays current usage, while the scheduler primarily evaluates requested resources.

#### Step 5 — Identify the correct solution

Depending on the task, possible solutions include:

- Correcting unnecessarily high CPU or memory requests.
- Freeing capacity by removing workloads that are no longer required.
- Scheduling Pods on another eligible node.
- Adding node capacity when permitted.
- Checking node selectors, affinity, taints, and tolerations.

**💡 CKA Tip:** A node can show low actual CPU usage but still reject a Pod because existing resource requests have already consumed its allocatable CPU.

---

### 4. CrashLoopBackOff — Container Keeps Restarting

`CrashLoopBackOff` means Kubernetes is delaying repeated restart attempts after a container exits.

Possible causes include:

- Application crashes.
- Incorrect startup command or arguments.
- Missing configuration.
- Failed dependency connections.
- Memory exhaustion.
- Failing liveness probes.

#### Troubleshoot

```bash
k describe pod <pod-name> -n <namespace>
```

```bash
k logs <pod-name> -n <namespace> --previous
```

```bash
k get pod <pod-name> -n <namespace> -o yaml
```

Check the termination reason, exit code, application logs, container command, environment variables, and probes.

**💡 CKA Tip:** `CrashLoopBackOff` is a symptom, not a root cause. Look at the previous container state and logs.

---

### 5. Pod Running but Not Ready

A Pod may be `Running` while the Deployment remains unavailable.

Example:

```text
NAME           READY   STATUS
backend-pod    0/1     Running
```

This may indicate:

- Readiness probe failure.
- Application not listening on the expected port.
- Incorrect probe path.
- Application still initializing.
- Missing dependency.

#### Check readiness probes

```bash
k describe pod <pod-name> -n <namespace>
```

```bash
k get pod <pod-name> -n <namespace> -o yaml
```

Look for:

```yaml
readinessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 10
```

If the application doesn't serve `/health` on port `8080`, the readiness probe will fail.

Correct the probe to match the application's actual behavior rather than removing it without investigation.

---

### 6. Node MemoryPressure and DiskPressure

Sometimes the issue is not the container's memory limit but the node itself.

#### Check node conditions

```bash
k describe node <node-name> \
  | less -p Conditions
```

Look for:

```text
MemoryPressure
DiskPressure
PIDPressure
Ready
```

These conditions can affect Pod scheduling and availability.

#### Check node usage

```bash
k top node <node-name>
```

If you have node access, investigate memory and disk usage:

```bash
free -h
df -h
```

Check kubelet logs:

```bash
sudo journalctl -u kubelet -n 50 --no-pager
```

**Important:** Node-level OOM events and container memory-limit OOM events are not identical. Investigate the actual termination reason and node conditions.

---

### 7. Quick Resource Troubleshooting Cheat Sheet

| Symptom | First thing to check |
|---|---|
| `OOMKilled` | `kubectl describe pod` — Last State, memory limit |
| `CrashLoopBackOff` | `kubectl logs --previous` |
| `Pending` | `kubectl describe pod` — FailedScheduling events |
| `Insufficient cpu` | `kubectl describe node` — Allocated resources |
| `Insufficient memory` | Node allocatable memory and Pod requests |
| Deployment `1/3` ready | Deployment conditions and individual Pods |
| Pod `Running` but `0/1` | Readiness probe and application logs |
| `MemoryPressure` | Node conditions, kubelet logs, memory usage |
| `DiskPressure` | Node conditions and disk usage |

### My Recommended Troubleshooting Sequence

```bash
# 1. Check Deployment
k get deploy -n <namespace>

# 2. Check Pods
k get pods -n <namespace> -o wide

# 3. Describe Deployment
k describe deploy <deployment-name> -n <namespace>

# 4. Describe affected Pod
k describe pod <pod-name> -n <namespace>

# 5. Check logs
k logs <pod-name> -n <namespace> --previous

# 6. Inspect node resources
k describe node <node-name> \
  | less -p "Allocated resources"

# 7. Check current usage
k top nodes
k top pods -A

# 8. Verify after fixing
k rollout status deployment/<deployment-name> \
  -n <namespace>
```

**💡 CKA Tip:** If a Deployment is not fully ready, don't immediately modify the Deployment. First determine whether the problem is **scheduling, resource limits, application crashes, or readiness probes**.

Documentation: [Resource Management for Pods and Containers](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)

---

## 24. Official Documentation

These are useful references to bookmark during preparation.

| Topic | Official Documentation |
|---|---|
| Kubernetes Basics | [Concepts](https://kubernetes.io/docs/concepts/) |
| Container Runtimes | [Container Runtimes](https://kubernetes.io/docs/setup/production-environment/container-runtimes/) |
| kubeadm Installation | [Creating a Cluster](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/) |
| kubeadm Upgrades | [Upgrade Guide](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/) |
| etcd | [Operating etcd](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/) |
| RBAC | [RBAC Authorization](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) |
| NetworkPolicy | [Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/) |
| Storage | [Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/) |
| DNS | [DNS Debugging](https://kubernetes.io/docs/tasks/administer-cluster/dns-debugging-resolution/) |
| Static Pods | [Static Pod Guide](https://kubernetes.io/docs/tasks/configure-pod-container/static-pod/) |
| Patching | [kubectl patch](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/update-api-object-kubectl-patch/) |
| Flannel | [Flannel GitHub](https://github.com/flannel-io/flannel) |
| Calico | [Calico Documentation](https://docs.tigera.io/calico/latest/about/) |
| Helm | [Helm Documentation](https://helm.sh/docs/) |
| Resource Management | [Requests and Limits](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/) |
| Application Troubleshooting | [Debug Applications](https://kubernetes.io/docs/tasks/debug/debug-application/) |
| Node-pressure Eviction | [Node-pressure Eviction](https://kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/) |

**💡 CKA Tip:** Familiarize yourself with the official documentation before the exam. Follow the current exam rules for permitted websites and resources.

---

## 25. Final CKA Tips

Here are the habits I recommend developing during preparation.

### Before starting a task

- Verify the required SSH server.
- Verify the cluster context.
- Verify the namespace.
- Read the task requirements carefully.
- Identify the resources that need to be created or modified.

### While working

- Use imperative commands when appropriate.
- Generate YAML instead of writing everything from scratch.
- Use `kubectl explain` for unfamiliar fields.
- Inspect events and logs when something fails.
- Understand the root cause before applying fixes.
- Avoid spending too much time on a single question.

### When troubleshooting

- Check kubelet and the container runtime when nodes are unhealthy.
- Use `crictl` if the Kubernetes API server is unavailable.
- Check API server readiness using `/readyz?verbose`.
- Inspect static Pod manifests when control plane components fail.
- Verify Pod CIDR and CNI configuration for networking problems.
- Check DNS, Service selectors, EndpointSlices, and NetworkPolicies.
- Check Pod requests and limits when scheduling fails or containers restart.
- Inspect node allocated resources when Pods are `Pending`.
- Investigate `OOMKilled` using container termination details and logs.
- Check readiness probes when Pods are running but not ready.

### After making changes

- Verify resource status.
- Check readiness.
- Check logs and events.
- Confirm that the requested behavior actually works.

--
