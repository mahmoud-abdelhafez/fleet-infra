Table of contents:
- Infrastructure setup
	- Install kind cluster 
	- Install flux cli
	- Kustomize CLI
	- create github repo
	- Flux bootstrap
- Flux Main components walkthrough
	- Source controller
	- Kustimization controller
	- Helm controller
	- Notification controller 
	-   Image automation controller 
- Installing a HelmRelease
	- Add helm repository 
	- Create a HR mainfest 
	- Test locally with kustimize  CLI
- Troubleshooting


Infrastructure setup:
- Install Kind & kind cluster test cluster. https://kind.sigs.k8s.io/docs/user/quick-start
On Linux:
```
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.11.1/kind-linux-amd64
chmod +x ./kind
mv ./kind /some-dir-in-your-PATH/kind
```
On macOS via Homebrew:
```
brew install kind
```
On macOS via Bash:
```
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.11.1/kind-darwin-amd64
chmod +x ./kind
mv ./kind /some-dir-in-your-PATH/kind
```

Install kubectl:
On Linux:
```
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
mv ./kubectl ~/.local/bin/kubectl
kubectl version --client
```

On macOS: 
```
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/darwin/amd64/kubectl"
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl
sudo chown root: /usr/local/bin/kubectl
kubectl version --client
```

Install Flux CLI: 
On Linux:
```
curl -s https://fluxcd.io/install.sh | sudo bash
```
On macOS:
```
brew install fluxcd/tap/flux
OR:
curl -s https://fluxcd.io/install.sh | sudo bash
```

Install KS:  https://kubectl.docs.kubernetes.io/guides/introduction/resources_controllers/
On Linux/macOS
```
curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh"  | bash
```
On macOS:
```
brew install kustomize
```

Creat a KIND  cluster
```
vim ./kind.yaml
# Two node (one workers) cluster config
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker

kind create cluster --config ./kind.yaml
```

Flux bootstrap:
- Create a github repo that flux will reconcile it on your local pick whatever name you like.
- Generate a github personal access token , Will be used for Flux bootstrap.
	- From your profile >  settings  > Developer settings >  personal Access tokens > Generate new token
```
export GITHUB_TOKEN=ghp_YOUR_TOKEN
```
Flux bootstrap
```
 flux bootstrap github  --owner=org  --repository=fleet   --branch=master  --team=flux --path=./clusters/staging
```

Flux Main components walkthrough:
```
kubectl get pods -n flux-system

NAME                                           READY   STATUS    RESTARTS   AGE
helm-controller-b5c45677f-kx9mg                1/1     Running   0          3d8h
image-automation-controller-57f8bf65c7-27tmz   1/1     Running   0          11h
image-reflector-controller-558c56b989-htk56    1/1     Running   0          6d5h
kustomize-controller-7fd4d6b4c8-gfbqs          1/1     Running   0          11h
notification-controller-66545867f4-6sm7w       1/1     Running   0          3d8h
source-controller-cb7694584-sxb9x              1/1     Running   0          2d9h
```
- source-controller  (https://fluxcd.io/docs/components/source/)
- kustomize-controller (https://fluxcd.io/docs/components/kustomize/)
- helm-controller (https://fluxcd.io/docs/components/helm/)
- notifcation-controller 
- image-automation
- image-reflector

Create the kustomizations && Installing a HelmRelease for staging cluster:
```
mkdir clusters/production
mkdir clusters/staging
```
- Add the kustomization for the Infra staging's applications 
```
vim clusters/staging/infrastructure

apiVersion: kustomize.toolkit.fluxcd.io/v1beta1
kind: Kustomization
metadata:
  name: infrastructure
  namespace: flux-system
spec:
  interval: 10m0s
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./infrastructure/staging
  prune: true
  validation: client
```
- Add the sources for the staging cluster
```
apiVersion: kustomize.toolkit.fluxcd.io/v1beta1
kind: Kustomization
metadata:
  name: sources
  namespace: flux-system
spec:
  interval: 10m0s
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./infrastructure/sources/
  prune: true
  validation: client
```
- Create the required directories 
```
mkdir sources
mkdir infrastructure/base
mkdir infrastructure/staging
mkdir infrastructure/production
mkdir infrastructure/sources
```
- Make sure that all  ks layers have been reconciled on the cluster
```
flux get ks
```


Example1: 
Install the metric server:
```
Metrics Server collects resource metrics from Kubelets and exposes them in Kubernetes apiserver through Metrics API for use by Horizontal Pod Autoscaler and Vertical Pod Autoscaler. Metrics API can also be accessed by kubectl top, making it easier to debug autoscaling pipelines.
```
- Add the Metrics server's helm repository, Using flux cli make sure it has reconciled on the cluster 
```
vim infrastructure/sources/banzaicloud.yaml

apiVersion: source.toolkit.fluxcd.io/v1beta1
kind: HelmRepository
metadata:
  name: banzaicloud
  namespace: flux-system
spec:
  interval: 24h
  url: https://kubernetes-charts.banzaicloud.com
```
- Create the base Metrics server's  HR:
```
vim infrastructure/base/metrics-server/metrics-server.yaml
---
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: metrics-server
spec:
  releaseName: metrics-server
  chart:
    spec:
      chart: metrics-server
      version: 0.0.8
      sourceRef:
        kind: HelmRepository
        name: banzaicloud
        namespace: flux-system
  interval: 1m
  values:
    rbac:
      create: true
      pspEnabled: false
    serviceAccount:
      create: true
    apiService:
      create: true
    hostNetwork:
      enabled: false
    image:
      repository: gcr.io/google_containers/metrics-server-amd64
      tag: v0.3.6
      pullPolicy: IfNotPresent
    args:
      - --logtostderr
    replicas: 1
```
- Add the kustomization file:
```
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - metrics-server.yaml
```
- Add the metrics server application on the staging cluster
```
mkdir infrastructure/staging/metrics-server

vim infrastructure/staging/metrics-server/metrics-server.yaml
---
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: metrics-server
spec:
  releaseName: metrics-server
  values:
    replicas: 1

vim infrastructure/staging/metrics-server/kustomization.yaml
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: default
resources:
  - ../../base/metrics-server
patchesStrategicMerge:
  - metrics-server.yaml
```
- Test locally with kustomize  CLI
```
kustomize build . 
```

Task: 
- Install the metrics server on the production cluster, The application should have two replicas instead of the default one replica

Troubleshooting:
- install the PodInfo app on the staging cluster using the following resources:
	- Helm Repo  And HR
```
apiVersion: source.toolkit.fluxcd.io/v1beta1
kind: HelmRepository
metadata:
  name: podinfo
spec:
  interval: 5m
  url: https://stefanprodan.github.io/podinfo
```

```
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: podinfo
  namespace: podinfo
spec:
  releaseName: podinfo
  chart:
    spec:
      versions: 6.1.0
      chart: pod-info
      sourceRef:
        kind: HelmRepository
        name: podinfo
        namespace: flux-system
  interval: 5m
  install:
    remediation:
      retries: 3
  # Default values
  # https://github.com/stefanprodan/podinfo/blob/master/charts/podinfo/values.yaml
  values:
    cache: redis-master.redis:6379
    ingress:
      enabled: true
      annotations:
        kubernetes.io/ingress.class: nginx
```
