## State of the Art of GitOps with Helm

Highly customizable GitOps pipeline built with Helm

APPENDIX: The end of Kustomize vs Helm argument

## What is Helm

- Helm is a package manager for K8s
- "Package" is called "Chart" in Helm

> ![What is Helm](https://helm.sh/src/img/chart-illustration.png)
> https://helm.sh/

## What is GitOps

WHAT: Auditability(Every change is git-trackable) / Security(less priv in CI)
HOW: Pull desired state / Sync K8s resources Git → Cluster

> ![What is GitOps?](https://images.contentstack.io/v3/assets/blt300387d93dabf50e/blt15812c9fe056ba3b/5ce4448f32fd88a3767ee9a3/download)
> https://www.weave.works/technologies/gitops/

### State-of-the-Art GitOps Solutions

1. Weaveworks [Flux](https://github.com/weaveworks/flux) Operator
2. [Argo CD](https://github.com/argoproj/argo-cd)
3. MISSING PIECE

#### Flux

- `flux` fetches git commits and reconsile K8s resources.
- `helm-operator` reconciles `HelmRelease` resources to reconsile K8s resources.

> ![Flux Deployment Pipeline](https://github.com/weaveworks/flux/raw/master/site/images/deployment-pipeline.png)
> https://github.com/weaveworks/flux

#### Argo CD

- Fetches git commits to reconcile K8s resources
- Plug-in any K8s manifests builder (kustomize, helm, ksonnet, kubecfg, etc.)

> ![Argo CD Architecture](https://argoproj.github.io/argo-cd/assets/argocd_architecture.png)
> https://argoproj.github.io/argo-cd/#architecture

PLUG: Wanna declaratively manage Argo CD projects? Use [the community Helm chart](https://github.com/chatwork/charts/tree/master/argoproj-crd)

## Known Problems

1. Limited Customizability
2. High Number of Total Moving-Parts

### Limited Customizability

`flux` and `argocd` has limited extension points.

ex) `argocd` supports custom "hooks".

- But the deployment is hard-coded to `kubectl apply`.
- How to `helm install` it?

### High Nummber of Total Moving-Parts

Worth maintaining both CI and CD systems separately? Your team size?

**CI (lint, diff, test on PUSH)**:
Travis, CircleCI, Concourse, Jenkins, Argo CI, ...

**CD (deploy/sync/reconcile on PULL)**:
**Flux**, **Argo CD**, Spinnaker, ...

## Goal: Filling the MISSING PIECE

- Flux
- Argo CD
- MISSING PIECE = **Single** customizable system that handles both **CI** and **CD**

## Example: The Single Tool

- `brigade` is K8s scripting system
- Basically CI system whose pipelines are written in JavaScript(Turing-Complete!)

![Brigade](https://docs.brigade.sh/img/design-02.png)

https://github.com/brigadecore/brigade

## Example Solution

- [brigade](https://github.com/brigadecore/brigade)(an open, event-driven K8s scripting platform) as an universal workflow engine that runs both CI and GitOps/CD pipelines

- [helmfile](https://github.com/roboll/helmfile) to declaratively manage all the apps on K8s. **Use whatever you like tho.**

## Recipe

1. Write the desired state of your cluster
2. Edit & develop locally
3. Push Git commit
4. CI: Test on PR
5. CD: Apply on commit to `master`

## Install

### Prereqs.

1. Create the demo repository:

   ```console
   ORG=mumoshu # Replace with your org name!
   REPO=$ORG/demo-$(</dev/urandom dd ibs=1 obs=1 count=9 | od -tx1 -An | tr -d ' ')

   hub create $REPO
   ```

2. Create a GitHub App for Brigade following [this guide](https://github.com/brigadecore/brigade-github-app/blob/c04ea3fa28f2e0a3a64d74131bfef1fe7698355a/README.md#1-create-a-github-app)
   If you have created one before, discover it in https://github.com/settings/apps
  - Navigate to `GitHub App > Private keys > Private key`, generate a private key by clicking `Generate a private key` button, download the private key and set its full path to `BRIGADE_GITHUB_APP_KEY`
  - Navigate to `GitHub App > About`, take a note of `App ID`, set it to `BRIGADE_GITHUB_APP_ID`
  - Navigate to `GitHub App > Webhook secret (optional)`. Generate a random password and put it there, also set it to `BRIGADE_PROJECT_SECRET`
4. Set envvars:

   ```
   export NGROK_TOKEN=<Your ngrok token shown in https://dashboard.ngrok.com/get-started>
   export BRIGADE_GITHUB_APP_KEY=<Local path to the private key file for TLS client auth from the step 2>
   export BRIGADE_GITHUB_APP_ID=<App ID from the step 2>
   export BRIGADE_PROJECT_SECRET=<Webhook (shared) secret from the step 2>
   export SSH_KEY=$HOME/.ssh/id_rsa (Or whatever ssh private key you want Brigade to use while git-cloning private Git repos)
   export GITHUB_TOKEN=<Used for updating commit/pull request statuses>
   ```

### The Desired State

SPOILER: We use this locally and remotely:

`helmfile.yaml`:

```
releases:
- name: frontend
  chart: flagger/podinfo
  namespace: test
  values:
  - nameOverride: frontend
```

See [helmfile.yaml](helmfile.yaml) for the full example.

a.k.a

```
helm upgrade --install \
  frontend flagger/podinfo \
  --set nameOverride=frontend`
```

### Bootstrap

Run:

```
# See what will be installed

$ helmfile diff

# Actually install all the things

$ helmfile apply
```

#### `helmfile apply` will install...

- Brigade Server
- Brigade GitHub App
- Brigade Project
- In-Cluster Ngrok Tunnel (GitHub Webhooks to Brigade GitHub App
- Other apps for demonstration purpose

### Local Development

Run:

```
$ git checkout -b change-blah
$ $EDITOR helmfile.yaml
```

(Again,) Run:

```
$ helmfile diff
$ helmfile apply
```

### Remote Deployment

Run:

```
$ git add helmfile.yaml && \
  git commit -m 'Change blah' && \
  git push origin master
$ hub pull-request
```

So that `Brigade` (Again) Runs:

```
$ helmfile diff
```

### Review

Review the PR on GitHub.

### Merge

- Merge the pull request into `master` so that
- `brigade` pulls the commit and applies it by (AGAIN!) running:
  ```
  $ helmfile apply
  ```

Voila! You've implemented GitOps.

### Implementation

The `brigade` script looks like:

```javascript
const { events, Job , Group} = require("brigadier")
const dest = "/workspace"
const image = "mumoshu/helmfile-gitops:dev"

events.on("push", (e, p) => {
  console.log(e.payload)
  var gh = JSON.parse(e.payload)
  if (e.type == "pull_request") {
    // Run "helmfile diff" for PRs
    run("diff")
  } else {
    // Run "helmfile apply" for commits to master
    run("apply")
  }
});
```

See [brigade.js](brigade.js) for full example.

### The utility function

```
function run(cmd) {
    var job = new Job(cmd, image)
    job.tasks = [
        "mkdir -p " + dest,
        "cp -a /src/* " + dest,
        "cd " + dest,
        `helmfile ${cmd}`,
    ]
    job.run()
}
```

## Great!

Another GitOps solution that helps!

- Flux
- Argo CD
- Brigade (PROPOSED!)

## Fin.

Try youself!

https://gitpitch.com/mumoshu/helmfile-gitops

## One More Thing

"The end of the "Kustomize vs Helm argument"

### Everyone Does This

- `helm template mychart | kubectl apply -f`
- `helm template mychart --outputs-dir manifests/ && (kustomize build | kubectl apply -f -)`

### Don't use `kubectl apply -f`

When you want:

- `helm diff`: Preview changes before apply
- `helm test`: Run tests included in the chart
- `helm stauts`: List helm-managed resources and the installation note
- `helm get values`: Which settings I used when installing this?

### We Can Do Better

- (Optionally) Generate K8s manifests from Helm chart
- Patch K8s manifests with Kustomize (JSON Patch and Strategic-Merge Patch available)
- Install the patched manifests with Helm

### Example: helm-x

Shameless Plug: https://github.com/mumoshu/helm-x

`helm`:

```
$ helm install myapp mychart
```

`helm-x`:

```
$ helm x install myapp mychart
```

### Usage

Patch and diff/install/up whatever "as Helm chart":

```
$ helm x [diff|install|upgrade] --install myapp WHAT --version 1.2.4 \
  -f values.yaml \
  --strategic-merge-patch path/to/strategicmerge.patch.yaml \
  --jsonpatch path/to/json.patch.yaml
```

WAHT can be:

- Helm chart
- Kustomization
- Directory containing K8s maniefsts

## Great!

You've got everything!

1. GitOps operator (Flux or Argo CD) **OR** Universal system for running CI and CD pipelines (brigade)
2. Universal tool for deploying whatever (helm, kustomize, k8s manifests, helm-x, helmfile)

## Who the author was?

@mumoshu

AWS Container Hero

OSS enthusiast maintaining 10+ K8s-related OSS:

- [kubernetes-incubator/kube-aws](https://github.com/kubernetes-incubator/kube-aws),
- [weaveworks/eksctl](https://github.com/weaveworks/eksctl)
- [roboll/helmfile](https://github.com/roboll/helmfile)
- [brigadecore/brigade](https://github.com/brigadecore/brigade)
