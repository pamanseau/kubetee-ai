# KubeTEE AI Multi Clusters


## Host Cluster & Rancher UI Installation

> The host cluster is the main cluster that will manage all the child clusters
> Rancher is the main security and management for all the clusters and provide the access to project and namespaces
>

### K3S Cluster for simplicity and functionalities over distant data centers

### Rancher UI Installation

```sh
helm repo add rancher-latest https://releases.rancher.com/server-charts/latest

helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
```

```sh
curl -o isrg-root-x1.pem https://letsencrypt.org/certs/isrgrootx1.pem
kubectl create secret generic tls-ca-additional --from-file=cacerts.pem=isrg-root-x1.pem -n cattle-system
```

```sh
helm upgrade --install rancher rancher-latest/rancher \
  --namespace cattle-system \
  --create-namespace \
  --version 2.11.3 \
  -f values-rancher-kubecon.yaml
```

## K3S Rancher Installation

> Child Clusters with K3S will not be FIFS certified

## RK2 Rancher Installation

> Child Clusters should have RKE2 for FIFS certification. Control-plane and workers should be located in the same REGION with differnet Zones for redundancy
>
> - USA East Virginia EAST-1
>   - Zone A, B, C, D, E, F
>
> - USA East Virginia EAST-2
>   - Zone A, B, C
> 
> - Amsterdam [AMS]
>   - AMS1
>   - AMS2

*TODO*:
  - [] Establish the Continent, Regions, Zones standard and labeling
  - [] Deploy a cluster discovery pod that will label the cluster for Karmada


## LONGHORN Install from Rancher

> [!NOTE] Install Network Operator with Milvus and Whereabouts for using private LAN for efficiency

```sh
helm upgrade --install longhorn longhorn/longhorn -n longhorn --create-namespace --version 1.9.1 -f values-longhorn-1.9.1.yaml
```

### Rancher Installallation Longhorn

- [] TODO: Automate adding /data additional drives to nodes

- Default
  - Storage Class
    - locality: best-effort
    - delete snapshot when trim: enable
- Add nvme drives and unschedule OS
- General Settings
  - Default Data Locality: best-effort
  - Snapshot Maximum Count: 8
  - Replica Auto Balance: best-effort
  - Remove Snapshots During Filesystem Trim: enable
  - Guaranteed Instance Manager CPU for V1 Data Engine: 5
- Backup
  - Backup Target: s3://longhorn@eu-fra-01/
  - Backup Target Credential Secret: s3-minio

- Create `retain` class
  - dataLocality: best-effort
  - fsType: ext4
  - unmapMarkSnapChainRemoved: enabled

- Create `retain-single` class
  - replicas: 1

## Docker Registery Secrets

regcred


## GPU NVIDIA

Install the NVIDIA Operator [Documentation](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/getting-started.html#operator-install-guide)

Configure [Multi-Instances](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/gpu-operator-mig.html) and [Time-Slicing](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/gpu-sharing.html)


### Confidential Container Operator

Create Namespace
```sh
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: confidential-containers-system
EOF

kubectl patch namespace confidential-containers-system --type=merge -p \
  '{"metadata":{"labels":{"pod-security.kubernetes.io/enforce":"privileged","pod-security.kubernetes.io/audit":"privileged","pod-security.kubernetes.io/warn":"privileged"}}}'

kubectl apply -f manifests/rancher-cc-permissions.yaml
```

```sh
export RELEASE_VERSION=v0.15.0          
kubectl apply -k "github.com/confidential-containers/operator/config/release?ref=${RELEASE_VERSION}"
```

### Kata Container Operator Helm deployment

> [!WARNING]
> Branch used to stage new development

```sh
helm upgrade --install kata-deploy \
  --namespace kube-system \
  kata-containers/tools/packaging/kata-deploy/helm-chart/kata-deploy \
  -f kata-containers/tools/packaging/kata-deploy/helm-chart/kata-deploy/values-k3s.yaml
```

### Helm Network Operator

```sh
kubectl create ns nvidia-network-operator
kubectl label --overwrite ns nvidia-network-operator pod-security.kubernetes.io/enforce=privileged
```

```sh
helm upgrade --install network-operator nvidia/network-operator --version 25.7.0 -n nvidia-network-operator -f values-network-operator-staging.yaml
```

#### NicClusterPolicy

- [ ] Todo: Different for each Provider
  - [ ] E2E Config

Apply Nic Cluster Policy for IB
```sh
kubectl apply -f manifests/nic-cluster-policy.yaml
kubectl get nicclusterpolicy
kubectl describe nicclusterpolicy nic-cluster-policy
```

### Helm GPU-Operator

```sh
kubectl create ns gpu-operator
kubectl label --overwrite ns gpu-operator pod-security.kubernetes.io/enforce=privileged
```

Check if Node Feature Discovery is running

```sh
kubectl get nodes -o json | jq '.items[].metadata.labels | keys | any(startswith("feature.node.kubernetes.io"))'
```

- [] TODO: Version 25.3.3 ENV Variable [Issue #1694](https://github.com/NVIDIA/gpu-operator/issues/1694)

```sh
helm upgrade --install gpu-operator nvidia/gpu-operator --version 25.3.2 -n gpu-operator --create-namespace  -f values-gpu-operator-staging.yaml
```

```sh
kubectl apply -f nvidia-test.yaml
```

## Cert-manager & External DNS

Install CRDs

```sh
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.18.2/cert-manager.crds.yaml
```

```sh
helm upgrade --install cert-manager jetstack/cert-manager --version v1.18.2  --namespace cert-manager --create-namespace -f values-cert-manager.yaml
```

### Install External DNS with Cloudflare

> [!IMPORTANT]
> Change `cloudflare.apiToken` for CloudFlare and `txtOwnerId` with cluster name
>
> When using API Token authentication, the token should be granted **Zone Read, DNS Edit privileges, and access to All zones**.
>
> If you would like to further restrict the API permissions to a specific zone (or zones), you also need to use the `zoneIdFilters` so that the underlying API requests only access the zones that you explicitly specify, as opposed to accessing all zones.
>
> Change the `txtOwnerId: "kubetee"` in the values-external-dns.yaml

```sh
helm upgrade --install external-dns bitnami/external-dns --version 9.0.2 --namespace kube-system -f values-external-dns-[cluster].yaml
```

### Cert-Manager LetsEncrypt Cluster Issuer

```sh
kubectl apply -f manifests/secret-cloudflare.yaml
kubectl apply -f manifests/cert-manager-ClusterIssuer.yaml
```

## Kube Prometheus Stack with Grakafa

```sh
helm upgrade --install kube-prometheus-stack prometheus-community/kube-prometheus-stack --version 75.13.0 --namespace monitoring --create-namespace -f values-kube-prometheus-stack.yaml
```

### Calico Prometheus Monitoring

Installation [documentations](https://sbulav.github.io/kubernetes/monitoring-calico-kube-stack-prometheus/)

[Documentation Monitoring Calico](https://docs.tigera.io/calico/latest/operations/monitor/monitor-component-metrics)


### Loki

SingleBinary installed until required more servers

```sh
helm upgrade --install loki grafana/loki --version 6.39.0 -n loki --create-namespace -f values-loki-staging.yaml
```

### Promtail

```sh
helm upgrade --install promtail grafana/promtail --version 6.17.0 -n loki -f values-promtail.yaml
```

