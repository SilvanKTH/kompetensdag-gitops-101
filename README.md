# Introduction to GitOps

This repository contains instructions for setting up FluxCD and templates for implementing a (simplistic) GitOps workflow, allowing for continuous deployment / delivery. The demo application may be simple, but the principles can be applied in any Kubernetes environment.

## What is GitOps?

GitOps is based on the foundation "git as the single source of truth". All code running on a cluster is persisted in git in a declarative way.

## What is a Kustomization?

A Kustomization in the Kubernetes sense (there are also Kustimizations in FluxCD, more on that below) allows for enforcing the DRY principle across clusters. You can organize your repository with so-called overlays and let the Kustomization render the differences.

The idea behind Kustomizations is that you have a so-called base and only need to define what differs between the environments that you wish to control, e.g. dev, staging and production, or blue-green deployments.

## What is FluxCD?

FluxCD is a set of controllers that allow developers to deploy code to Kubernetes the GitOps way. It consists of controllers that serve different purposes:

- Source Controller - used for fetching manifests from a source, e.g. a git repository
- Kustomize Controller - used for compiling the manifests from a source
- Helm Controller - used for deploying helm manifests, similar to Kustomizations

# Live Demo

Local dev only, requires a git repository that is reachable through either ssh or https. This demo uses Docker and minikube.

Follow the installation guides here:

