# kubecon-cluster

KubeCon Workshop cluster config

## Troubleshooting

To see if your IDE is connected to the cluster run:

```shell
kubectl get pods --all-namespaces
```

### Gitops Hands-On

#### Step 1

```shell
cd /workspace/podinfo && \
echo "#" >> Makefile && \
git add . && git commit -m "my first build" && git push origin master
```

#### Step 2

In the next command `USER` should be of the form e.g. `training-user-2`.

```shell
watch gcloud container images list-tags gcr.io/dx-training/USER-podinfo
```

*e.g.* `watch gcloud container images list-tags gcr.io/dx-training/training-user-2-podinfo`

Check the output of the script for the most recent entry (check time). It
should look like `master-5fd5dde`, where `5fd5dde` is the tag.

#### Step 3

In the IDE edit `podinfo-dep.yaml` inside `workspace/cluster/dev` to be

```yaml
image: gcr.io/dx-training/USER-podinfo:master-TAG
```

Or use this sed command

```bash
cd /workspace/cluster && \
export MASTER_TAG=$(gcloud container images list-tags gcr.io/dx-training/${IDE_USERNAME}-podinfo | grep master |  awk '{print $2}') && \
sed -i.bak "s,quay.io/stefanprodan/podinfo:0.3.0,gcr.io/dx-training/${IDE_USERNAME}-podinfo:${MASTER_TAG},g" ./dev/podinfo-dep.yaml && \
rm ./dev/podinfo-dep.yaml.bak
```

Commit the changes to master

```shell
cd /workspace/cluster && \
git add . && git commit -m "my first deploy" && git push origin master
```



#### Step 4

```shell
kubectl create namespace dev
watch kubectl -n dev get deployments
```

#### Step 5

Note the Cluster-IP.

```shell
export CLUSTER_IP=$(kubectl -n dev get svc podinfo -o jsonpath='{.spec.clusterIP}') && \ 
curl ${CLUSTER_IP}:9898/version
```

#### Step 6

Edit `podinfo-dep.yaml` and add the `annotations:` section:

```yaml
apiVersion: apps/v1beta2
kind: Deployment
metadata:
  annotations:
    flux.weave.works/automated: "true"
    flux.weave.works/tag.podinfod: glob:master-*
  name: podinfo
```

```shell
git add . && git commit -m "automate deploy" && git push origin master
```

#### Step 7

Edit `podinfo/pkg/version/version.go` to be:

```go
var VERSION = "0.3.1"
```

```shell
cd ../../podinfo
git add .  && git commit -m "release v0.3.1 to dev" && git push origin master
git tag 0.3.1
git push origin 0.3.1
```

#### Step 8

```shell
kubectl -n dev get svc
watch curl CLUSTER-IP:9898/version
kubectl create namespace prod
```

#### Step 9

```shell
cd ../cluster && cp -r dev prod
```

In `prod/prodinfo-svc.yaml` change the `namespace` from `dev` to `prod`.

In `prod/prodinfo-dep.yaml` change `namespace` from `dev` to `prod`, also
change the flux filter to:

```yaml
flux.weave.works/tag.podinfod: glob:*.*.*
```

```shell
git pull origin master
git add .  && git commit -m "prod config" && git push origin master
```

#### Step 10

```shell
watch kubectl -n prod get deployments
kubectl -n prod get svc
curl CLUSTER-IP:9898/version
```
