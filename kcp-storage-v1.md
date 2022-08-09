# KCP Storage

This is a notebook for working on KCP Storage.

---

## First Time Setup

### Setup Python

WARNING - Run this script outside the notebook (copy paste to your terminal).

This script sets up the notebook virtualenv and the `bash_kernel` to be able to execute bash cells.


```bash
python -m venv .venv
. .venv/bin/activate
pip install notebook
pip install bash_kernel
python -m bash_kernel.install
deactivate
```

### Setup for Mac - Podman Machine, Kind, etc.

WARNING - Run this script outside the notebook (copy paste to your terminal).

Refer to [kind rootless docs](https://kind.sigs.k8s.io/docs/user/rootless).


```bash
brew install podman docker kind ko jq
sudo podman-mac-helper install

podman machine init --cpus 6 --memory 6144
podman machine start
podman machine ssh
# next commands run inside the machine shell
sudo bash -c 'cat << EOF > /etc/systemd/system/user@.service.d/delegate.conf
[Service]
Delegate=yes
EOF'
sudo bash -c 'cat << EOF > /etc/modules-load.d/iptables.conf
ip6_tables
ip6table_nat
ip_tables
iptable_nat
EOF'
sudo systemctl daemon-reload
sudo modprobe -v ip6_tables
sudo lsmod | grep ip6
exit
```

## Env

DON'T FORGET to run this step in every new notebook/shell or after restarting the notebook kernel.


```bash
export PATH="${PATH}:${PWD}/bin"
export KIND_EXPERIMENTAL_PROVIDER=podman
export SYNCER_LOCAL_IMAGE="ko.local/github.com/kcp-dev/kcp/cmd/syncer"
export KUBECONFIG=".kcp/admin.kubeconfig"
```

## Build

Run these after updating the code or rebasing from upstream.

### Build KCP Binaries

Builds the project binaries into `bin/` dir.


```bash
make
```

### Build Syncer Image

This script works for Mac M1 (arm64) with podman machine (linux vm).
TODO - Need to adjust it for linux hosts.


```bash
podman machine start
ko build --local -t latest --tag-only --preserve-import-paths --platform=linux/arm64 ./cmd/syncer
podman images
```

## KCP

### Run KCP

WARNING - Run this script outside the notebook (copy paste to your terminal).

Running kcp in the foregeround will block the notebook, so let it run in the background.

If you need to reset kcp state, just delete the entire `.kcp` dir.

The binary itself is in the `bin/` dir and should be in PATH after the Env step.

```bash
kcp start
```

### Workspace

Set up the workspace, which is a virtual cluster (or tenant).

```bash
kubectl config use-context default
kubectl ws create kcp-storage --enter --ignore-existing
kubectl ws create-context kcp-storage --overwrite
kubectl config get-contexts
```

## Workload Clusters

Start kind clusters on podman machine.

### Create 2 fresh clusters

WARNING - this step is destructive and will delete the existing clusters first.

```bash
kind delete cluster --name cluster1
kind delete cluster --name cluster2
kind create cluster --name cluster1
kind create cluster --name cluster2
kubectl config use-context kcp-storage # back to kcp
kubectl config get-contexts
```

### Configure contexts

This step makes sure that the kubeconfig file of kcp also has the contexts for the workload clusters.

This is needed after every kcp restart because it resets the kubeconfig file.


```bash
kind export kubeconfig --name cluster1
kind export kubeconfig --name cluster2
kubectl config use-context kcp-storage # back to kcp
kubectl config get-contexts
```

### Stop and start clusters

This step will restart the workload clusters using podman/kind.

You can take just one of those to stop a single cluster.


```bash
podman stop cluster1-control-plane
podman stop cluster2-control-plane
podman start cluster1-control-plane
podman start cluster2-control-plane
```

## Syncer

Run syncer per cluster to connect it to kcp.

### Syncer deploy

Load syncer image into the clusters, prepare the syncer yaml, and re-apply it to the clusters.


```bash
#SYNCER_RESOURCES=persistentvolumeclaims,storageclasses.storage.k8s.io,statefulsets.apps,services
SYNCER_RESOURCES=persistentvolumeclaims,persistentvolumes

kind load docker-image $SYNCER_LOCAL_IMAGE --name cluster1
kind load docker-image $SYNCER_LOCAL_IMAGE --name cluster2

kubectl kcp workload sync cluster1 --syncer-image $SYNCER_LOCAL_IMAGE --resources=$SYNCER_RESOURCES --output-file syncer-cluster1.yaml
kubectl kcp workload sync cluster2 --syncer-image $SYNCER_LOCAL_IMAGE --resources=$SYNCER_RESOURCES --output-file syncer-cluster2.yaml

kubectl delete --ignore-not-found -f syncer-cluster1.yaml --context kind-cluster1
kubectl delete --ignore-not-found -f syncer-cluster2.yaml --context kind-cluster2

kubectl apply -f syncer-cluster1.yaml --context kind-cluster1
kubectl apply -f syncer-cluster2.yaml --context kind-cluster2
```

### Syncer current namespace

```bash
kubectl config set-context kind-cluster1 --namespace $(kubectl get ns -o name --context kind-cluster1 | grep --color=none kcp-syncer | cut -d/ -f2)
kubectl config set-context kind-cluster2 --namespace $(kubectl get ns -o name --context kind-cluster2 | grep --color=none kcp-syncer | cut -d/ -f2)
kubectl config get-contexts
```

### Syncer status

```bash
kubectl get -f syncer-cluster1.yaml --context kind-cluster1
echo; echo ====================; echo ====================; echo; 
kubectl get -f syncer-cluster2.yaml --context kind-cluster2
```

### Syncer logs

```bash
kubectl logs --tail=10 deploy/kcp-syncer --context kind-cluster1
echo; echo ====================; echo ====================; echo; 
kubectl logs --tail=10 deploy/kcp-syncer --context kind-cluster2
```

### Check ready


```bash
kubectl get workloadclusters -o wide
```

## Application

A simple app with a statefulset and a PVC template to be provisioned per sts instance.

### Deploy app to KCP


```bash
kubectl config set-context kcp-storage --namespace=app1
kubectl create ns app1 --context kcp-storage
kubectl delete -f app1.yaml --context kcp-storage
kubectl apply -f app1.yaml --context kcp-storage
```

### Check app status


```bash
kubectl get ns --context kcp-storage
kubectl get -f app1.yaml --context kcp-storage
kubectl get all -A -l app=registry --context kind-cluster1
kubectl get all -A -l app=registry --context kind-cluster2
```

## Cleanup


```bash
podman delete cluster1-control-plane
podman delete cluster2-control-plane
rm -rf .kcp
```

# Hacks

## After KCP restart

Reset kubeconfig contexts:

```bash
kind export kubeconfig --name cluster1
kubectl config use-context root
kubectl ws use kcp-storage
kubectl ws create-context kcp-storage --overwrite
kubectl config set-context kcp-storage --namespace app1
kubectl config use-context kcp-storage
kubectl config get-contexts
```