* [Docker](https://docs.docker.com/desktop/)
* [Minikube](https://minikube.sigs.k8s.io/docs/start/)

## Agenda

1.  [Running Gitea](#running-gitea)
2.  [Create Flux Repository](#create-flux-repository-in-gitea)
3.  [Start minikube - Local Kubernetes Cluster](#start-minikube---local-kubernetes-cluster)
4.  [Bootstrap FluxCD](#bootstrap-fluxcd)
5.  [Create Namespaces](#create-namespaces)
6.  [Install Traefik Helm Chart](#install-treafik-helm-chart)
7.  [Kubernetes Kustomizations](#kubernetes-kustomizations)
8.  [Adding External git Repositories to Flux](#adding-external-git-repositories-to-flux)
9.  [Adding Overlays for the Other Environments](#adding-overlays-for-the-other-environments)
10. [Flux Reconciliation](#flux-reconciliation)

## Running Gitea

You can run a local instance of Gitea using docker compose, see also [docker-compose.yaml](/gitea/docker-compose.yaml).

```bash
cd kompetensdag-gitops-101/gitea
docker compose up -d
```

Once this is running, visit http://localhost:3000 and configure the following:

| Type                    | Value                         | Default |
| ------------------------|-------------------------------|---------|
| Database Type           | SQLite3                       | Yes     |
| Path                    | /data/gitea/gitea.db          | Yes     |
| Site Title              | Gitea: Git with a cup of tea  | Yes     |
| Repository Root Path    | /data/git/repositories        | Yes     |
| Server domain           | host.docker.internal          | No      |
| SSH Server Port         | 22                            | Yes     |
| Gitea HTTP Listen Port  | 3000                          | Yes     |
| Gitea Base URL          | http://localhost:3000/        | Yes     |
| Log Path                | /data/gitea/log               | Yes     |

Hit *Install Gitea* and create a user - this user is purely for the local instance. Since we're not using a mailserver the email doesn't matter at this point. You can e.g. use:

* Username: flux-demo
* Email Address: flux-demo@localhost
* Password: <random> # remember your credentials

## Create Flux repository in Gitea

Create a new repository in Gitea using **+** sign in the top right corner called **flux-demo**. Leave everything as default.
Then, create a token that allows read-write access to that repository by browsing to **Settings** >> **Applications** >> **Generate New Token**, give the token a name and select *read-write* for *repository*. Save the token somewhere safe, we will need this token in a bit.

Clone the repository to a any directory outside of this repository:

```bash
cd ../..
git clone http://localhost:3000/flux-demo/flux-demo.git
cd flux-demo
cp ../kompetensdag-gitops-101/flux-template/README.md ./
git add README.md
git commit -m "chore: first commit"
git push # we are not concerned with branch protection in this lab
```

## Start minikube - local Kubernetes cluster

Create a minikube cluster as follows:

```bash
minikube start --driver=docker
```

Once the minikube startup has succeeded, confirm that all services have started:

```bash
kubectl get nodes # should be 'minikube' and status should be 'Ready'
kubectl get pods -A # all pods should be either 'Running' or 'Completed'
```

## Bootstrap FluxCD

First, we need to install the Flux CLI. Do so by by following the [installation guide](https://fluxcd.io/flux/cmd/) for your OS.

```bash
which flux
flux --version # results in flux version 2.8.8 in my case
```

The Flux CLI is used to deploy the Flux controllers to the Kubernetes cluster. We use a token for authentication.

```bash
GITEA_IPADDR=$(ipconfig getifaddr en0) # on Mac, the address on which Kubernetes can reach the git server
GITEA_TOKEN=<your token>

flux bootstrap git \
  --url=http://${GITEA_IPADDR}:3000/flux-demo/flux-demo.git \
  --branch=main \
  --path=clusters/minikube \
  --username=flux-demo \
  --password=${GITEA_TOKEN} \
  --allow-insecure-http
```

<NOTE>: This will create the gotk-components.yaml file in the specified path on the main branch, but gets stuck at generating the source secret. Verify in a different terminal if the controllers have been created. Then, you can press Ctrl+C and continue with the manual bootstrapping process below.

```bash
kubectl get all -n flux-system # should display four deployments
git pull # should retrieve the clusters/minikube/flux-system/gotk-components.yaml file from the upstream flux-demo repoistory
```

Then, create the git repository source for the flux-demo repository via the Flux CLI. This will allow the Flux source controller to regularly update its index.

```bash
flux create source git flux-system \
  --url=http://${GITEA_IPADDR}:3000/flux-demo/flux-demo.git \
  --branch=main \
  --interval=1m \
  --export > clusters/minikube/flux-system/gotk-sync.yaml
```

Additionally append the following Flux kustomization to the source:

```bash
flux create kustomization flux-system \
  --source=flux-system \
  --path="./clusters/minikube" \
  --prune=true \
  --interval=1m \
  --export >> clusters/minikube/flux-system/gotk-sync.yaml
```

This results in the following file that we can dissect as follows:

```yaml
---
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: flux-system
  namespace: flux-system
spec:
  interval: 1m0s # how often the source controller will fetch from the upstream repository
  ref:
    branch: main # from which branch it will fetch
  url: http://<gitea-ip-addr>:3000/flux-demo/flux-demo.git # the upstream repository, given the GITEA_IPADDR
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: flux-system
  namespace: flux-system
spec:
  interval: 1m0s # how often Kustomizations will be reconciled
  path: ./clusters/minikube # the path on which the Kustomizations will be applied
  prune: true # whether resources will be retained or not, e.g. upon accidental deletion
  sourceRef:
    kind: GitRepository # pointer to the repository
    name: flux-system
```

Now you can apply the manifest directly, verify that the sources and kustomizations are in place, and commit to the flux-demo repository:

```bash
kubectl apply -f clusters/minikube/flux-system/gotk-sync.yaml
flux get all # should display gitrepository/flux-system and kustomization/flux-system
git add -A
git commit -m "feat: add flux git repo and kustomization"
git push
```

## Create namespaces

Now let's put FluxCD to the test. Let's create three namespaces declaratively using GitOps. Create the following three directories and copy the manifests from the flux-template directory:

```yaml
mkdir -p clusters/minikube/dev && \
    cp ../kompetensdag-gitops-101/flux-template/demo/dev/DEV.md ./clusters/minikube/dev/DEV.md && \
    cp ../kompetensdag-gitops-101/flux-template/demo/dev/namespace.yaml ./clusters/minikube/dev/namespace.yaml
mkdir -p clusters/minikube/stage && \
    cp ../kompetensdag-gitops-101/flux-template/demo/stage/STAGE.md ./clusters/minikube/stage/STAGE.md && \
    cp ../kompetensdag-gitops-101/flux-template/demo/stage/namespace.yaml ./clusters/minikube/stage/namespace.yaml
mkdir -p clusters/minikube/prod && \
    cp ../kompetensdag-gitops-101/flux-template/demo/prod/PROD.md ./clusters/minikube/prod/PROD.md && \
    cp ../kompetensdag-gitops-101/flux-template/demo/prod/namespace.yaml ./clusters/minikube/prod/namespace.yaml
```

Your flux repo should resemble something like this:

```bash
➜  flux-demo git:(main) ✗ tree
.
├── README.md
└── clusters
    └── minikube
        ├── dev
        │   ├── DEV.md
        │   └── namespace.yaml
        ├── flux-system
        │   ├── gotk-components.yaml
        │   └── gotk-sync.yaml
        ├── prod
        │   ├── PROD.md
        │   └── namespace.yaml
        └── stage
            ├── STAGE.md
            └── namespace.yaml

7 directories, 9 files
```

Commit everything and wait for Flux to create our namespaces.

```bash
git add -A
git commit -m "feat: add dev, stage, and prod Kubernetes namespaces"
git push
```

This should result in the following namespaces:

```bash
➜  flux-demo git:(main) kubectl get ns
NAME              STATUS   AGE
default           Active   11m
dev               Active   9s
flux-system       Active   9m42s
kube-node-lease   Active   11m
kube-public       Active   11m
kube-system       Active   11m
prod              Active   9s
stage             Active   9s
```

These resources are applied directly through the kustomize controller that is fetching the latest revision of the main branch. You can see this in the logs of the Kustomize controller:

```bash
kubectl logs -n flux-system deployments/kustomize-controller
```

## Install Treafik Helm Chart

Flux is capable of installing helm charts using its helm controller. To do this, inspect and copy the following files:

1. We need to have a source that is pointing to an upstream repo.

```yaml
---
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: traefik
  namespace: flux-system
spec:
  url: https://traefik.github.io/charts # upstream repo
  interval: 10m # governs how often a source is fetched
```

2. If the source is a helm chart, implement a helm release pointing to the given source.

```yaml
---
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: traefik
  namespace: kube-system
spec:
  interval: 5m # how often a source is queried
  chart:
    spec:
      chart: traefik # name of the chart
      version: "39.x"   # flexible version constraint
      sourceRef:
        kind: HelmRepository
        name: traefik
        namespace: flux-system
  values: # equivalent to helm values file
    service:
      type: LoadBalancer
```

Copy the files and commit to the Flux repository. This will install the chart.

```bash
mkdir -p clusters/minikube/sources
cp ../kompetensdag-gitops-101/flux-template/demo/sources/traefik.yaml clusters/minikube/sources/traefik.yaml
mkdir -p clusters/minikube/infra
cp ../kompetensdag-gitops-101/flux-template/demo/infra/traefik.yaml clusters/minikube/infra/traefik.yaml
git add -A
git commit -m "feat: install traefik ingress controller"
git push
```

Wait until the sources and helmreleases are deployed:

```bash
flux get sources helm
flux get helmreleases -n kube-system # this is where the chart gets deployed by default
kubectl get pods -n kube-system -l app.kubernetes.io/instance=traefik-kube-system # should be running after a short while
```

## Kubernetes Kustomizations

Let's link an imaginary application repository to our Flux controllers, and use Kubernetes Kustomizations to tailor the application to each environment. For that, create a new repository in Gitea as we did before, and name the repository *app-demo*.

```bash
cd ..
git clone http://localhost:3000/flux-demo/app-demo.git
cd app-demo
cp ../kompetensdag-gitops-101/app-template/README.md ./README.md
git add -A
git commit -m "chore: first commit"
git push
```

Then, let's create the base template for the repository:

```bash
cp -r ../kompetensdag-gitops-101/app-template/base ./base
```

and the Kustomizations:

```bash
cp -r ../kompetensdag-gitops-101/app-template/overlays ./overlays
```

This should look like this:

```bash
➜  app-demo git:(main) ✗ tree
.
├── README.md
├── base
│   ├── deployment.yaml
│   ├── kustomization.yaml
│   └── service.yaml
└── overlays
    ├── dev
    │   ├── index.html
    │   └── kustomization.yaml
    ├── prod
    │   ├── index.html
    │   └── kustomization.yaml
    └── stage
        ├── index.html
        └── kustomization.yaml

6 directories, 10 files
```

To render the different templates to actual manifests that can be consumed by the kube-api server, you can use kubectl.

```bash
kubectl kustomize overlays/dev # will render the template without applying anything
```

If you're happy with the results (it should render), commit the changes made.

```bash
git add -A
git commit -m "feat: add base application and overlays for dev, stage and prod"
git push
```

## Adding External git Repositories to Flux

Now we want to apply the templates using GitOps, that means we need to instruct Flux to

1. Fetch an external repository
2. Add the flux kustomizations

We can do this as follows using the Flux CLI, or by simply committing the corresponding manifests to the flux-demo repoistory.

```bash
cd ../flux-demo
flux create source git app-demo \
  --url=http://${GITEA_IPADDR}:3000/flux-demo/app-demo.git \
  --branch=main \
  --interval=1m \
  --export > clusters/minikube/sources/app-demo.yaml
```

Commit this and wait until the source appears and is marked as *Ready*.

```bash
git add -A
git commit -m "feat: add app-demo git source"
git push
flux get sources git --watch # Ctrl+C to exit once the app-demo source shows up
```

And then add the kustomizations for the dev overlay:

```bash
flux create kustomization app-dev \
  --source=app-demo \
  --path="./overlays/dev" \
  --prune=true \
  --interval=1m \
  --export >> clusters/minikube/dev/app.yaml
```

Commit this and verify that the app is being deployed in all its glory:

```bash
git add -A
git commit -m "feat: deploy app in dev namespace using GitOps"
git push
flux get kustomizations --watch # Ctrl+C to exit once the app-dev source shows up
```

Then, you'll be able to see the Kubernetes resources being applied.

```bash
kubectl get all -n dev # should show you the pod, service and deployment
kubectl describe cm -n dev web-content # should show the dev version
kubectl describe ingressroute.traefik.io -n dev web # should show the web-dev.local route
```

To check whether this works, you can do the following:

1. Open a new terminal an run the following command - you can leave this running for the time being:

```bash
minikube tunnel
```

2. Run the following curl command to see whether the static HTML file is correctly rendered:

```bash
curl -H "Host: web-dev.local" http://127.0.0.1
```

## Adding Overlays for the Other Environments

Now you can add further overlays to also apply the stage and prod deployments. They all have the same common denominator (base) but come with some slight configuration differences.

| Type              | Dev           | Stage           | Prod            |
|-------------------|---------------|-----------------|-----------------|
| namespace         | dev           | stage           | prod            |
| automatic reload  | no            | yes             | yes             |
| image             | nginx:stable  | nginx:stable    | nginx:1.30.2    |
| replicas          | 1             | 1               | 3               |
| ingressroute      | web-dev.local | web-stage.local | web-prod.local  |
| html file         | dev           | stage           | prod            |

Once the base has been established, customizing the overlays is often pretty straightforward, since a lot of the boilerplate code can be reused.

We have already got the source for the demo-app in place, at this stage we only need to add the kustomizations for the overlays that we want to deploy.

```bash
flux create kustomization app-stage \
  --source=app-demo \
  --path="./overlays/stage" \
  --prune=true \
  --interval=1m \
  --export >> clusters/minikube/stage/app.yaml

flux create kustomization app-prod \
  --source=app-demo \
  --path="./overlays/prod" \
  --prune=true \
  --interval=1m \
  --export >> clusters/minikube/prod/app.yaml
```

Commit this and Flux will pick up the kustomizations.

```bash
git add clusters/minikube/stage/app.yaml
git commit -m "feat: deploy app in stage namespace"
git add clusters/minikube/prod/app.yaml
git commit -m "feat: deploy app in prod namespace"
git push
```

This will be picked up by the Flux controller in a short while.

```bash
flux get kustomizations --watch
kubectl get all -n stage
kubectl get all -n prod
```

Given that your minikube tunnel is still running in the background, you can verify that the ingressroutes work correctly:

```bash
curl -H "Host: web-stage.local" http://127.0.0.1
curl -H "Host: web-prod.local" http://127.0.0.1
```

## Flux Reconciliation

Now let's verify how Flux ensures that git remains the single source of truth. Let's edit the ingress route in the dev namespace:

```bash
kubectl patch ingressroutes.traefik.io web -n dev --type='json' \
  -p='[
    {
      "op": "replace",
      "path": "/spec/routes/0/match",
      "value": "Host(`web-lab.local`)"
    }
  ]'
```

Then let's see how long this takes to reconcile:

```bash
start=$(date +%s)
for i in {1..30}; do
    now=$(date +%s)
    elapsed=$((now - start))
    echo "[$elapsed s] Checking..."
    http_status=$(curl -s -m 2 -o /dev/null -w "%{http_code}" \
        -H "Host: web-dev.local" \
        http://127.0.0.1)
    echo "HTTP $http_status"
    if [ "$http_status" -eq 200 ]; then
        echo "Success after ${elapsed}s — exiting loop"
        break
    fi
    sleep 2
done
```

To suspend or resume git repositories, kustomizations, or helmreleases, it is possible to use the Flux CLI to persist changes in a imperative or declarative manner:

```bash
# to persist changes
flux suspend source <git, helm, ...> <name>
flux suspend <kustomization, helmrelease, ...> -n <namespace> <name>

# to resume GitOps
flux resume source <git, helm, ...> <name>
flux resume <kustomization, helmrelease, ...> -n <namespace> <name>
```

# Tearing Down the Lab

To tear down the lab, stop the minikube tunnel, stop (and optionally delete) the cluster, tear down the docker container, and delete the repositories.

```bash
# stop the minikube tunnel with Ctrl+C
minikube stop
minikube delete # optional
docker compose down -v # use the -v flag to delete the volume
cd ..
rm -rf app-demo/
rm -rf flux-demo
```

# Links

- FluxCD: https://fluxcd.io/flux/
- Helm: https://helm.sh/docs
- Traefik: https://doc.traefik.io/traefik/
