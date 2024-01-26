# retool-local-helm
A template to help you Retool on your local machine via Helm and Minikube

## Prereqs

💻 A Text Editor/IDE set up, and some comfort using a Terminal for CLI commands

**Install:**

🐙 [Git](https://git-scm.com/download/mac), along with [a SSH key configured](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent) for your GitHub account in the [tryretool](https://github.com/tryretool) org.

☸️ [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl-macos/): to run commands against k8s clusters

🐳 [Docker Desktop](https://docs.docker.com/desktop/install/mac-install/) (version 4.25+): our chosen container/vm manager

  In general settings:

 -  choose VirtioFS so that virtualization framework is enabled
  Enable Use Rosetta for x86/amd64 emulation on Apple Silicon
  In resources:

 - Allocate at least 4 CPU and 8GB Memory

 - Docker Desktop provides an option to start a single-node cluster automatically. Leave this disabled.

🛞 [minikube](https://minikube.sigs.k8s.io/docs/start/): our chosen tool for running a local k8s cluster

- Set the default driver to [docker](https://minikube.sigs.k8s.io/docs/drivers/docker/):

  ```minikube config set driver docker```

- Bump up the default CPU and Memory allocation:

  ```
  minikube config set cpus 4
  minikube config set memory 8192
  ```

  These values cannot surpass the resources you set for Docker Desktop

**Concepts**:

Read the first 27 pages of [Kubernetes for Application Developers](https://slack-files.com/T03Q236LG-F05NZRE7KML-0e8046321e)

- Skip installing VirtualBox, we'll be using Docker as our vm driver

Watch this [video intro to Helm](https://www.youtube.com/watch?v=9cwjtN3gkD4)

- It's based on Helm 2 and so [is a bit outdated](https://helm.sh/docs/faq/changes_since_helm2/), but nonetheless is a great concise summary to many Helm concepts

Go through the [hello-minikube](https://kubernetes.io/docs/tutorials/hello-minikube/) tutorial

---


### Intro

Our [Helm Chart](https://github.com/tryretool/retool-helm) is, at time of writing, _de facto_ owned by the Infra team.

- See [Releases](https://github.com/tryretool/retool-helm/releases): this is the closest we have to a changelog

### Getting Started

0) Use this template to create a new repo owned your github user, e.g local-helm

1) Clone that repo to your machine. If you don't already have one, create a folder in your home directory to help keep organized, e.g ~/local-retool

2) Open up that repo in our text editor of choice

3) Open up a terminal and start Minikube (`minikube start`)

4) Start the Minikube Dashboard: `minikube dashboard`. Open the dashboard in a browser.

5) Look at the `Nodes` section to verify your resource allocation.

    Check `persistent volume claims` (if you are using the postgres subchart) and `secrets` if there are any old objects left-over, and delete them if you want to start fresh.

6) Peruse the `values.yaml` file, [the standard way](https://helm.sh/docs/chart_template_guide/values_files/) to inject configuration into templates.

### Choosing a Chart Version

First, add the Retool Helm Chart Repository. Open a new terminal tab and run

```
helm repo add retool https://charts.retool.com
```

Then, run

```
helm search repo retool/retool
```

to confirm it's added.

To "deploy" Retool to Minikube, we install a chart via `helm install`. [Helm's docs](https://helm.sh/docs/helm/helm_install/#helm) note there are 6 ways to express the chart we want to use, but here we only care about 2 of them:

#### By Chart Reference:

The reference here is to the same repository we just added in the previous step.

Helm docs continued:

> When you use a chart reference with a repo prefix ('example/mariadb'), Helm will look in the local configuration for a chart repository named 'example', and will then look for a chart in that repository whose name is 'mariadb'. It will install the latest stable version of that chart...or supply a version number with the '--version' flag.

So running `helm install my-app retool/retool` would pull the latest stable chart version from https://charts.retool.com.

With a specific version:

```
helm install my-app retool/retool --version 5.0.10 -f values.yaml
```


#### Unpacking a chart to your local directory

[`helm pull`](https://helm.sh/docs/helm/helm_pull/#helm) can download a chart and unpack it into a directory on your machine. Use the `--version` flag to download a specific version, and since charts are packaged as gzipped tar archives, you need to pass the `--untar` flag to unpack it:

```
helm pull retool/retool --version 6.0.13 --untar
```

Once that's added, change into the chart directoy and install dependencies:

```
cd retool
helm dep build
```

Downloading the chart locally allows us to easily reference the template files (in `./retool/templates`) and make modifications to them as part of testing. Once the chart is unpacked, install with:

```
helm install my-app ./retool -f values.yaml
```

#### Debugging Templates

The helm docs have a [nice primer](https://helm.sh/docs/chart_template_guide/debugging/#helm) on debugging chart installs. The two most useful commands are:

**helm template --debug**

> `helm template --debug` will test rendering chart templates locally.

You can redirect that output into a file for easy inspection:

```
helm template --debug ./retool -f values.yaml > debug.yaml
```

**helm install --dry-run --debug**

> `helm install --dry-run --debug:` ...have the server render your templates, then return the resulting manifest file

Similarly, output to a file:

```
helm install my-app ./retool -f values.yaml --dry-run --debug > dryrun.yaml
```

### Installing the chart

This workshop comes with a `values.yaml` file based on the `6.0.13` release, with a few modifications:

- `insecureCookies` is set to `true`

- Workflows is disabled

- `livenessProbe`, `readinessProbe`, and `startupProbe` are all disabled

- `ingress` is disabled

- `replicaCount` is set to `1` (all service types run on the same pod)

- The `Persistent Volume Claim` of the postgres subchart is set to `8Gi` (I can't explain why, but this is necessary for the postgres pod to work)

The chart also has 3 basic [service types](https://kubernetes.io/docs/concepts/services-networking/service/#publishing-services-service-types), `ClusterIP`, `NodePort`, and `LoadBalancer`, with `LoadBalancer` uncommented by default to expose the Minikube cluster to traffic.

To get Retool running:
1) Create a release:

```
helm install my-app ./retool -f values.yaml
```

Go to the Minikube dashboard to inspect the deployment, pods, etc.

2) Use the Minikube [`service` command](https://minikube.sigs.k8s.io/docs/handbook/accessing/) if you're using a `NodePort` or the `minikube tunnel` command for a `LoadBalancer` to expose your cluster to your host machine

```
minikube service my-app --url

// e.g http://127.0.0.1:65651
```

or

```
minikube tunnel

// e.g http://localhost:3000
```
