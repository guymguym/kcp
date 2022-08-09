```bash
export PATH="${PATH}:${PWD}/bin"
export KUBECONFIG=".kcp/admin.kubeconfig"
export KIND_EXPERIMENTAL_PROVIDER=podman
export SYNCER_LOCAL_IMAGE="ko.local/github.com/kcp-dev/kcp/cmd/syncer"

brew install podman docker kind ko jq kubernetes-cli
podman machine init --cpus 6 --memory 6144
podman machine start

go mod vendor
make
ko build --local -t latest --tag-only --preserve-import-paths --platform=linux/arm64 ./cmd/syncer

kcp start
kubectl ws create kcp-storage --enter --ignore-existing
kubectl ws create-context kcp-storage --overwrite

# syncer
kubectl kcp workload sync cluster1 \
  --syncer-image $SYNCER_LOCAL_IMAGE \
  --resources=persistentvolumeclaims,persistentvolumes \
  --output-file syncer-cluster1.yaml
kind create cluster --name cluster1
kind load docker-image $SYNCER_LOCAL_IMAGE --name cluster1
kubectl --context kind-cluster1 apply -f syncer-cluster1.yaml

# apply the bad deployment twice
kubectl --context kcp-storage create ns app1 
kubectl --context kcp-storage -n app1 apply -f app1.yaml
kubectl --context kcp-storage -n app1 apply -f app1.yaml
```


Rebuild kubeconfig contexts:

```bash
kubectl config use-context root
kubectl ws use kcp-storage
kubectl ws create-context kcp-storage --overwrite
kubectl config set-context kcp-storage --namespace app1
kubectl config use-context kcp-storage
kubectl config get-contexts
kind export kubeconfig --name cluster1
```
